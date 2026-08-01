# LLM 自主维护姿势图库 — 实现计划

> **Goal:** 让 LLM 通过新增工具自行搜索、下载、质检、维护姿势参考图库，供 ControlNet 画图使用。

**Architecture:** 新增 `core/pose_library.py` 的 `PoseLibrary` 类（图源搜索/下载/质检/入库/查询），`main.py` 注册两个 LLM 工具（`search_pose_image`、`query_pose_library`），`models.py` 增加 `pose_library_config` 配置节（图源切换开关 + 质检开关），`_conf_schema.json` 暴露配置页。

---

### Task 1: `core/pose_library.py` — PoseLibrary 类

**Files:**
- Create: `core/pose_library.py`

**Interfaces:**
- Produces: `PoseLibrary(config, data_dir)` — `async query(keyword) -> list[dict]`、`async search_and_download(description, count=5, describe_cb=None) -> list[dict]`
- Consumes: `PluginConfig.pose_library`（Task 2 定义）、`save_image_bytes`/`guess_image_content_type`（utils.py，已存在）

- [ ] **Step 1: 写搜索+下载+质检+入库+查询完整实现**

```python
"""LLM 自主维护的姿势参考图库。"""

import asyncio
import json
import os
import re
import time
import uuid

import aiohttp
from PIL import Image
from astrbot.api import logger

from ..utils import guess_image_content_type, save_image_bytes

GELBOORU_API = "https://gelbooru.com/index.php"
RULE34_API = "https://api.rule34.xxx/index.php"
LIBRARY_DIR_NAME = "pose_library"
INDEX_NAME = "index.json"
IMAGES_DIR_NAME = "images"
MIN_IMAGE_SIZE = 500        # 过滤小图
SCORE_FLOOR = 25            # 过滤低分图
MAX_DOWNLOAD_RETRY = 2


class PoseLibrary:
    """LLM 自主维护的姿势参考图库。"""

    def __init__(self, config, data_dir: str):
        self._config = config
        self._root = os.path.join(data_dir, LIBRARY_DIR_NAME)
        self._images_dir = os.path.join(self._root, IMAGES_DIR_NAME)
        self._index_path = os.path.join(self._root, INDEX_NAME)
        os.makedirs(self._images_dir, exist_ok=True)

    # ---------- 查询 ----------

    async def query(self, keyword: str) -> list[dict]:
        """按关键词模糊匹配本地索引（tags/description）。"""
        keyword = str(keyword or "").strip().lower()
        if not keyword:
            return []
        index = self._load_index()
        results = []
        for entry in index:
            haystack = f"{entry.get('tags', '')} {entry.get('description', '')}".lower()
            if keyword in haystack:
                results.append(entry)
        return results

    # ---------- 搜索 + 下载 + 质检 + 入库 ----------

    async def search_and_download(self, description: str, count: int = 5,
                                  describe_cb=None) -> list[dict]:
        """描述 → tag → 图源搜索 → 下载 → 质检 → 入库。返回新入库条目。"""
        if not self._config.enable:
            return []

        count = max(1, min(int(count), self._config.max_download_per_search))
        tags = await self._translate_tags(description, describe_cb)

        posts = await self._search_api(tags, count)
        if not posts:
            logger.warning(f"[OmniDraw] 姿势图库搜索无结果: {tags}")
            return []

        existing_urls = {e.get("source_url") for e in self._load_index()}
        entries = []
        async with aiohttp.ClientSession() as session:
            for post in posts[:count]:
                url = str(post.get("file_url") or "").strip()
                if not url or url in existing_urls:
                    continue
                try:
                    image_bytes = await self._download(session, url)
                    if not image_bytes:
                        continue
                    # 尺寸过滤
                    try:
                        img = Image.open(io_BytesIO(image_bytes))
                        w, h = img.size
                        if max(w, h) < MIN_IMAGE_SIZE:
                            logger.info(f"[OmniDraw] 姿势图过小跳过: {w}x{h}")
                            continue
                    except Exception:
                        continue
                    # 质检（可选）
                    if self._config.enable_quality_check and describe_cb:
                        ok = await self._quality_check(describe_cb, url)
                        if not ok:
                            logger.info(f"[OmniDraw] 姿势图质检未通过，跳过: {url}")
                            continue
                    entry = self._store(image_bytes, url, post)
                    entries.append(entry)
                    existing_urls.add(url)
                except Exception as exc:
                    logger.warning(f"[OmniDraw] 姿势图入库失败 {url}: {exc}")
                    continue

        logger.info(f"[OmniDraw] 姿势图库新增 {len(entries)} 张")
        return entries

    # ---------- 内部 ----------

    def _load_index(self) -> list[dict]:
        if not os.path.exists(self._index_path):
            return []
        try:
            with open(self._index_path, "r", encoding="utf-8") as f:
                data = json.load(f)
            return data if isinstance(data, list) else []
        except Exception:
            return []

    def _save_index(self, index: list[dict]) -> None:
        tmp_path = f"{self._index_path}.{uuid.uuid4().hex}.tmp"
        with open(tmp_path, "w", encoding="utf-8") as f:
            json.dump(index, f, ensure_ascii=False, indent=2)
        os.replace(tmp_path, self._index_path)

    def _store(self, image_bytes: bytes, source_url: str, post: dict) -> dict:
        """图片落盘 + 索引追加（原子写）。"""
        ext = guess_image_content_type(source_url).split("/")[-1] or "png"
        filename = f"{uuid.uuid4().hex}.{ext}"
        filepath = os.path.join(self._images_dir, filename)
        with open(filepath, "wb") as f:
            f.write(image_bytes)

        entry = {
            "id": uuid.uuid4().hex[:12],
            "file": filepath,
            "tags": str(post.get("tags") or ""),
            "source_url": source_url,
            "description": "",
            "created_at": int(time.time()),
        }
        index = self._load_index()
        index.append(entry)
        self._save_index(index)
        return entry

    async def _translate_tags(self, description: str, describe_cb) -> str:
        """中文描述 → 英文动漫 tag（用 LLM，失败降级英文原词）。"""
        if not describe_cb:
            return description.strip().replace(" ", "_")
        # 这里复用 describe_cb 同款 LLM 调用能力 —— 由 main.py 注入的
        # callback 负责翻译；若无 callback 直接返回原词。
        ...
```

> 注：为控制复杂度，`_translate_tags` 的实际 LLM 翻译与 `_quality_check` 的 vision 判断由 main.py 注入的回调完成，pose_library.py 只负责流程编排。因此 Task 1 的实现中 `_translate_tags` 和 `_quality_check` 在 main.py 的注入回调里实现（见 Task 3）。

**修正后的 Task 1 接口（最终版）：**

- [ ] **Step 1: 实现 PoseLibrary 类**（接口见上，`_translate_tags`/`_quality_check` 由外部回调注入）

```python
class PoseLibrary:
    def __init__(self, config, data_dir: str): ...

    async def query(self, keyword: str) -> list[dict]: ...

    async def search_and_download(self, description: str, count: int = 5,
                                  translate_cb=None, quality_cb=None) -> list[dict]:
        """
        translate_cb(description) -> str tags     # LLM 翻译成英文 tag
        quality_cb(image_url) -> bool             # vision 质检
        """
        ...
```

- [ ] **Step 2: 语法检查**

```bash
python -c "import ast; ast.parse(open('core/pose_library.py', encoding='utf-8').read()); print('OK')"
```

---

### Task 2: `models.py` + `_conf_schema.json` — 配置

**Files:**
- Modify: `models.py`, `_conf_schema.json`

**Interfaces:**
- Produces: `PluginConfig.pose_library: PoseLibraryConfig`（含 `enable`、`source`、`enable_quality_check`、`max_download_per_search`）

- [ ] **Step 1: models.py 增加 `PoseLibraryConfig` dataclass**

```python
@dataclass
class PoseLibraryConfig:
    enable: bool = True
    source: str = "gelbooru"          # gelbooru / rule34
    enable_quality_check: bool = True
    max_download_per_search: int = 5
```

`from_dict()` 从 `pose_library_config` 节读取：
```python
pose_lib_conf = _ensure_dict(config_dict, "pose_library_config")
# ... 读取 4 个字段并归一化写回 ...
pose_library=PoseLibraryConfig(
    enable=_to_bool(pose_lib_conf.get("enable", True)),
    source=str(pose_lib_conf.get("source", "gelbooru")).strip() or "gelbooru",
    enable_quality_check=_to_bool(pose_lib_conf.get("enable_quality_check", True)),
    max_download_per_search=_to_int(pose_lib_conf.get("max_download_per_search", 5), 5, minimum=1),
),
```

- [ ] **Step 2: _conf_schema.json 新增 `pose_library_config` 节**

```json
"pose_library_config": {
  "description": "姿势图库（LLM 自主维护）",
  "type": "object",
  "hint": "LLM 可通过 search_pose_image 工具搜索下载姿势参考图入库，供 ControlNet 画图使用。",
  "items": {
    "enable": {"description": "启用姿势图库", "type": "bool", "default": true},
    "source": {
      "description": "图源", "type": "string",
      "options": ["gelbooru", "rule34"],
      "hint": "gelbooru 偏 SFW；rule34 包含 NSFW 内容，适合成人场景。",
      "default": "gelbooru"
    },
    "enable_quality_check": {"description": "启用 vision 质检", "type": "bool", "default": true},
    "max_download_per_search": {"description": "每次搜索下载上限", "type": "int", "default": 5}
  }
}
```

- [ ] **Step 3: 语法检查**

```bash
python -c "import ast; ast.parse(open('models.py', encoding='utf-8').read()); json.load(open('_conf_schema.json', encoding='utf-8')); print('OK')"
```

---

### Task 3: `main.py` — 初始化 + 两个 LLM 工具

**Files:**
- Modify: `main.py`

**Interfaces:**
- Consumes: `PoseLibrary`（Task 1）、`PluginConfig.pose_library`（Task 2）
- Produces: `@llm_tool(name="search_pose_image")`、`@llm_tool(name="query_pose_library")`

- [ ] **Step 1: `_apply_runtime_config` 中初始化**

```python
self.pose_library = PoseLibrary(self.plugin_config, self.data_dir)
```

- [ ] **Step 2: 新增 LLM tag 翻译回调（复用副脑 provider 的 chat 调用）**

```python
async def _translate_pose_tags(self, description: str) -> str:
    """调用文本 LLM 把中文姿势描述翻译成英文动漫 tag 列表。"""
    # 复用 _describe_image 同款 provider 获取 + chat/completions 调用
    # prompt: "将以下姿势描述翻译成 3-8 个英文 danbooru 风格标签，逗号分隔：{description}"
    # 失败返回英文原词
```

- [ ] **Step 3: 新增 vision 质检回调**

```python
async def _check_pose_image(self, image_url: str) -> bool:
    """vision LLM 判断图片是否清晰完整人体姿势。"""
    # 复用 _describe_image 同款调用，prompt:
    # "这是一张动漫图片。它是否包含清晰、完整、无遮挡过多的人体姿势（适合作为姿势参考图）？仅回答 YES 或 NO。"
    # 返回 "YES" in 回答
```

- [ ] **Step 4: 两个 LLM 工具**

```python
@llm_tool(name="query_pose_library")
async def tool_query_pose_library(self, event, keyword: str) -> str:
    """查询本地姿势参考图库。画图前先调用，找到后把 file 路径作为 refs 传给 generate_image。
    Args:
        keyword (string): 姿势关键词，如 "公主抱"、"双人拥抱"、"princess carry"。
    """
    event = self._unwrap_message_event(event)
    permission_error = self._permission_denied_message(event)
    if permission_error:
        return permission_error
    entries = await self.pose_library.query(keyword)
    if not entries:
        return f"姿势库中未找到与「{keyword}」匹配的姿势。可调用 search_pose_image 搜索并入库。"
    lines = [f"姿势图 {i+1}: {e['file']}" for i, e in enumerate(entries)]
    return "姿势库匹配结果：\n" + "\n".join(lines) + "\n可将 file 路径作为 refs 传给 generate_image。"

@llm_tool(name="search_pose_image")
async def tool_search_pose_image(self, event, description: str, count: int = 5) -> str:
    """搜索并下载姿势参考图入库。当需要特定姿势（尤其双人互动）且 query_pose_library 无结果时调用。
    Args:
        description (string): 姿势描述，如 "双人公主抱，女生搂住男生脖子"。
        count (int): 下载数量，默认 5。
    """
    event = self._unwrap_message_event(event)
    permission_error = self._permission_denied_message(event)
    if permission_error:
        return permission_error
    entries = await self.pose_library.search_and_download(
        description, count,
        translate_cb=self._translate_pose_tags,
        quality_cb=self._check_pose_image,
    )
    if not entries:
        return f"未找到合适的「{description}」姿势图，请换一种描述重试。"
    lines = [f"姿势图 {i+1}: {e['file']} (tags: {e['tags'][:80]})" for i, e in enumerate(entries)]
    return "已入库姿势图：\n" + "\n".join(lines) + "\n可将 file 路径作为 refs 传给 generate_image 使用。"
```

- [ ] **Step 5: 语法检查**

```bash
python -c "import ast; ast.parse(open('main.py', encoding='utf-8').read()); print('OK')"
```

---

### Task 4: 验证

- [ ] 三个文件语法通过
- [ ] `PoseLibrary.query` 空索引返回空列表
- [ ] 配置默认值: `enable=True, source="gelbooru", enable_quality_check=True, max_download_per_search=5`
