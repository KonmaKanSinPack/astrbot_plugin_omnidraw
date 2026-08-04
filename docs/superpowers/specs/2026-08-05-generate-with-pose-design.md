# generate_with_pose：姿势生图五步编排工具

> 日期: 2026-08-05
> 项目: astrbot_plugin_omnidraw
> 状态: 设计已确认（对话批准），本文件为正式规格

---

## 1. 目标

LLM 画姿势受控图时遵循固定五步流程，由**单个编排工具**保证流程 100% 走完（LLM 只调一次，不依赖 LLM 自主编排工具的自觉性）：

```
① 搜 pose → ② 拟 prompt → ③ 视觉查冲突 → ④ 优化 prompt → ⑤ 生成
```

参考学术范式（2026 调研）：CRAFT 的 DVQ（YES/NO 结构化检查 + 理由）模式、SIDiffAgent 的经验记忆机制。本设计把 DVQ 检查**前置到生成之前**——检查参考 pose 图而非生成图，失败不用重生成，成本远低于生成后自检。

## 2. 工具接口

```python
@llm_tool(name="generate_with_pose")
async def tool_generate_with_pose(
    self,
    event: AstrMessageEvent,
    intent: str,
    count: int = 1,
    aspect_ratio: str = "",
    size: str = "",
) -> str:
    """按固定流程生成姿势受控图：搜姿势 → 拟提示词 → 视觉检查冲突 → 优化 → 生成。
    Args:
        intent (string): 用户想生成的画面需求（自然语言，含角色/动作/双人交互等）。
        count (int): 生成数量，默认 1。
        aspect_ratio (string): 宽高比例，如 1:1、3:4、9:16。
        size (string): 分辨率，如 1024x1024。
    """
```

docstring 引导：**需要姿势受控生成时优先用此工具**（比手动组合多个工具更可靠）。

## 3. 五步流水线（内部实现）

### ① 搜 pose
- `intent` → 文本 LLM 提取 1-2 个 booru 标签（复用 `_translate_pose_tags` 的格式规则：小写下划线、≤2 个）
- 本地优先：`pose_library.query(tags)` → 未命中 → `search_and_download(tags, translate_cb, quality_cb)`（复用合并后的单工具逻辑）
- 都无 → **降级**（见 §5）

### ② 拟稿
- 文本 LLM（`_judge_llm` 纯文本调用，不花视觉钱）：
  - 输入：intent + pose tags + **经验桶命中**（若有，见 §6）
  - 输出：完整正向 prompt（含质量词结构，同 Advanced_V37 的 suffix 风格）
- 职责分离：**拟稿不看图**（省视觉调用），冲突由 ③ 兜底修正

### ③ DVQ 检查（新方法 `_check_pose_compatibility`）
- 视觉 LLM（`quality_provider`，破甲提示词，`_to_vision_data_url` 转图）
- 逐项 YES/NO 检查清单（CRAFT 式，每个 NO 附理由 + 修改建议）：
  ```
  a. 图中人物数量与 prompt 要求一致？
  b. 所有肢体完整可见（无被遮挡画不出的部分）？
  c. 身体朝向/视角与 prompt 动作描述兼容？
  d. 服装/道具与 prompt 描述无冲突？
  e. 该姿势能承载 prompt 要求的动作（如公主抱要求两人）？
  ```
- 返回视觉 LLM 的**完整原始响应**（含 YES 项、NO 理由与建议），不做二次加工
- 判定通过/有冲突只做轻量检查（是否含 NO 行），不解析改写响应内容

### ④ 优化（≤2 轮）
- 文本 LLM 按完整检查结果（理由/建议）定向修改 prompt → 回到 ③ 复检
- 轮数上限：2（即最多 2 次视觉调用），超限用最后一版直接生成

### ⑤ 生成 + 经验回写
- 最终 prompt + pose file → 复用现有 `generate_image` 的 refs+pose 链路（内部调用生成 + 下发图片，不重复造轮子）
- 成功后：最终 prompt 回写经验桶（§6）

## 4. 返回格式

- 成功：图片已下发 + 简短说明（用了哪个 pose、检查了几轮）
- 降级：明确告知 LLM "已降级为普通生成，原因：xxx"（LLM 向用户说明时带出）

## 5. 降级策略（用户确认）

任一环节失败：
- 搜不到 pose / 视觉 provider 不可用 / 检查超时 / 拟稿失败 / 经验桶读写异常
→ 自动降级为普通无 pose 生成（`generate_image` 不含 refs）
→ 返回文案明确写"已降级为普通生成，原因：xxx"

## 6. 经验库（tag 桶，SIDiffAgent 范式）

**为什么按 tag 桶而非单图**：pose 库图多、搜索按关键词模糊匹配，同一张图下次大概率搜不回（命中率低）。按"搜索 tags"聚合则命中率高——同一类需求（如"公主抱"）每次提取的 tags 一致，桶稳定命中。

- **存储**：`<data_dir>/pose_library/experience.json`，结构：
  ```json
  {
    "princess_carry": ["<成功prompt1>", "<成功prompt2>", ...],   // 每桶保留最近 5 条
    "standing_sex leg_raised": [...]
  }
  ```
- **写入**：⑤ 成功后，key = 本次搜索用的 tags，value 前插 + 截断 5 条
- **读取**：② 拟稿前，用提取的 tags 查桶 → 命中用最近一条成功 prompt 做种子（结合新 intent 调整）
- **原子写**：tmp + os.replace（复用 index.json 的既有模式）
- 冷启动：某姿势类别第一次使用时桶为空 → 从零拟稿（正常）

## 7. 成本上限

| 调用 | 次数/单 | 说明 |
|---|---|---|
| 视觉 | ≤2 | DVQ 检查 + 至多 1 次复检 |
| 文本 | 2-3 | 标签提取、拟稿、优化 |
| 生图 | 1-2 | 正常 1 次；降级时也是 1 次 |

不做"生成后自检"（CRAFT 闭环的下一阶段，YAGNI）。

## 8. 复用清单

**复用（不重复造轮子）**：
- `pose_library.query` / `search_and_download`（含去重、质检、原子入库）
- `_judge_llm`（quality_provider 视觉链路 + 纯文本调用）
- `_to_vision_data_url`（文件 → data URL）
- `_translate_pose_tags` 的 booru 规则
- `generate_image` 的 refs + pose 生成链路（`_run_text2img_generation` / 图片下发）
- 破甲提示词风格（§3.③ 沿用）

**新增**：
- `tool_generate_with_pose`（编排入口）
- `_check_pose_compatibility`（DVQ 检查 + 破甲）
- `_draft_pose_prompt` / `_refine_pose_prompt`（两个文本调用封装）
- `_extract_pose_tags`（intent → booru 标签，复用翻译规则）
- experience.json 读写（`_load_experience` / `_save_experience` / `_get_experience_seed`）

## 9. 错误处理

- 每一步 try/except，失败即降级（§5），不中断向用户出图
- 视觉调用失败区分：provider 不可用 vs 检查超时 → 都降级，但原因文案不同
- 日志：每步打点（`[OmniDraw] generate_with_pose 步骤N 结果: ...`），便于排查

## 10. 测试

1. **分步单测**：标签提取、拟稿、DVQ 结果解析（YES/NO → 理由列表）
2. **降级路径**：mock 视觉 provider 抛错 → 确认降级 + 通知文案
3. **经验桶**：写入/读取/截断 5 条/原子写
4. **集成**：真实"双人公主抱"→ 检查轮数 ≤2、出图、经验回写成功
5. **回归**：`search_pose_image`（合并后单工具）不受影响

## 11. 部署

- 同步 `main.py` 到生产实例 + 重启插件
- 无新配置项（复用 `quality_provider`）

## 12. 后续可做（不在本次范围）

- 生成后自检闭环（CRAFT 式：生成图 → DVQ → 定向重生成）
- Fuse 模式姿势工作流（UniControl-XL，已单独集成）
