# LLM 工具返回结果优化 — 设计文档

## 背景

当前 omnidraw 的 LLM 工具（`generate_image`、`generate_selfie`）在生图成功后总是返回固定的系统提示文本，失败时抛出异常信息。用户无法控制 LLM 对生成结果的描述行为。

需求：增加配置开关 `describe_generated_image`，开启时主动调用一次 vision LLM API 对生成的图像进行描述并返回描述文字，关闭时返回简单成功文本。

## 配置开关

- **字段名**: `describe_generated_image`
- **类型**: `bool`
- **默认值**: `true`（开启）
- **位置**: `reply_config` 配置节

## 行为定义

### 生成成功

**开关 = true**（开启描述）：
1. 图片照常发送到聊天
2. 主动调用一次 vision LLM API，将生成的图像以 data URL 格式发给该 LLM
3. LLM 返回中文图像描述（如 "这是一张动漫风格的少女肖像，她有着棕色双马尾..."）
4. 工具将这段描述文字作为返回值 → 调用方 LLM 直接看到自然语言描述

API 调用复用 optimizer 的 provider 配置：
- endpoint: `build_chat_completions_endpoint(provider.base_url)` → `/v1/chat/completions`
- model: `optimizer_model` 或 provider.model
- vision 格式: OpenAI-compatible，`content` 为数组 `[text, image_url]`
- prompt: "请用中文简要描述这张图片的内容，包括主体、风格、场景和氛围。控制在 100 字以内。"
- 降级: API 调用失败时回退到简单成功文本

**开关 = false**（关闭描述）：返回 `"已成功生成并发送 {n} 张图片。"`

### 生成失败

无论开关状态，统一返回脱敏后的失败报告：
- `"画图失败：{脱敏错误信息}"`
- `"自拍失败：{脱敏错误信息}"`

## 改动范围

### 1. `models.py` — PluginConfig
- 新增 `describe_generated_image: bool` 字段（默认 `true`）
- `from_dict()` 从 `reply_config.describe_generated_image` 读取，默认 `true`

### 2. `main.py` — 新增 `_describe_image` 方法 + 修改 tool 返回逻辑

**`_describe_image(image_url)` 核心代码：**
```python
async def _describe_image(self, image_url: str) -> str:
    """调用 vision LLM 描述图片内容，返回中文描述文本。"""
    # 1. 获取 provider（复用 optimizer 链路配置）
    provider = self.plugin_config.get_provider(chain[0]) if chain else ...
    
    # 2. 构建 chat/completions endpoint
    endpoint = build_chat_completions_endpoint(provider.base_url)
    
    # 3. 如果图片是 HTTP URL，下载并转为 data URL（vision API 需要）
    if not image_url.startswith("data:"):
        # 下载 → base64 编码 → data:image/...;base64,...
    
    # 4. 构造 vision API 请求
    payload = {
        "model": optimizer_model,
        "messages": [{
            "role": "user",
            "content": [
                {"type": "text", "text": "请用中文简要描述这张图片..."},
                {"type": "image_url", "image_url": {"url": data_url}}
            ]
        }],
        "max_tokens": 300,
    }
    
    # 5. POST 请求 → 提取 choices[0].message.content 返回
    # 失败返回空字符串 ""
```

**tool 成功路径修改：**
```python
if self.plugin_config.describe_generated_image:
    image_url = self._get_image_result_url(valid_results[0])
    if image_url:
        desc = await self._describe_image(image_url)
        if desc:
            return desc  # ← LLM 描述文本
# 降级：返回简单成功文本
return f"已成功生成并发送 {sent} 张图片。"
```

**tool 失败路径修改：**
```python
return f"画图失败：{self._safe_plugin_error_message(exc)}"  # 脱敏
```

## 数据流

```
LLM 调用 tool_generate_image
  → 生成图片，发送到聊天
  → describe_generated_image == true?
    → YES: _describe_image(image_url)
      → HTTP POST /v1/chat/completions (vision)
      → 返回中文描述文字
    → NO: 返回 "已成功生成并发送 X 张图片。"
  → LLM 收到返回值，展示给用户
```
