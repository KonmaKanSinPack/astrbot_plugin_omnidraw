# LLM 工具返回结果优化 — 设计文档

## 背景

当前 omnidraw 的 LLM 工具（`generate_image`、`generate_selfie`）在生图成功后总是返回固定的系统提示文本（如"系统提示：已成功下发 X 张图。"），失败时返回异常信息。用户无法控制 LLM 对生成结果的描述行为。

需求：增加配置开关，让用户选择 LLM 工具返回生图结果时是否让 LLM 对生成的图像进行描述。

## 配置开关

- **字段名**: `describe_generated_image`
- **类型**: `bool`
- **默认值**: `true`（开启）
- **位置**: `reply_config` 配置节（与现有 `draw_pending_message` 等 UI 文本配置放在一起）

## 行为定义

### 生成成功

**开关 = true**（开启描述）：
- 图片仍然发送到聊天
- 工具返回值包含生成的图像数据（data URL / HTTP URL），引导 LLM 对图像进行中文描述
- 格式: `"已生成并发送 {n} 张图片。以下是生成的图像，请用中文简要描述图像内容：\n{image_url}"`

**开关 = false**（关闭描述）：
- 图片仍然发送到聊天
- 工具返回简单成功文本
- 格式: `"已成功生成并发送 {n} 张图片。"`

### 生成失败

无论开关状态，统一返回失败报告：
- 格式: `"画图失败：{error_message}"`
- 错误信息经过脱敏处理（API Key、Token 等）

## 改动范围

### 1. `models.py` — PluginConfig
- 新增 `describe_generated_image: bool` 字段（默认 `true`）
- `from_dict()` 从 `reply_config.describe_generated_image` 读取，默认 `true`
- 归一化写入 `reply_conf["describe_generated_image"]`

核心代码：
```python
# PluginConfig dataclass 新增字段
@dataclass
class PluginConfig:
    # ... 现有字段 ...
    describe_generated_image: bool  # 新增

# from_dict() 中读取（与其他 reply_conf 字段一起归一化）
reply_conf["describe_generated_image"] = _to_bool(
    reply_conf.get("describe_generated_image", True)
)

# 传入构造器
return cls(
    # ... 现有参数 ...
    describe_generated_image=_to_bool(reply_conf.get("describe_generated_image", True)),
)
```

### 2. `main.py` — tool_generate_image / tool_generate_selfie

修改成功和失败返回逻辑。

核心代码（以 `tool_generate_image` 为例）：
```python
# --- 成功路径 ---
sent = await self._send_generated_images(event, valid_results)
self._record_generated_images(event, sent)

if self.plugin_config.describe_generated_image:
    # 开关开启：收集图像 URL，让 LLM 描述
    image_refs = []
    for r in valid_results:
        url = self._get_image_result_url(r)
        if url:
            image_refs.append(url)
    if image_refs:
        return (
            f"已生成并发送 {sent} 张图片。"
            f"以下是生成的图像，请用中文简要描述图像内容：\n"
            + "\n".join(image_refs)
        )
# 开关关闭 或 无图像 URL：返回简单成功文本
return f"已成功生成并发送 {sent} 张图片。"

# --- 失败路径 ---
except Exception as exc:
    logger.error(f"[OmniDraw] LLM 画图工具失败: {exc}", exc_info=True)
    # return_result 模式保持不变（JSON）
    if self._plugin_bool(return_result, default=False):
        return json.dumps({...})
    # 统一返回脱敏后的失败文本
    return f"画图失败：{self._safe_plugin_error_message(exc)}"
```

`tool_generate_selfie` 同理，文案中"图片"替换为"自拍"。
