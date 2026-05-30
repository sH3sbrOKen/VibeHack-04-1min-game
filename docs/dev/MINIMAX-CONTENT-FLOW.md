# MINIMAX-CONTENT-FLOW — 外部模型内容生成分工

> 适用场景：用 MiniMax 免费 token 批量生成信息卡文案和音频，降低 Claude Code 用量。
>
> 结局文案录入见 [ENDINGS-CONTENT-FLOW.md](./ENDINGS-CONTENT-FLOW.md)（CSV 批量导入流程）。

---

## 音频需求（BGM + SFX）

> 当前已有一版音频（MP3），路演前可能需要重新生成。程序不介入音频内容，由文案/制作人负责。

### 文件清单与要求

| 文件名 | 类型 | 循环 | 说明 |
|--------|------|------|------|
| `bgm-main.mp3` | BGM | 是 | 现代都市日常，轻微焦虑，手机消息不断涌入的紧迫感，电子+钢琴，偏暗，BPM约100 |
| `bgm-start.mp3` | BGM | 是 | 开场等待界面，静谧略带悬念，比bgm-main更克制，深夜盯着手机屏幕的感觉 |
| `bgm-soviet.mp3` | BGM | 是 | 1986年苏联地下电台，模拟信号噪声+神秘弦乐，冷战感，绿色荧光屏既视感 |
| `bgm-spacetime.mp3` | BGM | 是 | 跨越时空的无线电频率，失真信号+空灵合成器，疏离感，像在宇宙中等待回应 |
| `bgm-android.mp3` | BGM | 是 | 机械入侵感，低频威胁性电子音，红色警报氛围，节奏不规律带抖动，令人不安 |
| `sfx-msg.mp3` | SFX | 否 | 手机收到新消息提示音，清脆简短，类微信消息音但更有设计感，1秒内 |
| `sfx-correct.mp3` | SFX | 否 | 正确回复反馈，轻快积极，像完成一个小任务，1秒内 |
| `sfx-wrong.mp3` | SFX | 否 | 错误回复反馈，低沉短促，轻微失落感，不刺耳，1秒内 |
| `sfx-intrusion-radio.mp3` | SFX | 否 | 苏联电台突然闯入，模拟信号接通声，嗡嗡电流+短暂静电噪声，约1.5秒 |
| `sfx-intrusion-android.mp3` | SFX | 否 | 伪人AI突然入侵，机械警报感，低频脉冲+金属质感，约1.5秒，令人警觉 |

### 音量参考（程序侧常量，供音频制作参考响度）

- BGM 主线/超时空：`BGM_VOL = 0.4`（中低音量）
- BGM 乱入（苏联/伪人）：`BGM_INTRUSION_VOL = 0.5`（略突出）
- SFX：`SFX_VOL = 0.6`（清晰可闻）

### 文件交付

生成完成后直接放入 `music/` 文件夹，替换同名文件，通知程序确认路径无误后 commit。

### MiniMax 提示词（重新生成全套）

```
请生成以下10个音频文件，全部输出为 MP3 格式，文件命名如括号所示：

BGM（循环背景音乐，各30秒以上，结尾可无缝衔接循环）：
1. bgm-main.mp3 — 现代都市日常氛围，轻微焦虑感，手机消息不断涌入的紧迫节奏，电子+钢琴，偏暗，BPM约100
2. bgm-start.mp3 — 开场等待界面，静谧略带悬念，比bgm-main更克制，像深夜盯着手机屏幕
3. bgm-soviet.mp3 — 1986年苏联地下电台氛围，模拟信号噪声+神秘弦乐，冷战感，绿色荧光屏既视感，循环
4. bgm-spacetime.mp3 — 跨越时空的无线电频率，失真信号+空灵合成器，疏离感，像在宇宙中等待回应
5. bgm-android.mp3 — 机械入侵感，低频威胁性电子音，红色警报氛围，节奏不规律带抖动，令人不安

SFX（音效，短促）：
6. sfx-msg.mp3 — 手机收到新消息的提示音，清脆简短，类微信消息音但更有设计感，1秒内
7. sfx-correct.mp3 — 正确回复的反馈音，轻快积极，像完成一个小任务，1秒内
8. sfx-wrong.mp3 — 错误回复的反馈音，低沉短促，轻微失落感，不刺耳，1秒内
9. sfx-intrusion-radio.mp3 — 苏联电台突然闯入的音效，模拟信号接通声，嗡嗡电流+短暂静电噪声，约1.5秒
10. sfx-intrusion-android.mp3 — 伪人AI突然入侵的音效，机械警报感，低频脉冲+金属质感，约1.5秒，令人警觉

全部输出 MP3 格式。
```

### MiniMax 提示词（仅转换格式，保持内容不变）

```
我有10个WAV音频文件，请帮我全部转换为MP3格式，保持音质不变，文件名不变只改后缀：

bgm-main.wav → bgm-main.mp3
bgm-start.wav → bgm-start.mp3
bgm-soviet.wav → bgm-soviet.mp3
bgm-spacetime.wav → bgm-spacetime.mp3
bgm-android.wav → bgm-android.mp3
sfx-msg.wav → sfx-msg.mp3
sfx-correct.wav → sfx-correct.mp3
sfx-wrong.wav → sfx-wrong.mp3
sfx-intrusion-radio.wav → sfx-intrusion-radio.mp3
sfx-intrusion-android.wav → sfx-intrusion-android.mp3
```

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
