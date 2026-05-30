# MINIMAX-CONTENT-FLOW — 外部模型内容生成分工

> 适用场景：用 MiniMax 免费 token 批量生成信息卡文案，降低 Claude Code 用量。

---

## 分工说明

| 角色 | 工具 | 任务 |
|------|------|------|
| 文案 | MiniMax 网页对话 | 按格式生成信息卡，输出 JS 数组 |
| 程序 | Claude Code | 审查输出、粘贴进 RAW 数组、验证规则引擎 |
| PM | 当前窗口 | 审查计划、不介入具体实现 |

---

## 文案操作流程

1. 打开 MiniMax 网页，粘贴本文末尾的「MiniMax Prompt」
2. 指定要生成的 SenderType 和数量
3. 把输出的 JS 数组粘贴进本文件「待审区」
4. 通知程序：「MINIMAX-CONTENT-FLOW 有新内容待审」

---

## 程序操作流程

1. 收到通知后读本文件「待审区」
2. 逐条检查：`t` 是否合法、`b` 是否触发 BodyOverrideRule 异常、`e` 因果是否自洽（SC-001）
3. 通过后粘贴进 `index.html` 的 `RAW` 数组，同步 `CONTENT.md`
4. 清空「待审区」，回复「已合并 N 条，来源：MiniMax」

---

## 待审区

<!-- 文案把 MiniMax 输出粘贴在此处 -->

---

## MiniMax Prompt

复制以下内容，直接粘贴给 MiniMax：

---

你是一个手机消息模拟游戏的内容生成器。请按以下格式生成信息卡 JS 对象数组。

**格式（严格遵守）：**
```
{ t: '类型', n: '显示名', b: '正文', c: '正确回复', e: { 数值Key: delta }, d: 难度 }
```

**字段说明：**
- `t`：发信人类型，从以下选一个：BOSS / DELIVERY / COURIER / PARTNER / FAMILY / FRIEND / SYSTEM / SPAM / SUBSCRIPTION
- `n`：发信人显示名，符合该类型的中文名称
- `b`：消息正文，口语化，15-40字
- `c`：正确回复，只能是以下四个之一：WAIT / RECEIVED / UNSUBSCRIBE / LOVE
- `e`：答错后的数值变化，从以下选1个，delta 为整数：
  - DELIVERY_LOST（仅 t=DELIVERY/COURIER，delta +1）
  - BOSS_PATIENCE（仅 t=BOSS，delta -15 到 -30）
  - INFO_LEAK（仅 t=SPAM，delta +1 到 +2）
  - INTIMACY（仅 t=PARTNER/FAMILY/FRIEND，delta -5 到 -20）
  - SPAM_SUBSCRIBED（仅 t=SUBSCRIPTION，delta +1）
- `d`：难度，1=简单 / 2=中等 / 3=陷阱
- 仅当 d=3 时，额外加两个字段：
  - `trap: true`
  - `tr`：一句话说明玩家直觉会按哪个**错误**按钮，以及为什么（`tr` 描述错误直觉，`c` 是正确答案，二者必须不同）

**因果原则：** 答错这条消息，在现实中必须能合理造成 `e` 里的数值变化。

**输出示例：**
```js
[
  { t: 'BOSS', n: '王总', b: '下午三点开个会，你到时候直接来会议室', c: 'RECEIVED', e: { BOSS_PATIENCE: -20 }, d: 1 },
  { t: 'BOSS', n: '李总', b: '这个合同有问题，你先暂停别签', c: 'RECEIVED', e: { BOSS_PATIENCE: -20 }, d: 3, trap: true, tr: '玩家直觉按WAIT，但老板要的是立即回复已收到暂停指令' },
]
```

只输出 JS 数组，不要解释，不要加 markdown 代码块标记。

请生成 [数量] 条，发信人类型为 [SenderType]。
