---
name: "mag-ai-personal-todo-assistant"
description: "企业微信待办管理助手skill，Manages WeCom (企业微信) todos via wecom-cli: create, update, view, and delete todos from natural language. Invoke when user wants to create/modify/view/delete a todo, says '提醒我', '帮我记一下', '创建待办', '修改待办', '查看我的待办', '删除待办', or similar todo-related intent."
---

# 企业微信待办管理助手

## 功能概述

接受自然语言输入，自动识别用户意图（创建、修改、查看、删除），解析为结构化数据，然后通过 `wecom-cli` 命令行工具在企业微信中完成对应的待办操作。

**核心流程**: 自然语言 → 意图识别 → 结构化解析 → JSON 参数构造 → wecom-cli 调用 → 返回结果

**支持的操作**:

| 操作 | 触发关键词 | wecom-cli 命令 |
|------|-----------|---------------|
| 创建待办 | "提醒我"、"帮我记一下"、"创建待办" | `create_todo` |
| 修改待办 | "修改"、"改一下"、"更新"、"把...改成" | `update_todo` / `change_todo_user_status` |
| 查看待办 | "查看"、"我的待办"、"列表"、"有哪些" | `get_todo_list` / `get_todo_detail` |
| 删除待办 | "删除"、"删掉"、"去掉" | `delete_todo` |

---

## 前置条件

### 1. wecom-cli 安装检查

每次启动时，先检查 `wecom-cli` 是否已安装：

```bash
wecom-cli --help
```

**如果未安装**，执行以下两步（需要 Node.js 环境）：

```bash
# 第一步：安装 CLI 本体
npm install -g @wecom/cli

# 第二步：安装 AI Agent Skills
npx skills add WeComTeam/wecom-cli -y -g
```

安装后再次运行 `wecom-cli --help` 验证。

### 2. 认证配置检查

检查 wecom-cli 是否已初始化（配置文件路径：`~/.config/wecom/bot.enc`）：

```bash
wecom-cli todo --help
```

**如果报认证错误或未初始化**，需要用户执行：

```bash
wecom-cli init
```

用户需提供企业微信智能机器人的 **Bot ID** 和 **Secret**。获取方式：
1. 企业微信客户端 → 工作台 → 智能机器人 → 创建机器人 → 手动创建
2. 在创建页面底部点击「API 模式创建」
3. 连接方式选择「使用长连接」
4. 点击「获取 Secret」，获取 Bot ID 和 Secret

> 认证配置仅需一次，长期有效。

### 3. 用户 userid 配置

待办操作需要当前用户的 `userid`。按以下逻辑处理：

**检查本地配置文件**（路径：`<skill目录>/config.json`）：
- 如果存在且包含 `userid` 字段，直接使用
- 如果不存在，进入首次配置流程

**首次配置流程**：
1. 询问用户的企业微信姓名或别名
2. 通过命令搜索 userid：
   ```bash
   wecom-cli todo search_todo_userid '{"keyword": "用户姓名"}'
   ```
3. 从返回结果中确认正确的 userid
4. 将 userid 保存到配置文件：
   ```json
   {"userid": "确认的userid", "name": "用户姓名"}
   ```
5. 后续操作时自动读取此 userid 作为默认参与人

---

## 意图识别

收到用户输入后，首先判断操作意图：

| 意图 | 识别特征 | 示例 |
|------|---------|------|
| **创建** | 包含"提醒"、"记一下"、"创建"、"新建"、"安排" | "提醒我明天开会" |
| **修改** | 包含"修改"、"改"、"更新"、"改成"、"调整为"、"完成" | "把开会那个待办改成下午4点" |
| **查看** | 包含"查看"、"看看"、"列表"、"有哪些"、"我的待办" | "查看我的待办" |
| **删除** | 包含"删除"、"删掉"、"去掉"、"取消" | "删掉开会的待办" |

**特殊处理**：
- "把xxx标记为已完成" → 修改待办（修改状态）
- "xxx不用做了" → 删除待办（需确认）
- "改一下" / "更新" 后未指定待办 → 先进入查看待办流程让用户选择

---

## 创建待办工作流程

### 步骤 1：环境检查

1. 检查 wecom-cli 是否已安装（`wecom-cli --help`）
2. 检查是否已认证初始化
3. 检查/获取当前用户 userid

### 步骤 2：自然语言解析

从用户的自然语言输入中提取以下字段：

| 字段 | 说明 | 必填 |
|------|------|------|
| `content` | 待办内容（标题） | 是 |
| `end_time` | 截止时间，格式 `YYYY-MM-DD HH:mm:ss` | 设置提醒时必填 |
| `remind_type_list` | 提醒方式，uint32 数组 | 否 |
| `follower_id` | 参与人 userid，默认为当前用户 | 是 |

**解析规则详见下方「自然语言解析规则」章节。**

### 步骤 3：构造 JSON 参数

将解析结果构造为 wecom-cli 所需的 JSON 格式：

```json
{
  "content": "待办内容",
  "follower_list": {
    "followers": [
      {
        "follower_id": "USERID",
        "follower_status": 1
      }
    ]
  },
  "end_time": "2026-08-07 15:00:00",
  "remind_type_list": [1]
}
```

**字段说明**：
- `content`: 待办标题，简洁明了，不超过一行
- `follower_list.followers[].follower_id`: 参与人 userid，默认为当前用户
- `follower_list.followers[].follower_status`: 固定为 `1`（接受）
- `end_time`: 截止时间，格式 `YYYY-MM-DD HH:mm:ss`
- `remind_type_list`: 提醒方式数组，可传多个值

### 步骤 4：创建待办

执行 wecom-cli 命令（注意 JSON 参数用单引号包裹，Windows 下用双引号转义）：

**Linux/macOS**:
```bash
wecom-cli todo create_todo '{"content": "待办内容", "follower_list": {"followers": [{"follower_id": "USERID", "follower_status": 1}]}, "end_time": "2026-08-07 15:00:00", "remind_type_list": [1]}'
```

**Windows (PowerShell)**:
```powershell
wecom-cli todo create_todo '{\"content\": \"待办内容\", \"follower_list\": {\"followers\": [{\"follower_id\": \"USERID\", \"follower_status\": 1}]}, \"end_time\": \"2026-08-07 15:00:00\", \"remind_type_list\": [1]}'
```

> Windows 环境下，JSON 参数中的双引号需要用 `\"` 转义，外层用单引号包裹。

### 步骤 5：返回结果

成功时，返回结果包含 `todo_id`。向用户展示创建成功的待办摘要：

```
待办创建成功
- 内容：完成季度报告
- 截止时间：2026-08-07 15:00
- 提醒方式：到期时提醒
- 待办ID：todo_id_xxx
```

---

## 修改待办工作流程

### 步骤 1：解析修改意图

从用户输入中识别两部分信息：

| 信息 | 说明 | 示例 |
|------|------|------|
| **目标待办** | 要修改哪个待办 | "开会那个"、"刚才创建的"、todo_id |
| **修改内容** | 要改什么字段 | 截止时间、内容、提醒方式、状态 |

**可修改的字段**：

| 修改内容 | 自然语言示例 | 对应参数 |
|----------|-------------|---------|
| 待办内容 | "把内容改成..." | `content` |
| 截止时间 | "截止时间改成明天下午4点" | `end_time` |
| 提醒方式 | "改成提前1小时提醒" | `remind_type_list` |
| 待办状态 | "标记为已完成" | `todo_status: 0` |
| 参与人 | "增加张三为参与人" | `follower_list`（全量替换） |

### 步骤 2：定位目标待办

根据用户描述定位待办的 `todo_id`：

**方式 A：用户提供 todo_id**
直接使用。

**方式 B：通过内容关键词查找**
用户说"开会那个待办"时，调用 `get_todo_list` 查找：

```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"todo_status\": 1, \"limit\": 50}'
```

从返回列表中匹配内容包含"开会"的待办。如果匹配到多个，列出供用户选择；如果未匹配到，告知用户未找到相关待办。

**方式 C：最近创建的待办**
用户说"刚才创建的"、"上一个待办"时，调用 `get_todo_list` 获取最近的待办列表，取第一条。

**方式 D：查看后选择**
用户说"改一下"但未指定哪个待办时，先执行查看待办流程展示列表，让用户选择序号。

### 步骤 3：构造修改参数

根据要修改的字段构造 JSON。**只传需要修改的字段，未提及的字段不要传**。

**修改内容**:
```json
{"todo_id": "TODO_ID", "content": "新内容"}
```

**修改截止时间**:
```json
{"todo_id": "TODO_ID", "end_time": "2026-08-08 16:00:00"}
```

**修改提醒方式**:
```json
{"todo_id": "TODO_ID", "remind_type_list": [5]}
```

**修改待办状态（标记完成/重新激活）**:
```json
{"todo_id": "TODO_ID", "todo_status": 0}
```
> `todo_status`: 0-已完成，1-进行中

**修改多个字段**:
```json
{"todo_id": "TODO_ID", "content": "新内容", "end_time": "2026-08-08 16:00:00", "remind_type_list": [7]}
```

**修改参与人（全量替换）**:

> 注意：`follower_list` 为全量替换，不是追加。修改前需先通过 `get_todo_detail` 查询现有参与人，合并后提交。

```json
{"todo_id": "TODO_ID", "follower_list": {"followers": [{"follower_id": "USERID_1", "follower_status": 1}, {"follower_id": "USERID_2", "follower_status": 1}]}}
```

### 步骤 4：执行修改

```powershell
wecom-cli todo update_todo '{\"todo_id\": \"TODO_ID\", \"content\": \"新内容\", \"end_time\": \"2026-08-08 16:00:00\"}'
```

### 步骤 5：返回结果

```
待办修改成功
- 待办ID：todo_id_xxx
- 修改内容：截止时间 → 2026-08-08 16:00
```

### 特殊操作：修改参与人状态

当用户说"我完成了那个待办"（仅改自己的状态，不改待办整体状态）时，使用 `change_todo_user_status`：

```powershell
wecom-cli todo change_todo_user_status '{\"todo_id\": \"TODO_ID\", \"follower_id\": \"USERID\", \"user_status\": 2}'
```

> `user_status`: 0-拒绝，1-接受，2-已完成

---

## 查看待办工作流程

### 步骤 1：解析查看意图

| 用户意图 | 自然语言示例 | 处理方式 |
|----------|-------------|---------|
| 查看全部待办 | "查看我的待办"、"我的待办列表" | `get_todo_list`，不传 todo_status |
| 查看进行中待办 | "我有哪些未完成的待办"、"进行中的待办" | `get_todo_list`，`todo_status: 1` |
| 查看已完成待办 | "已完成的待办" | `get_todo_list`，`todo_status: 0` |
| 按时间筛选 | "今天的待办"、"这周的待办" | `get_todo_list`，传入时间范围 |
| 查看某个待办详情 | "xxx待办的详情" | 先 `get_todo_list` 找到 todo_id，再 `get_todo_detail` |

### 步骤 2：获取待办列表

**基础查询（全部待办）**:
```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"limit\": 50}'
```

**筛选进行中**:
```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"todo_status\": 1, \"limit\": 50}'
```

**按时间范围筛选**:

| 自然语言 | create_begin_time | create_end_time |
|----------|------------------|----------------|
| "今天的待办" | 今天 00:00:00 | 今天 23:59:59 |
| "这周的待办" | 本周一 00:00:00 | 本周日 23:59:59 |
| "最近一周" | 7天前 00:00:00 | 今天 23:59:59 |
| "最近一个月" | 30天前 00:00:00 | 今天 23:59:59 |

> 注意：筛选时间必须落在当天前后 30 天以内。

```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"create_begin_time\": \"2026-08-06 00:00:00\", \"create_end_time\": \"2026-08-06 23:59:59\", \"limit\": 50}'
```

### 步骤 3：获取待办详情

如果用户想查看某个待办的完整信息，先从列表中获取 `todo_id`，再调用详情接口：

```powershell
wecom-cli todo get_todo_detail '{\"todo_id_list\": [\"TODO_ID_1\"]}'
```

### 步骤 4：格式化展示结果

**列表展示格式**:

```
你的待办列表（共 3 条）

1. [进行中] 部门例会
   截止时间：2026-08-07 15:00
   提醒：到期时提醒
   待办ID：todo_id_1

2. [进行中] 完成季度报告
   截止时间：2026-08-14 09:00
   提醒：提前1天提醒
   待办ID：todo_id_2

3. [已完成] 提交月度总结
   截止时间：2026-08-05 18:00
   待办ID：todo_id_3
```

**详情展示格式**:

```
待办详情
- 内容：完成季度报告
- 状态：进行中
- 创建人：zhangsan
- 创建时间：2026-08-06 10:30
- 截止时间：2026-08-14 09:00
- 提醒：提前1天提醒
- 参与人：zhangsan（进行中）、lisi（进行中）
- 待办ID：todo_id_xxx
```

> 注意：展示时用内容标题代替 userid，不直接暴露原始 userid。

---

## 删除待办工作流程

### 步骤 1：解析删除意图

从用户输入中识别要删除的目标待办。

| 自然语言 | 目标待办定位方式 |
|----------|----------------|
| "删除xxx待办" | 通过内容关键词查找 |
| "删掉刚才创建的待办" | 获取最近一条 |
| "删除todo_id是xxx的待办" | 直接使用 |
| "xxx不用做了" | 通过内容关键词查找 |

### 步骤 2：定位目标待办

与修改待办的定位流程一致（参见「修改待办工作流程 - 步骤 2」），通过 `get_todo_list` 查找匹配的待办。

### 步骤 3：确认删除

**删除操作不可逆，必须向用户确认**：

```
确认要删除以下待办吗？
- 内容：部门例会
- 截止时间：2026-08-07 15:00
- 待办ID：todo_id_xxx

（删除后无法恢复，请回复"确认"继续）
```

如果用户回复"确认"、"是"、"删除"等肯定词，继续执行；否则取消操作。

### 步骤 4：执行删除

```powershell
wecom-cli todo delete_todo '{\"todo_id\": \"TODO_ID\"}'
```

### 步骤 5：返回结果

```
待办已删除
- 内容：部门例会
- 待办ID：todo_id_xxx
```

---

## 自然语言解析规则

> 以下规则适用于创建和修改待办场景。

### 1. 待办内容提取

从自然语言中识别待办的核心事项：

| 自然语言 | 提取的 content |
|----------|---------------|
| "提醒我明天下午3点开会" | "开会" |
| "下周五之前完成季度报告" | "完成季度报告" |
| "帮我记一下给张三发邮件" | "给张三发邮件" |
| "明天上午10点部门例会，记得准备材料" | "部门例会，准备材料" |

**规则**：
- 去掉时间修饰词（"明天"、"下午3点"等），保留核心事项
- 如果有多个事项，合并为一条待办内容
- 内容应简洁，不超过一行

### 2. 时间解析规则

将中文时间表达转换为 `YYYY-MM-DD HH:mm:ss` 格式。**必须基于当前系统时间计算**。

#### 日期解析

| 自然语言 | 含义 | 示例（当前2026-08-06） |
|----------|------|----------------------|
| "今天" | 当前日期 | 2026-08-06 |
| "明天" | 当前日期+1 | 2026-08-07 |
| "后天" | 当前日期+2 | 2026-08-08 |
| "大后天" | 当前日期+3 | 2026-08-09 |
| "下周一" | 下一个周一 | 2026-08-10 |
| "下周五" | 下一个周五 | 2026-08-14 |
| "这周五" | 本周五 | 2026-08-07 |
| "X月X日" / "X月X号" | 指定日期 | 如"8月15日" → 2026-08-15 |
| "X号" | 本月X日 | 如"15号" → 2026-08-15 |
| "下个月X号" | 下月X日 | 如"下个月5号" → 2026-09-05 |
| "三天后" / "3天后" | 当前日期+N | 2026-08-09 |
| "下周五之前" | 下周五当天 | 2026-08-14 |

#### 时间解析

| 自然语言 | 含义 | 转换结果 |
|----------|------|---------|
| "上午9点" / "早上9点" | 09:00:00 | 09:00:00 |
| "下午3点" | 15:00:00 | 15:00:00 |
| "下午3点半" | 15:30:00 | 15:30:00 |
| "晚上8点" | 20:00:00 | 20:00:00 |
| "下午3点15分" | 15:15:00 | 15:15:00 |
| "下班前" | 18:00:00 | 18:00:00 |
| "中午" | 12:00:00 | 12:00:00 |
| 未指定时间 | 默认 09:00:00 | 09:00:00 |

#### 相对时间解析

| 自然语言 | 含义 |
|----------|------|
| "2小时后" | 当前时间 + 2小时 |
| "30分钟后" | 当前时间 + 30分钟 |
| "1小时后提醒我" | 当前时间 + 1小时 |

#### 组合示例

| 自然语言 | 当前时间 | 解析结果 |
|----------|---------|---------|
| "明天下午3点开会" | 2026-08-06 10:30 | 2026-08-07 15:00:00 |
| "下周五上午10点" | 2026-08-06 | 2026-08-14 10:00:00 |
| "8月15号下午2点半" | 2026-08-06 | 2026-08-15 14:30:00 |
| "3小时后" | 2026-08-06 10:30 | 2026-08-06 13:30:00 |

### 3. 提醒方式解析

| 自然语言 | remind_type_list |
|----------|-----------------|
| "到期提醒" / "到时候提醒我" | [1] |
| "提前15分钟提醒" | [3] |
| "提前1小时提醒" | [5] |
| "提前2小时提醒" | [6] |
| "提前1天提醒" / "前一天提醒" | [7] |
| "提前2天提醒" | [8] |
| "提前1周提醒" | [9] |
| "提前15分钟和提前1天都提醒" | [3, 7] |
| "提前半小时提醒" | [3]（归入15分钟档） |
| 未提及提醒方式 | 不传 remind_type_list 或传 [0] |

**remind_type_list 完整取值表**：

| 值 | 含义 |
|----|------|
| 0 | 不提醒 |
| 1 | 到期时 |
| 3 | 提前 15 分钟 |
| 5 | 提前 1 小时 |
| 6 | 提前 2 小时 |
| 7 | 提前 1 天 |
| 8 | 提前 2 天 |
| 9 | 提前 1 周 |

> 注意：当 `remind_type_list` 包含非 0 值时，`end_time` 为必填。

### 4. 参与人解析

| 自然语言 | 处理方式 |
|----------|---------|
| "提醒我..." | 仅当前用户为参与人 |
| "帮我创建待办..." | 仅当前用户为参与人 |
| "提醒我和张三..." | 当前用户 + 搜索张三的 userid |
| "给张三创建待办" | 搜索张三的 userid 作为参与人 |

**搜索他人 userid**：
```bash
wecom-cli todo search_todo_userid '{"keyword": "张三"}'
```

从返回结果中确认正确的 userid 后，添加到 `follower_list.followers` 数组中。

---

## JSON 参数构造模板

### 创建待办

#### 最简模板（仅内容，无截止时间）

```json
{
  "content": "待办内容",
  "follower_list": {
    "followers": [
      {"follower_id": "USERID", "follower_status": 1}
    ]
  }
}
```

#### 标准模板（内容 + 截止时间 + 提醒）

```json
{
  "content": "待办内容",
  "follower_list": {
    "followers": [
      {"follower_id": "USERID", "follower_status": 1}
    ]
  },
  "end_time": "2026-08-07 15:00:00",
  "remind_type_list": [1]
}
```

#### 多参与人模板

```json
{
  "content": "待办内容",
  "follower_list": {
    "followers": [
      {"follower_id": "USERID_1", "follower_status": 1},
      {"follower_id": "USERID_2", "follower_status": 1}
    ]
  },
  "end_time": "2026-08-07 15:00:00",
  "remind_type_list": [3, 7]
}
```

### 修改待办

#### 修改内容
```json
{"todo_id": "TODO_ID", "content": "新内容"}
```

#### 修改截止时间
```json
{"todo_id": "TODO_ID", "end_time": "2026-08-08 16:00:00"}
```

#### 修改提醒方式
```json
{"todo_id": "TODO_ID", "remind_type_list": [5]}
```

#### 标记完成/重新激活
```json
{"todo_id": "TODO_ID", "todo_status": 0}
```

#### 修改多个字段
```json
{"todo_id": "TODO_ID", "content": "新内容", "end_time": "2026-08-08 16:00:00", "remind_type_list": [7]}
```

#### 修改参与人（全量替换）
```json
{"todo_id": "TODO_ID", "follower_list": {"followers": [{"follower_id": "USERID_1", "follower_status": 1}, {"follower_id": "USERID_2", "follower_status": 1}]}}
```

### 修改参与人状态
```json
{"todo_id": "TODO_ID", "follower_id": "USERID", "user_status": 2}
```

### 查看待办

#### 待办列表
```json
{"follower_id": "USERID", "limit": 50}
```

#### 筛选进行中
```json
{"follower_id": "USERID", "todo_status": 1, "limit": 50}
```

#### 按时间筛选
```json
{"follower_id": "USERID", "create_begin_time": "2026-08-06 00:00:00", "create_end_time": "2026-08-06 23:59:59", "limit": 50}
```

#### 待办详情
```json
{"todo_id_list": ["TODO_ID_1"]}
```

### 删除待办
```json
{"todo_id": "TODO_ID"}
```

---

## 错误处理

### 常见错误及解决方案

| 错误信息 | 原因 | 解决方案 |
|----------|------|---------|
| `command not found: wecom-cli` | 未安装 wecom-cli | 执行 `npm install -g @wecom/cli` |
| 认证失败 / 无 Bot 配置 | 未执行 init | 执行 `wecom-cli init` 配置 Bot ID 和 Secret |
| `follower not found` | userid 错误 | 重新搜索 userid: `wecom-cli todo search_todo_userid` |
| `end_time is required` | 设置了提醒但未提供截止时间 | 补充 end_time 字段 |
| `no permission` | 机器人可见范围未包含用户 | 在企业微信管理后台调整机器人可见范围 |
| `todo not found` | todo_id 错误或待办已被删除 | 重新查询待办列表获取正确的 todo_id |
| `cannot modify deleted todo` | 待办已被删除 | 告知用户该待办已删除，无法修改 |
| `only allow modify own todo` | 尝试修改非本应用创建的待办 | 仅支持修改通过本 Skill 创建的待办 |
| `time range exceeds 30 days` | 查询时间范围超过30天 | 缩小查询时间范围至30天以内 |
| 网络超时 | 网络问题或企业微信服务波动 | 重试一次 |

### 错误处理流程

1. 捕获命令执行的 stderr 输出
2. 匹配错误信息表，定位原因
3. 向用户说明错误原因和解决方案
4. 如果是可恢复错误（如网络超时），自动重试一次
5. 如果是配置问题，引导用户完成配置后重试

---

## 完整示例

### 创建示例

#### 示例 1：简单待办

**用户输入**: "提醒我明天下午3点开会"

**解析结果**:
- content: "开会"
- end_time: "2026-08-07 15:00:00"（假设当前为2026-08-06）
- remind_type_list: [1]（到期时提醒）
- follower_id: 当前用户 userid

**执行命令**:
```powershell
wecom-cli todo create_todo '{\"content\": \"开会\", \"follower_list\": {\"followers\": [{\"follower_id\": \"USERID\", \"follower_status\": 1}]}, \"end_time\": \"2026-08-07 15:00:00\", \"remind_type_list\": [1]}'
```

#### 示例 2：带提前提醒

**用户输入**: "下周五之前完成季度报告，提前1天提醒我"

**解析结果**:
- content: "完成季度报告"
- end_time: "2026-08-14 09:00:00"（下周五，未指定时间默认09:00）
- remind_type_list: [7]（提前1天）
- follower_id: 当前用户 userid

#### 示例 3：无截止时间的简单待办

**用户输入**: "帮我记一下给张三发邮件"

**解析结果**:
- content: "给张三发邮件"
- 无 end_time
- 无 remind_type_list
- follower_id: 当前用户 userid

#### 示例 4：多参与人待办

**用户输入**: "后天上午10点部门评审会，提醒我和李四，提前15分钟"

**解析结果**:
- content: "部门评审会"
- end_time: "2026-08-08 10:00:00"
- remind_type_list: [3]（提前15分钟）
- followers: [当前用户 userid, 李四的 userid（需搜索）]

#### 示例 5：相对时间

**用户输入**: "2小时后提醒我提交报告"

**解析结果**（假设当前时间 2026-08-06 10:30:00）:
- content: "提交报告"
- end_time: "2026-08-06 12:30:00"
- remind_type_list: [1]（到期时提醒）
- follower_id: 当前用户 userid

### 修改示例

#### 示例 6：修改截止时间

**用户输入**: "把开会那个待办的截止时间改成明天下午4点"

**处理流程**:
1. 意图识别：修改待办
2. 定位待办：调用 `get_todo_list` 查找内容包含"开会"的待办
3. 解析新时间："明天下午4点" → "2026-08-07 16:00:00"
4. 执行修改：

```powershell
wecom-cli todo update_todo '{\"todo_id\": \"查到的TODO_ID\", \"end_time\": \"2026-08-07 16:00:00\"}'
```

#### 示例 7：标记待办完成

**用户输入**: "季度报告那个待办我已经完成了"

**处理流程**:
1. 意图识别：修改待办状态
2. 定位待办：查找内容包含"季度报告"的待办
3. 执行修改：

```powershell
wecom-cli todo update_todo '{\"todo_id\": \"查到的TODO_ID\", \"todo_status\": 0}'
```

#### 示例 8：修改内容和提醒

**用户输入**: "把部门例会的内容改成部门周会，提醒改成提前1小时"

**处理流程**:
1. 意图识别：修改待办
2. 定位待办：查找内容包含"部门例会"的待办
3. 执行修改：

```powershell
wecom-cli todo update_todo '{\"todo_id\": \"查到的TODO_ID\", \"content\": \"部门周会\", \"remind_type_list\": [5]}'
```

#### 示例 9：修改参与人状态

**用户输入**: "那个评审会我不参与了"

**处理流程**:
1. 意图识别：修改参与人状态
2. 定位待办：查找内容包含"评审会"的待办
3. 执行修改：

```powershell
wecom-cli todo change_todo_user_status '{\"todo_id\": \"查到的TODO_ID\", \"follower_id\": \"当前用户USERID\", \"user_status\": 0}'
```

### 查看示例

#### 示例 10：查看全部待办

**用户输入**: "查看我的待办"

**执行命令**:
```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"limit\": 50}'
```

**展示结果**:
```
你的待办列表（共 2 条）

1. [进行中] 部门例会
   截止时间：2026-08-07 15:00
   提醒：到期时提醒

2. [进行中] 完成季度报告
   截止时间：2026-08-14 09:00
   提醒：提前1天提醒
```

#### 示例 11：查看进行中待办

**用户输入**: "我有哪些未完成的待办"

**执行命令**:
```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"todo_status\": 1, \"limit\": 50}'
```

#### 示例 12：按时间筛选

**用户输入**: "今天的待办有哪些"

**执行命令**:
```powershell
wecom-cli todo get_todo_list '{\"follower_id\": \"USERID\", \"create_begin_time\": \"2026-08-06 00:00:00\", \"create_end_time\": \"2026-08-06 23:59:59\", \"limit\": 50}'
```

#### 示例 13：查看待办详情

**用户输入**: "季度报告那个待办的详细信息"

**处理流程**:
1. 先通过 `get_todo_list` 查找内容包含"季度报告"的待办，获取 todo_id
2. 再调用详情接口：

```powershell
wecom-cli todo get_todo_detail '{\"todo_id_list\": [\"查到的TODO_ID\"]}'
```

### 删除示例

#### 示例 14：删除指定待办

**用户输入**: "删掉开会的那个待办"

**处理流程**:
1. 意图识别：删除待办
2. 定位待办：查找内容包含"开会"的待办
3. 确认删除：

```
确认要删除以下待办吗？
- 内容：开会
- 截止时间：2026-08-07 15:00

（删除后无法恢复，请回复"确认"继续）
```

4. 用户确认后执行：

```powershell
wecom-cli todo delete_todo '{\"todo_id\": \"查到的TODO_ID\"}'
```

#### 示例 15：删除最近创建的待办

**用户输入**: "删掉我刚才创建的待办"

**处理流程**:
1. 意图识别：删除待办
2. 定位待办：调用 `get_todo_list` 获取最近的待办（取第一条）
3. 确认删除并执行

---

## 注意事项

1. **userid 安全**：userid 是敏感标识，仅用于接口传参，禁止向用户展示原始 userid
2. **时间基准**：所有时间解析基于当前系统时间，务必获取准确的当前时间
3. **企业规模限制**：wecom-cli 目前优先支持 10 人以下企业
4. **待办内容简洁**：content 应简洁明了，过长内容会被截断显示
5. **JSON 转义**：Windows PowerShell 环境下，JSON 参数需正确转义双引号
6. **userid 获取**：禁止自行猜测或构造 userid，必须通过 `search_todo_userid` 命令获取
7. **机器人可见范围**：确保企业微信智能机器人的可见范围包含待办参与人
8. **todo_id 来源**：todo_id 只能来自 `create_todo` 返回结果或 `get_todo_list` 查询结果，禁止自行推测或构造
9. **参与人全量替换**：`update_todo` 中的 `follower_list` 为全量替换（非追加），新增参与人需先查现有参与人再合并提交
10. **删除不可逆**：`delete_todo` 删除后无法恢复，必须向用户确认后执行
11. **查询时间限制**：`get_todo_list` 筛选时间必须落在当天前后 30 天以内
12. **仅限本应用待办**：查询和修改仅支持通过本 Skill（本机器人）创建的待办