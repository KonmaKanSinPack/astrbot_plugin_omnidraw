"""LLM 自主维护的姿势参考图库。"""

import asyncio
import json
import os
import time
import uuid

import aiohttp
from astrbot.api import logger

from ..utils import guess_image_content_type

GELBOORU_API = "https://gelbooru.com/index.php"
RULE34_API = "https://api.rule34.xxx/index.php"
LIBRARY_DIR_NAME = "pose_library"
INDEX_NAME = "index.json"
IMAGES_DIR_NAME = "images"
MIN_IMAGE_SIZE = 500        # 过滤小于 500px 的图
SCORE_FLOOR = 25            # 过滤低分图
MAX_DOWNLOAD_RETRY = 2
DOWNLOAD_HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
        "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
    )
}


class PoseLibrary:
    """LLM 自主维护的姿势参考图库。

    目录结构:
      {data_dir}/pose_library/images/{uuid}.{ext}   <- 姿势图片文件
      {data_dir}/pose_library/index.json            <- 索引 [{id, file, tags, source_url, description, created_at}]
    """

    def __init__(self, config, data_dir: str):
        self._config = config
        self._root = os.path.join(data_dir, LIBRARY_DIR_NAME)
        self._images_dir = os.path.join(self._root, IMAGES_DIR_NAME)
        self._index_path = os.path.join(self._root, INDEX_NAME)
        os.makedirs(self._images_dir, exist_ok=True)

    # ---------- 查询 ----------

    async def query(self, keyword: str) -> list[dict]:
        """按关键词模糊匹配本地索引（tags/description），返回匹配条目。"""
        keyword = str(keyword or "").strip().lower()
        if not keyword:
            return []
        results = []
        for entry in self._load_index():
            haystack = f"{entry.get('tags', '')} {entry.get('description', '')}".lower()
            if keyword in haystack:
                results.append(entry)
        return results

    # ---------- 搜索 + 下载 + 质检 + 入库 ----------

    async def search_and_download(
        self,
        description: str,
        count: int = 5,
        translate_cb=None,
        quality_cb=None,
    ) -> list[dict]:
        """描述 -> tag 翻译 -> 图源搜索 -> 下载 -> 质检 -> 入库。返回新入库条目。

        translate_cb(description: str) -> str   # 把中文描述翻译成英文动漫 tag
        quality_cb(image_url: str) -> bool      # vision 质检是否适合做姿势参考图
        """
        if not self._config.enable:
            return []
        if not description or not str(description).strip():
            return []

        count = max(1, min(int(count), int(self._config.max_download_per_search or 5)))
        tags = await self._translate_tags(description, translate_cb)
        posts = await self._search_api(tags, count)
        if not posts:
            logger.warning(f"[OmniDraw] 姿势图库搜索无结果: {tags}")
            return []

        existing_urls = {e.get("source_url") for e in self._load_index()}
        entries = []
        # trust_env=True: 走系统代理（rule34/gelbooru 为国外站点，本机直连超时）
        async with aiohttp.ClientSession(trust_env=True) as session:
            for post in posts[:count]:
                url = str(post.get("file_url") or "").strip()
                if not url or url in existing_urls:
                    continue
                try:
                    image_bytes = await self._download(session, url)
                    if not image_bytes:
                        continue
                    if not await self._valid_dimensions(image_bytes):
                        continue
                    # vision 质检（可选）
                    if self._config.enable_quality_check and quality_cb:
                        if not await quality_cb(url):
                            logger.info(f"[OmniDraw] 姿势图质检未通过，跳过: {url}")
                            continue
                    entry = self._store(image_bytes, url, post)
                    entries.append(entry)
                    existing_urls.add(url)
                except Exception as exc:
                    logger.warning(f"[OmniDraw] 姿势图入库失败 {url}: {exc}")
                    continue

        logger.info(f"[OmniDraw] 姿势图库本次新增 {len(entries)} 张")
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
        """原子写索引文件（tmp + os.replace）。"""
        tmp_path = f"{self._index_path}.{uuid.uuid4().hex}.tmp"
        try:
            with open(tmp_path, "w", encoding="utf-8") as f:
                json.dump(index, f, ensure_ascii=False, indent=2)
            os.replace(tmp_path, self._index_path)
        finally:
            if os.path.exists(tmp_path):
                try:
                    os.remove(tmp_path)
                except OSError:
                    pass

    def _store(self, image_bytes: bytes, source_url: str, post: dict) -> dict:
        """图片落盘 + 索引追加，返回新条目。"""
        content_type = guess_image_content_type(source_url)
        ext = content_type.split("/")[-1] or "png"
        if ext not in ("png", "jpg", "jpeg", "webp", "gif"):
            ext = "png"
        filename = f"{uuid.uuid4().hex}.{ext}"
        filepath = os.path.join(self._images_dir, filename)
        with open(filepath, "wb") as f:
            f.write(image_bytes)

        entry = {
            "id": uuid.uuid4().hex[:12],
            "file": os.path.abspath(filepath),
            "tags": str(post.get("tags") or ""),
            "source_url": source_url,
            "description": "",
            "created_at": int(time.time()),
        }
        index = self._load_index()
        index.append(entry)
        self._save_index(index)
        return entry

    async def _translate_tags(self, description: str, translate_cb) -> str:
        """中文描述 -> 英文动漫 tag。有 callback 用 LLM，否则英文原词。"""
        if translate_cb:
            try:
                translated = await translate_cb(description)
                translated = str(translated or "").strip()
                if translated:
                    return translated
            except Exception as exc:
                logger.warning(f"[OmniDraw] 姿势 tag 翻译失败，使用原文: {exc}")
        # 降级: 空格转下划线
        return str(description).strip().replace(" ", "_")[:200]

    async def _search_api(self, tags: str, limit: int) -> list[dict]:
        """调用图源 API 搜索，返回 post 列表。图源由配置 source 切换。

        rule34 需带 user_id + api_key（账户选项页获取）；未配置时匿名请求。
        在 tag 后追加 meta 过滤语法 score:>=N，让服务端直接只返回高分图。
        """
        base_url = RULE34_API if str(self._config.source).lower() == "rule34" else GELBOORU_API
        # 追加 rule34/gelbooru 通用 meta 过滤: score:>=SCORE_FLOOR
        # 如 "doggystyle" → "doggystyle score:>=25"
        search_tags = f"{tags} score:>={SCORE_FLOOR}" if tags else f"score:>={SCORE_FLOOR}"
        params = {
            "page": "dapi",
            "s": "post",
            "q": "index",
            "json": "1",
            "tags": search_tags,
            "limit": limit,
        }
        # rule34 API key 认证
        if str(self._config.source).lower() == "rule34":
            if getattr(self._config, "api_user_id", "") and getattr(self._config, "api_key", ""):
                params["user_id"] = str(self._config.api_user_id)
                params["api_key"] = str(self._config.api_key)
        try:
            timeout = aiohttp.ClientTimeout(total=20)
            # trust_env=True: 走系统代理（rule34/gelbooru 为国外站点，本机直连超时）
            async with aiohttp.ClientSession(timeout=timeout, trust_env=True) as session:
                async with session.get(base_url, params=params, headers=DOWNLOAD_HEADERS) as resp:
                    if resp.status != 200:
                        logger.warning(f"[OmniDraw] 图源 API 返回 {resp.status}")
                        return []
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
            # 过滤: 非动画、非视频、尺寸、评分
            filtered = []
            for post in posts:
                if not isinstance(post, dict):
                    continue
                url = str(post.get("file_url") or "").strip()
                if not url:
                    continue
                if str(post.get("file_type") or "").lower() in ("swf", "video", "webm", "mp4"):
                    continue
                try:
                    if float(post.get("score", 0)) < SCORE_FLOOR:
                        continue
                except (TypeError, ValueError):
                    pass
                filtered.append(post)
            return filtered
        except Exception as exc:
            logger.warning(f"[OmniDraw] 姿势图库搜索失败: {exc}")
            return []

    async def _download(self, session: aiohttp.ClientSession, url: str) -> bytes:
        """下载图片字节，失败重试 MAX_DOWNLOAD_RETRY 次。"""
        for attempt in range(1, MAX_DOWNLOAD_RETRY + 2):
            try:
                async with session.get(url, headers=DOWNLOAD_HEADERS, timeout=aiohttp.ClientTimeout(total=15)) as resp:
                    if resp.status != 200:
                        raise RuntimeError(f"HTTP {resp.status}")
                    return await resp.read()
            except Exception as exc:
                if attempt > MAX_DOWNLOAD_RETRY:
                    logger.warning(f"[OmniDraw] 姿势图下载失败: {url} ({exc})")
                    return b""
                await asyncio.sleep(0.5 * attempt)
        return b""

    async def _valid_dimensions(self, image_bytes: bytes) -> bool:
        """PIL 校验图片可打开且最短边 >= MIN_IMAGE_SIZE。"""
        try:
            import io
            from PIL import Image
            with Image.open(io.BytesIO(image_bytes)) as img:
                w, h = img.size
            return min(w, h) >= MIN_IMAGE_SIZE
        except Exception:
            return False
