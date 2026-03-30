# Structured Extraction Prompt Guidelines

> 一次性结构化提取任务（非 ReAct agent）的 prompt 设计指南。
> 基于 Anthropic Claude 4 Best Practices + 实践经验。

Sources:
- [Claude 4 Prompting Best Practices](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices)
- [Multishot Prompting](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting)
- [XML Tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags)
- [Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use)

---

## 适用场景

单次 LLM 调用 + `tool_choice` 强制结构化输出。不涉及多轮对话、tool routing、ReAct 循环。

典型例子：从会议 transcript 提取 TODO 列表。

**与 agent-prompt-guidelines.md 的区别：**

| | Agent Prompt | Structured Extraction |
|---|---|---|
| 调用次数 | 多轮 ReAct | **一次** |
| 输出控制 | prompt 引导 + tool routing | **tool_choice 强制 schema** |
| 最有效手段 | Role + Principles + Context | **Few-shot examples** |
| Prompt 结构 | Role → Principles → Context → Constraints | **Instructions → Examples → Input** |

---

## 1. 核心原则

### Examples > Instructions

> "Examples are one of the most reliable ways to steer Claude's output."
> — Anthropic Prompting Best Practices

对于结构化提取，**3-5 个好的 examples 比一页 instructions 更有效**。Claude 4 模型从 examples 中泛化能力极强。

### Tool Schema 管结构，Prompt 管质量

三层分工：
- **tool_choice** — 强制调用指定 tool（100% 可靠）
- **tool schema** — 约束输出结构（字段名、类型、必填）
- **system prompt + examples** — 引导输出**内容质量**（怎么写 title、description 怎么组织）

Tool schema 保证 LLM 不会返回错误格式，但不保证内容好。内容质量由 prompt 中的 examples 决定。

### XML Tags 分隔结构

用 `<instructions>`、`<examples>`、`<example>`、`<input>` 等 XML tags 让 Claude 不混淆：

```
<instructions>
提取规则...
</instructions>

<examples>
<example>
输入: ...
输出: ...
</example>
</examples>

<input>
{actual_data}
</input>
```

---

## 2. Prompt 结构

推荐结构（从上到下）：

```
1. Role — 一句话身份定义
2. Capabilities — 系统能做什么（帮助 LLM 判断什么是 actionable）
3. Instructions — 简短的提取规则
4. Examples — 3-5 个高质量 few-shot（最重要的部分）
5. Input — 实际输入数据（transcript）放最后
6. Output Format — 指向 tool（"Use extract_todos tool"）
```

**长文本数据（transcript）放在 prompt 末尾，紧接着 output 指令。** Anthropic 测试显示 queries at the end 可提升 30% 响应质量。

---

## 3. Few-Shot Examples 设计

### 格式

```xml
<examples>
<example>
<transcript>
Meeting: Weekly sync\nAttendees: Simon, Dan\n\nDan: I'll send the project
timeline to everyone after the meeting.\nSimon: We need to schedule the
demo for next Friday.
</transcript>
<output>
[
  {
    "id": "todo-1",
    "title": "Send project timeline to the team",
    "description": "Send a Teams message to the team with the updated project timeline. Dan committed to sharing this after the meeting so everyone has the latest schedule.",
    "action": "send_message"
  },
  {
    "id": "todo-2",
    "title": "Schedule demo for next Friday",
    "description": "Create a calendar event for the demo next Friday. Simon brought this up as a follow-up item.\n\nSteps:\n1. Find available times on Friday\n2. Create the meeting invite with all attendees",
    "action": "schedule_meeting"
  }
]
</output>
</example>
</examples>
```

### 质量要求

每个 example 展示：
- **title**: 以动词开头，说明要做什么（"Send..."、"Create..."、"Schedule..."）
- **description**: 包含 what（做什么）+ why（为什么，会议中的上下文）+ how（步骤，如果多步）。用祈使语气（"Send a Teams message..."），不用第一人称（~~"I'll send..."~~）
- **action**: 正确的 action enum 值

### 多样性

Examples 应覆盖：
- 不同 action 类型（send_message、schedule_meeting、create_work_item）
- 单步 vs 多步描述
- 简单 vs 复杂任务
- Edge case：transcript 中没有 actionable item → 空数组

### 常见错误（Examples 中要避免）

- ❌ Description 复述 transcript（"Dan 说了..."）— 应该写 agent 计划（"I'll send..."）
- ❌ Title 太模糊（"Follow up"）— 应该具体（"Send Dan the project timeline"）
- ❌ Description 缺少 why（只写 what）— 应该解释会议上下文
- ❌ 多步骤任务不分步（大段文字）— 应该用 `\n1. ...\n2. ...` 格式

---

## 4. Instructions 写法

简短、直接，不要过度指令：

```
✅ "Identify commitments and follow-ups. Write each TODO as an action plan
   the user can approve."

❌ "You MUST carefully analyze every sentence in the transcript and
   ALWAYS extract ALL possible action items. CRITICAL: Do not miss any..."
```

Claude 4 模型对过度指令（MUST/ALWAYS/CRITICAL）非常敏感，会导致 overtriggering。用平实自然语言。

---

## 5. Tool Schema 设计

### Description 字段是关键

Tool schema 中 parameter 的 `description` 直接影响 LLM 填什么内容：

```json
{
  "description": {
    "type": "string",
    "description": "Action plan: what will be done, why (meeting context), and steps if multi-step. Written for user approval."
  }
}
```

这比在 prompt 里写一段话更有效——LLM 在生成每个字段时会参考对应的 description。

### Enum 约束

用 `enum` 限制选项值（如 action 类型），而非在 prompt 里列举。Schema 级约束比 prompt 级更可靠。

---

## 6. 与 Agent Prompt 的关系

| 场景 | 用哪个 guideline |
|------|-----------------|
| analyze()（一次性提取 TODO） | **本文档** — structured extraction |
| execute()（ReAct 多轮执行） | agent-prompt-guidelines.md |
| Skill 文件内容 | skill-writing-standard.md |
| Tool description | tool-description-standard.md |
