# ENDINGS-CONTENT-FLOW — 结局文案批量录入流程

> 适用于100个结局的低成本配置方案。
> 文案用 AI 生成，程序审查后直接粘进 index.html，无需 CSV 或转换脚本。

---

## 分工

| 角色 | 工具 | 任务 |
|------|------|------|
| 文案 | Claude Code（VSCode） | 用 AI Prompt 批量生成结局 JS 数组，粘贴进本文件待审区 |
| 程序 | Claude Code | 审查待审区内容，通过后粘进 index.html 对应故事线 endings[] |
| PM | 当前窗口 | 审查计划，不介入实现 |

---

## 结局对象格式

每个结局是一个 JS 对象，挂在对应故事线的 `endings: []` 数组里：

```js
{
  id: 'MAIN_E11',           // 全局唯一，格式：故事线缩写_E序号
  title: '余额归零',         // 结局标题，4-8字
  icon: '💸',               // 单个 emoji
  text: '本月意外支出超过两百元。\n大部分你不知道买了什么。\n（充了个会员，平台已跑路。）',
                            // 3行正文，\n 分隔，每行不超过20字
  priority: 80,             // 组合结局90，单项80，兜底1
  condition: (s) => s.UNEXPECTED_CHARGE >= 200,
                            // 触发条件函数，签名见下方
}
```

**condition 函数签名：** `(stats, handled, replied, replyLog, escapeType) => boolean`

| 参数 | 说明 |
|------|------|
| `s` / `stats` | 数值对象，如 `s.DELIVERY_LOST`、`s.BOSS_PATIENCE` |
| `h` / `handled` | 本局处理消息总数 |
| `r` / `replied` | 本局回复总数 |
| `replyLog` | 回复记录数组 `[{senderType, reply, correct}]` |
| `escapeType` | 脱离类型字符串，如 `'silent'`、`'defiant'`，未脱离为 null |

**condition 常用写法：**

| 触发条件描述 | 代码 |
|-------------|------|
| 单项数值越界 | `(s)=>s.DELIVERY_LOST>=3` |
| 多项组合 | `(s)=>s.INFO_LEAK>=3&&s.SPAM_SUBSCRIBED>=5` |
| 回复数量 | `(s,h)=>h>=30` |
| 脱离类型 | `(s,h,r,_,et)=>et==='silent'` |
| 兜底（总是触发） | `()=>true` |

---

## 文案操作流程

1. 打开 MiniMax 网页对话，粘贴本文末尾的「AI Prompt」
2. 填入故事线ID、数量、可用 statKey（见下方补充结局计划）
3. 把输出的 JS 数组粘贴进本文件「待审区」
4. 通知程序：`ENDINGS-CONTENT-FLOW 待审区有新内容，故事线：XXX`

---

## 程序操作流程（程序看这里）

1. 读本文件「待审区」
2. 逐条检查：
   - `id` 格式正确且全局不重复
   - `text` 三行，每行不超过20字
   - `priority` 符合规则（组合90/单项80/兜底1）
   - `condition` 函数引用的 statKey 在该故事线的 `statCfg` 中存在
3. 通过后将对象数组粘进 `index.html` 对应故事线的 `endings: []`
4. 清空待审区，回复 PM：「已导入 N 条结局，故事线：XXX」

---

## 待审区

<!-- 文案把 AI 输出粘贴在此处 -->

---

## AI Prompt

复制以下内容，直接粘贴给 MiniMax：

---

你是一个手机消息模拟游戏的结局生成器。根据以下格式生成结局 JS 对象数组。

**格式（严格遵守）：**
```
{ id: '故事线_E序号', title: '标题', icon: 'emoji', text: '第一行。\n第二行。\n（第三行括号补充。）', priority: 数字, condition: (s,h,r,log,et) => 布尔表达式 }
```

**字段规则：**
- `id`：全局唯一，格式 `故事线缩写_E序号`，如 `MAIN_E11`
- `title`：4-8字，点出结局核心
- `icon`：单个 emoji，契合结局氛围
- `text`：固定3行。第1行陈述事实，第2行补充后果，第3行括号内说一句细节或反讽。每行不超过20字，用 `\n` 分隔
- `priority`：组合触发条件填90，单项触发填80，兜底（总是触发）填1
- `condition`：JS 箭头函数，参数 `(s,h,r,log,et)`，其中 `s` 是数值对象，`h` 是处理消息数，`r` 是回复数，`et` 是 escapeType

**文案风格：**
- 口吻冷静，不煽情，陈述事实
- 第3行括号句带轻微荒诞或反讽感
- 参考现有结局风格：「骑手在大堂等了十分钟。\n后来选择退单了。\n（被扣了好评。）」

**只输出 JS 数组，不要解释，不要 markdown 代码块标记。**

请为故事线 [故事线ID]，生成 [数量] 条结局。可用数值键：[列出该故事线的 statCfg 键名]。

---

## 当前已配置结局数

| 故事线 | 已有 | 目标 | 缺口 | 状态 |
|--------|------|------|------|------|
| MAIN | 10 | 40 | 30 | ⏳ MiniMax 生成中 |
| SCAM_TARGET | 3 | 20 | 17 | ⏳ MiniMax 生成中 |
| DATA_AUCTION | 5 | 20 | 15 | ⏳ MiniMax 生成中 |
| SPACETIME_RADIO | 7 | 25 | 18 | ⏳ MiniMax 生成中 |
| **合计** | **25** | **105** | **80** | — |

---

## 补充结局计划（MiniMax 参数速查）

| 轮次 | 故事线 | 数量 | 起始序号 | statKey |
|------|--------|------|---------|---------|
| 1 | MAIN | 30 | MAIN_E11 | DELIVERY_LOST, BOSS_PATIENCE, INFO_LEAK, INTIMACY, SPAM_SUBSCRIBED, UNEXPECTED_CHARGE |
| 2 | SCAM_TARGET | 17 | SCAM_E04 | SCAM_ATTENTION, DATA_SOLD；escapeType: silent/defiant/null |
| 3 | DATA_AUCTION | 15 | DA_E06 | BID_PRICE, BUYER_COUNT |
| 4 | SPACETIME_RADIO | 18 | SR_E06 | SIGNAL_STRENGTH, CONNECTION |
