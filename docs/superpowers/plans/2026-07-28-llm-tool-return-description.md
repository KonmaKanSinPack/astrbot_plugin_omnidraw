# LLM 工具返回结果优化 — 实现计划

> **Goal:** 增加 `describe_generated_image` 配置开关，控制 LLM 工具返回值是否包含图像数据供 LLM 描述。

**Architecture:** 在 `PluginConfig` 新增 bool 字段，从 `reply_config` 读取。`tool_generate_image` 和 `tool_generate_selfie` 检查开关：开启时返回图像 URL 引导 LLM 描述，关闭时返回简单成功文本，失败时统一返回脱敏错误信息。

**Files:**
- Modify: `models.py` (PluginConfig + from_dict)
- Modify: `main.py` (tool_generate_image, tool_generate_selfie)

---

### Task 1: 添加 `describe_generated_image` 配置字段

**Files:** `models.py`

- [ ] **Step 1: 在 PluginConfig dataclass 末尾新增字段**

```python
# 在 show_request_model: bool 之后添加
describe_generated_image: bool
```

- [ ] **Step 2: 在 from_dict() 中归一化并传入**

在 reply_conf 归一化区域（`reply_conf["selfie_error_message"] = ...` 之后）添加：
```python
reply_conf["describe_generated_image"] = _to_bool(
    reply_conf.get("describe_generated_image", True)
)
```

在 `return cls(...)` 参数末尾添加：
```python
describe_generated_image=_to_bool(reply_conf.get("describe_generated_image", True)),
```

---

### Task 2: 修改 tool_generate_image 返回逻辑

**Files:** `main.py`

- [ ] **Step 1: 修改成功返回路径**

将：
```python
return f"系统提示：已成功下发 {sent} 张图。"
```
替换为：
```python
if self.plugin_config.describe_generated_image:
    image_refs = []
    for r in valid_results:
        url = self._get_image_result_url(r)
        if url:
            image_refs.append(url)
    if image_refs:
        return f"已生成并发送 {sent} 张图片。以下是生成的图像，请用中文简要描述图像内容：\n" + "\n".join(image_refs)
return f"已成功生成并发送 {sent} 张图片。"
```

- [ ] **Step 2: 修改失败返回路径**

将：
```python
return f"系统提示：画图失败 ({exc})。"
```
替换为脱敏版本：
```python
return f"画图失败：{self._safe_plugin_error_message(exc)}"
```

---

### Task 3: 修改 tool_generate_selfie 返回逻辑

**Files:** `main.py`

- [ ] **Step 1: 修改成功返回路径**（同 Task 2，文案改"图片"为"自拍"）

- [ ] **Step 2: 修改失败返回路径**（同 Task 2，文案改"画图"为"自拍"）

---

### Task 4: 验证

- [ ] 语法检查: `python -c "import ast; ast.parse(open('models.py').read()); ast.parse(open('main.py').read()); print('OK')"`
- [ ] 确认默认值为 `true`
- [ ] 确认 `return_result=true` 模式不受影响
