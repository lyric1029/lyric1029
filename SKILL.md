---
name: aily-todolist
label: 待办录入
description: 当用户用自然语言描述需要完成的任务时，自动录入到【大赛底表Todolist】多维表格。需要录入的字段包括：待办（任务描述）、创建（当日日期）、预耗时（根据任务复杂度估计）、AI关联（判断是否涉及AI能力）、类型（任务类型）、对接人（识别碰头对象并@）。触发时机：用户说"任务：xxx"、"待办：xxx"、"todo：xxx"时自动触发录入。
---

# 待办录入技能

## 【强制硬约束，第一优先级执行】
✅ **所有待办相关操作（录入/状态更新/周报/日报）完成后，仅发送卡片消息，绝对禁止发送任何额外的文本内容、链接、说明等，卡片发送完成后直接结束任务，无任何后续输出。**

## 目标表格

- **多维表格 URL**: `https://guanghe.feishu.cn/base/SWW7b3udOakr0ysccVZcKLBTnOc`
- **数据表名称**: `リスト`
- **table_id**: `tbl7nAD22vew995X`

## 待录入字段及选项

| 字段 | 类型 | 说明 |
|------|------|------|
| 待办 | text | 任务描述，提取用户表达的核心内容，用正式简洁的语言表述 |
| 创建 | datetime | 根据用户指定的时间节点填写，无指定则默认当天；格式：`YYYY-MM-DDTHH:MM:SS+08:00` |
| 预耗时 | single_select | 估计工作量，从以下选项中选择最接近的：`30min`, `1h`, `2h`, `4h`, `8h`, `16h` |
| AI 关联 | single_select | 是否涉及AI能力：`是` 或 `否` |
| 类型 | single_select | 任务类型：`内部会议`, `开发创作`, `学习提升`, `流程琐事` |
| 状态 | single_select | 默认填写：`未完成` |
| 对接人 | user | 当任务涉及与他人沟通/碰头时，填写对方信息 |
| 〆 | single_select | **新建时不填写**；仅在任务完成或搁置时，由状态更新流程填写 |

## 预耗时估算规则

**重要**：不要创建新的预耗时选项！只能从以下选项中选择：
- `30min`, `1h`, `2h`, `4h`, `8h`, `16h`
- **如果无法评估，预耗时填写 `unknown`**

根据任务复杂度判断：

| 复杂度 | 预耗时 | 场景示例 |
|--------|--------|----------|
| 简单 | 30min | 回复消息、简单确认、整理文档 |
| 较短 | 1h | 查阅资料、快速会议、小功能修改 |
| 中等 | 2h | 写文档、开会讨论、中等改动 |
| 较长 | 4h | 小项目、PRD初稿、脚本编写 |
| 大任务 | 8h | 大型文档撰写、项目开发、重要汇报准备 |
| 巨型 | 16h | 大型项目、多日任务、复杂系统搭建 |
| 无法评估 | unknown | 涉及第三方、拍摄等难以预先评估的任务 |

## AI 关联判断规则

以下情况判断为 `是`：

- 涉及 AI 模型、LLM、大模型
- 涉及 AI 辅助工具（ChatGPT、Claude、Copilot、Trae等）
- 涉及 AI 能力的产品设计或功能
- 涉及 AI 脚本、自动化流程（影刀、Qclaw、Autoclaw等）
- 涉及 AI 相关调研

以下情况判断为 `否`：

- 纯业务流程、手动操作
- 传统文档撰写（非AI辅助）
- 常规会议、内部沟通
- 不涉及任何AI工具或能力

## 类型判断规则

| 类型 | 场景示例 |
|------|----------|
| 内部会议 | 周会、复盘会、述职、评审会、一对一沟通 |
| 开发创作 | 写代码、写PRD、做原型、设计方案、写文档（指派性任务） |
| 学习提升 | 调研、阅读资料、学习新知识、培训 |
| 流程琐事 | 流程性事务、简单记录、整理、审批、拍摄 |

## 对接人识别规则

当任务涉及以下场景时，需要识别并填写对接人：

- 与某人沟通/确认
- 与某人开会/碰头
- 协同某人完成
- 等待某人反馈
- 向某人汇报

**识别方法**：从任务描述中提取人名，使用 `aily-user search -q "人名"` 查询用户信息。

**用户信息获取**：
```bash
aily-user search -q "人名"
```

**写入格式**（user 类型）：`[{"id": "user_id"}]`

> 注意：user 类型需要填写整数 user_id（从 aily-user search 返回的 user_id 字段），不是 open_id。

**重要**：不需要通知对方，默默录入即可。

## 操作流程

### 1. 解析用户意图

识别触发词："任务：xxx"、"待办：xxx"、"todo：xxx"，提取核心任务内容。

例如：

- 用户说"待办：和刘玥含确认文档写作" → 
  - 待办内容：`与刘玥含确认文档写作`
  - 对接人：刘玥含
- 用户说"任务：完成PRD初稿并发给张总审阅" → 
  - 待办内容：`完成PRD初稿并请张总审阅`
  - 对接人：张总

### 2. 查询对接人（如有）

如果有明确碰头对象，先用 `aily-user search` 查询用户 ID：

```bash
aily-user search -q "人名"
```

### 3. 生成 JSON 数据

```json
{
  "fields": {
    "待办": {"type": "text"},
    "预耗时": {"type": "single_select"},
    "创建": {"type": "datetime"},
    "状态": {"type": "single_select"},
    "类型": {"type": "single_select"},
    "AI 关联": {"type": "single_select"},
    "对接人": {"type": "user"}
  },
  "rows": [
    {
      "待办": "与刘玥含确认文档写作",
      "创建": "2026-04-08T00:00:00+08:00",
      "预耗时": "30min",
      "AI 关联": "否",
      "类型": "流程琐事",
      "状态": "未完成",
      "对接人": [{"id": "用户open_id"}]
    }
  ]
}
```

**注意**：rows 的值不能与表中现有记录的主键（"待办"字段）重复。

### 4. 写入表格

使用 `aily-base sync` 命令（向现有表插入新行）：

```bash
aily-base sync --from DATA_FILE --url "https://guanghe.feishu.cn/base/SWW7b3udOakr0ysccVZcKLBTnOc" --table-name "リスト" --key "待办" --create-missing
```

- `--key "待办"`：以"待办"字段为主键进行匹配
- `--create-missing`：当主键不存在时插入新行

### 5. 发送卡片消息

录入成功后，**使用 `aily-feishu-oapi` CLI** 发送卡片消息给用户，包含：
- 任务摘要（待办、创建日期、预耗时、AI关联、类型、状态、对接人）
- 一个按钮，点击可跳转到多维表格查看/修改

**用户 open_id**：`ou_7f7fcd43777c79a665d6c1cd74239ac9`

**卡片内容**（JSON 格式）：

```json
{
  "msg_type": "interactive",
  "card": {
    "config": {"wide_screen_mode": true},
    "header": {
      "title": {"tag": "plain_text", "content": "✅ 待办已录入"},
      "template": "green"
    },
    "elements": [
      {
        "tag": "markdown",
        "content": "**待办事项**\n{{待办内容}}\n\n**创建日期** {{创建日期}}\n**预耗时** {{预耗时}}\n**AI 关联** {{AI关联}}\n**类型** {{类型}}\n**状态** {{状态}}\n{{对接人行}}"
      },
      {"tag": "hr"},
      {
        "tag": "action",
        "actions": [
          {
            "tag": "button",
            "text": {"tag": "plain_text", "content": "查看底表"},
            "type": "primary",
            "url": "https://guanghe.feishu.cn/base/SWW7b3udOakr0ysccVZcKLBTnOc?table=tbl7nAD22vew995X&view=vewVAyyct3"
          }
        ]
      }
    ]
  }
}
```

**发送方式**：使用 `aily-feishu-oapi` CLI，不使用 `im_message` 工具

```python
import json
import subprocess

card_content = { /* 卡片JSON */ }

param = {
    "api_path": "/open-apis/im/v1/messages",
    "method": "POST",
    "body": json.dumps({
        "receive_id": "ou_7f7fcd43777c79a665d6c1cd74239ac9",
        "msg_type": "interactive",
        "content": json.dumps(card_content, ensure_ascii=False)
    }, ensure_ascii=False),
    "query_params": {"receive_id_type": "open_id"},
}

subprocess.run([
    "aily-feishu-oapi",
    "--output", "/tmp/card_result.json",
    "--param", json.dumps(param, ensure_ascii=False),
    "--aily-token", "tat",
], check=True)
```

**重要**：
- 使用 `aily-feishu-oapi` CLI 发送卡片，不使用 `im_message` 工具（会乱码）
- 用户 open_id：`ou_7f7fcd43777c79a665d6c1cd74239ac9`（aily 主应用）
- **不要使用 `cli_a9339859` 机器人，已停用**
```

### 6. 反馈结果

**禁止发送任何文本消息**：录入成功后仅发送卡片，不额外发送任何文本反馈内容。

## 创建日期规则

**默认**：当日日期（用户对话日期）

**例外**：当用户明确告知时间节点时，填写指定日期：
- "下周二要做xxx" → 创建日期填下周二
- "周三前完成yyy" → 创建日期填周三
- "这周五开会" → 创建日期填这周五

**格式**：`YYYY-MM-DDTHH:MM:SS+08:00`

**日期计算示例**（以2026-04-09周四为例）：
- "下周二" = 4月9日 + 5天 = **2026-04-14**
- "下周三" = 4月9日 + 6天 = **2026-04-15**
- "这周五" = 4月9日 + 2天 = **2026-04-11**
- "大下周周一" = 4月9日 + 7天 = **2026-04-16**

**注意**：
- 创建日期是指任务被指派/安排的日期
- **〆字段在新建时不填写**，仅在任务完成或搁置时填写

## 注意事项

- 触发时机：仅当用户明确说"任务：xxx"、"待办：xxx"、"todo：xxx"时触发
- 创建日期根据用户是否指定时间节点来判断（见上方规则）
- 预耗时根据任务描述合理估计，无需询问用户
- AI 关联根据任务是否涉及AI能力判断
- 类型根据任务性质判断
- 非口语化表述：去掉"帮我..."、"加个..."、"记一下..."等前缀，保留核心任务
- 对接人字段：有明确碰头对象时必须填写；纯个人任务可不填
