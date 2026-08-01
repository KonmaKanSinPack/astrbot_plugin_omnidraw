# LLM 自主维护姿势图库 — 设计文档

## 背景

双人互动场景下，纯文本 prompt 生成容易出透视错误、多余肢体、动作不符合人体规律。ControlNet（OpenPose）可彻底解决——但需要姿势参考图。本功能让 **LLM 自行上网搜索、下载、质检、维护姿势图库**，生成时自动选用库内姿势图驱动 ControlNet。

## 数据流

```
用户: "画一个双人公主抱"
  → LLM 调用 query_pose_library("公主抱")
      → 库里有 → 返回姿势图路径 → LLM 传给 generate_image(refs=姿势图) → ControlNet 生成
      → 库里没有 → LLM 调用 search_pose_image("双人公主抱")
          1. LLM 把描述翻译成英文动漫 tag（如 "princess_carry"）
          2. 按配置开关选图源（gelbooru / rule34）调用公开 API 搜索
          3. 下载 top-N 张
          4. vision LLM 质量把关（是否清晰完整人体姿势）
          5. 通过者存入库: 图片文件 + 索引条目 {id, file, tags, source_url, description}
          6. 返回新入库条目 → LLM 用其生成
```

## 配置（`_conf_schema.json` 新增 `pose_library_config`）

| 字段 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `enable` | bool | true | 总开关 |
| `source` | string | "gelbooru" | 图源切换开关: `gelbooru` / `rule34`（options） |
| `enable_quality_check` | bool | true | vision LLM 质量把关开关 |
| `max_download_per_search` | int | 5 | 每次搜索下载上限 |
| `api_user_id` | string | "" | rule34 API user_id（账户选项页获取） |
| `api_key` | string | "" | rule34 API key |
| `quality_provider` | string | "" | 质检/翻译使用的 Provider 节点 ID；留空跟随副脑链路 |

> 更新记录（2026-08-01）: 新增 `api_user_id`/`api_key`（rule34 认证）与 `quality_provider`（质检模型可配置）。

## 图源 API（均免费公开、JSON 输出）

- Gelbooru: `https://gelbooru.com/index.php?page=dapi&s=post&q=index&json=1&tags={tags}&limit={n}`
- Rule34: `https://api.rule34.xxx/index.php?page=dapi&s=post&q=index&json=1&tags={tags}&limit={n}`
  - 需带认证参数 `user_id` + `api_key`（账户选项页 https://rule34.xxx/index.php?page=account&s=options 获取）

**返回格式差异（实现中确认）**：
- Gelbooru: `{"post": [...]}`（字典包裹）
- Rule34 带认证: 直接返回数组 `[{...}]`
- 解析逻辑必须同时兼容两种格式：

**Rule34 tag 搜索语法（实测确认）**：
- ✅ 1-2 个 tag（空格分隔 AND）: 正常返回
- ❌ 3 个及以上 tag: 直接被 API 拒绝（返回空响应）
- ❌ 逗号分隔的 tag 串: 同样失败
- 因此翻译逻辑必须输出 **1-2 个、空格分隔** 的 tag（见下方 tag 翻译核心代码）
```python
data = await resp.json()
# rule34 带认证时直接返回数组 [{...}]；gelbooru 返回 {"post": [...]}
if isinstance(data, list):
    posts = data
elif isinstance(data, dict):
    posts = data.get("post")
else:
    posts = None
if not isinstance(posts, list):
    return []
```

每项含 `file_url`、`tags`、`score` 等。过滤条件：`score:>=25`、非动画/视频（排除 `swf`）、限制尺寸 ≥ 500px。

## 模块设计

### 新增 `core/pose_library.py` — `PoseLibrary` 类

```python
class PoseLibrary:
    """LLM 自主维护的姿势参考图库。"""

    def __init__(self, config: PluginConfig, data_dir: str):
        # 目录结构:
        #   {data_dir}/pose_library/images/{uuid}.{ext}   ← 姿势图片文件
        #   {data_dir}/pose_library/index.json            ← 索引 [{id, file, tags, source_url, description}]

    async def query(self, keyword: str) -> list[dict]:
        """按关键词模糊匹配本地索引（tags/description），返回匹配条目。"""
        # 读 index.json → 过滤 tags 或 description 包含 keyword 的条目

    async def search_and_download(self, description: str, count: int = 5,
                                  describe_cb=None) -> list[dict]:
        """描述 → tag 翻译 → 图源搜索 → 下载 → 质检 → 入库 → 返回新条目。"""
        # 1. describe_cb: 调用 LLM 把中文描述转英文动漫 tag（可复用副脑 provider）
        # 2. 按 self._source 选 API，拼 tags + 质量过滤
        # 3. aiohttp 搜索 → 取前 count 张 file_url
        # 4. 下载图片字节 → 临时文件
        # 5. 质量把关: 若启用，调 vision LLM 判断"是否清晰完整人体姿势"，
        #    过滤不合格的（复用 main.py 的 _describe_image 同款调用）
        # 6. 入库: 图片保存到 images/，索引追加条目并写回 index.json
        # 7. 返回 [{id, file, tags, source_url, description}]
```

### `main.py` — LLM 调用封装（质检走 AstrBot provider）

> 更新记录（2026-08-01）: 质检不再走副脑 HTTP chat/completions，
> 改用 AstrBot provider 抽象层 `text_chat()`（支持 vision）。

```python
async def _judge_llm(self, prompt: str, image_urls: Optional[list] = None,
                     system_prompt: str = "") -> str:
    """用 AstrBot provider 调用 LLM（支持 vision）。失败返回空串。

    优先用配置的 quality_provider 节点，否则用 AstrBot 当前使用的文本 provider。
    通过 provider.text_chat() 调用（AstrBot 抽象层），不自行拼 HTTP。
    """
    provider = None
    provider_id = getattr(self.plugin_config.pose_library, "quality_provider", "")
    if provider_id:
        try:
            getter = getattr(self.context, "get_provider", None)
            if callable(getter):
                provider = getter(provider_id)
        except Exception:
            provider = None
    if provider is None:
        try:
            provider = self.context.get_using_provider()
        except Exception:
            provider = None
    if provider is None or not hasattr(provider, "text_chat"):
        return ""

    try:
        llm_resp = await provider.text_chat(
            prompt=prompt, session_id=None, contexts=[],
            image_urls=image_urls or [], func_tool=None,
            system_prompt=system_prompt,
        )
        return str(getattr(llm_resp, "completion_text", "") or "").strip()
    except Exception as exc:
        logger.warning(f"[OmniDraw] AstrBot provider 质检调用失败: {exc}")
        return ""
```

质检（`_check_pose_image`）传 `image_urls=[image_url]` 走 vision；
tag 翻译（`_translate_pose_tags`）纯文本调用。质检调用失败时保守放行（返回 True），避免误杀全部图片。

### `main.py` — 两个新 LLM 工具

```python
@llm_tool(name="search_pose_image")
async def tool_search_pose_image(self, event, description: str, count: int = 5) -> str:
    """
    搜索并下载姿势参考图入库。当画图需要特定姿势（尤其双人互动）
    而 query_pose_library 找不到时调用。返回入库姿势图的信息，
    可将其 file 路径作为 refs 传给 generate_image。
    """
    # event 解包 → 权限检查（复用 _permission_denied_message）
    # self.pose_library.search_and_download(description, count, describe_cb=self._describe_image)
    # 返回 JSON 字符串或文本列表给 LLM

@llm_tool(name="query_pose_library")
async def tool_query_pose_library(self, event, keyword: str) -> str:
    """
    查询本地姿势参考图库。画图前先调用此工具寻找已有姿势图，
    找到后将返回的 file 路径作为 refs 传给 generate_image。
    """
    # self.pose_library.query(keyword) → 返回条目文本
```

### `main.py` — 初始化

```python
# __init__ 或 _apply_runtime_config 中:
self.pose_library = PoseLibrary(self.plugin_config, self.data_dir)
```

### `models.py` — PluginConfig

新增 `PoseLibraryConfig` dataclass（或直接字段）:
```python
@dataclass
class PoseLibraryConfig:
    enable: bool = True
    source: str = "gelbooru"     # gelbooru / rule34
    enable_quality_check: bool = True
    max_download_per_search: int = 5
```
`from_dict()` 从 `pose_library_config` 节读取。

### `_conf_schema.json` — 配置页

新增 `pose_library_config` 节（4 个字段，含图源切换 options）。

## 关键实现细节

1. **tag 翻译**：用 AstrBot provider 的 LLM（`_judge_llm`），prompt 要求输出 **1-2 个**英文 danbooru tag，**空格分隔**；后处理把逗号转空格、截断到最多 2 个（rule34 API 限制）；失败降级用英文原词。

```python
async def _translate_pose_tags(self, description: str) -> str:
    """把中文姿势描述翻译成英文动漫 tag（最多 2 个，空格分隔 AND）。

    rule34 API 实测: 1-2 个 tag（空格分隔 AND）正常返回；
    3 个及以上 tag 会被 API 拒绝（返回空响应）。因此必须限制数量。
    """
    prompt = (
        "将以下姿势描述翻译成 1-2 个最核心的英文动漫标签"
        "（danbooru 风格，使用下划线连接短语），"
        "只输出标签本身，空格分隔，不要任何解释或多余内容。\n"
        f"描述: {description}"
    )
    content = await self._judge_llm(prompt)
    # 清洗: 逗号转空格（兼容 LLM 输出逗号分隔），去掉多余符号
    import re as _re
    cleaned = _re.sub(r"[^\w\s_-]", " ", str(content or ""))
    tags = [tag for tag in cleaned.split() if tag][:2]  # rule34 最多 2 个 tag
    if tags:
        return " ".join(tags)
    return str(description).strip().replace(" ", "_")[:200]
```
2. **质量把关 prompt**：`"这是一张动漫图片。它是否包含清晰、完整、无遮挡过多的人体姿势（适合作为姿势参考图）？仅回答 YES 或 NO。"` — 返回含 YES 的保留。
3. **下载复用**：用 aiohttp 下载，失败重试 2 次，跳过损坏图（PIL 无法打开）。
4. **索引持久化**：每次入库写回 index.json（原子写: tmp + os.replace）。
5. **重复入库防重**：同 source_url 已存在则跳过。
6. **返回给 LLM 的信息**：简洁文本，含 file 路径（LLM 可直接作为 refs 使用）、tags、来源。
