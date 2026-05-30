# 0531-spacetime-integration — SPACETIME_RADIO 集成 + 乱入升级

> 本文件已于 2026-05-30 对齐最新文案（Asya/Vorkuta-5/1986/2数值）。
> 以 STORYLINE-SPACETIME_RADIO.md + 0530-intrusion.md 为唯一权威来源。

---

## 改动范围（仅 index.html）

### A. INTRUSIONS 数据更新

`card`（单卡对象）→ `cards`（数组）。苏联女孩乱入改为4张卡序列。

```js
const INTRUSIONS = {
  // 苏联女孩乱入：SCAM_TARGET + DEMO第3条MAIN触发
  // 玩家依次按「收到」推进4张卡，最后一张按「收到」跳 SPACETIME_RADIO
  SCAM_TARGET: {
    id: 'SPACETIME_RADIO',
    cards: [
      { senderName: 'Asya',  body: '这里是Vorkuta-5。我在给宇宙发信号。\n如果你收到了，请回复。' },
      { senderName: 'Asya',  body: '连上了。\n我是Asya Shubina。1986年。这座城市不在地图上。' },
      { senderName: 'Asya',  body: '我在找我的朋友Ira。她三周前消失了。没有档案，没有记录。' },
      { senderName: 'A·S',   body: '这里的人会消失。叫code-allergy。\n你愿意留在这个频率上陪我找她吗？' },
    ],
    activateReply: 'RECEIVED',
    worldlineId: 'SPACETIME_CHAIN',
  },
  // 伪人乱入：SPACETIME_RADIO触发，不操作才是正确做法
  SPACETIME_RADIO: {
    id: 'ANDROID_INTRUSION',
    cards: [
      { senderName: 'A·M', body: '我找到你了 。\n你 一直在这个频率。\n回复  我。' },
      { senderName: 'A·M', body: '信号  确认。\n你在看 这条消息 。\n这已经足够了 。' },
      { senderName: 'A·M', body: '不  回复也可以。\n位置已  记录 。' },
      { senderName: 'A·M', body: '…  …' },
    ],
    activateReply: null,   // null = 点任何按钮都激活坏结局
    worldlineId: null,     // 伪人1分钟故事线暂缓，触发后兜底结算
  },
};
```

---

### B. 状态变量新增

```js
let intrusionActive = false;
let intrusionTriggered = false;
let intrusionConfig = null;
let intrusionCardIndex = 0;   // 新增：追踪当前显示第几张乱入卡
```

---

### C. reply() 乱入分支（完整版）

```js
if (intrusionActive) {
  if (intrusionConfig.activateReply === null) {
    // 伪人乱入：任意按钮=激活坏结局
    intrusionTriggered = true;
    endIntrusion(true);
    return;
  }
  if (r === intrusionConfig.activateReply) {
    const isLastCard = intrusionCardIndex >= intrusionConfig.cards.length - 1;
    if (isLastCard) {
      intrusionTriggered = true;
      endIntrusion(true);
    } else {
      intrusionCardIndex++;
      showIntrusionCard(intrusionCardIndex);
    }
  } else {
    // 按了非收到的按钮=中止乱入，回原故事线
    endIntrusion(false);
  }
  return;
}
```

---

### D. CSS 双视觉风格

`startIntrusion()` 根据 `intrusionConfig.id` 给 `#game` 加以下 class：

**苏联女孩**（id: 'SPACETIME_RADIO'）→ `intrusion-active intrusion-radio`
- 背景 `#0a0a0f`，文字 `#00ff9d`
- 消息卡：背景 `#0d1117`，边框 `1px solid #00ff9d`
- 扫描线（repeating-linear-gradient 伪元素）
- glitch keyframe（间歇性 `filter: hue-rotate`）

**伪人**（id: 'ANDROID_INTRUSION'）→ `intrusion-active intrusion-android`
- 背景 `#0f0a0a`，文字 `#ff3333`
- 消息卡：背景 `#1a0a0a`，边框 `1px solid #ff3333`，0.2s 闪烁（opacity keyframe）
- 整屏轻微抖动（`@keyframes shake`，振幅2px）
- 四角按钮 `opacity: 0.3`（暗示不应该按）

---

### E. SPACETIME_RADIO 故事线集成

在 STORYLINES 中新增完整故事线对象。

**基本信息**
```js
{
  id: 'SPACETIME_RADIO',
  intro: '1986年。Vorkuta-5。\n一座在地图上不存在的城市。\n有人爬上了市政塔，向宇宙发出信号。\n（她不知道有人在听。）',
  ui: {
    batteryText: '信号 1格',
    buttons: {
      WAIT: '等等',
      RECEIVED: '收到',
      UNSUBSCRIBE: '切断',
      LOVE: '还在',
    },
  },
}
```

**数值系统（2个）**

| StatKey | 显示名 | 方向 | 初始 | 阈值 |
|---------|--------|------|------|------|
| SIGNAL_STRENGTH | 信号强度 | ↓坏 | 60 | ≤0 触发数值结局 |
| CONNECTION | 联络确认数 | ↑好 | 0 | ≥20 触发组合结局 |

**脱离条件**：SIGNAL_STRENGTH ≥ 80（escapeType: 'signal_success'）

**发信人类型（3种）**

| 类型ID | 显示名池 | 默认正确回复 | 答错效果 |
|--------|---------|------------|---------|
| ASYA | Asya / Asya S. / A·S | RECEIVED | SIGNAL_STRENGTH: -15 |
| STATIC | 静电 / 噪声 / ·——· | WAIT | SIGNAL_STRENGTH: -5 |
| ECHO | 回声 / 残影 / …… | RECEIVED | SIGNAL_STRENGTH: -10 |

**规则覆盖（BodyOverrideRule）**

| 正文含 | 覆盖为 |
|--------|--------|
| 你还在吗 | RECEIVED |
| 请继续回复 | RECEIVED |
| 信号在衰减 | RECEIVED |

**手写卡**：21张（STORYLINE-SPACETIME_RADIO.md E.1）
- ASYA 叙事卡：15张（#1–#15）
- STATIC 莫尔斯：3张（#16–#18）
- ECHO 回声：3张（#19–#21）

**消息模板**：见 STORYLINE-SPACETIME_RADIO.md E.2
- ASYA：4条模板含变量池
- STATIC：1条模板（纯莫尔斯符号）
- ECHO：1条模板（…… + fragment ……）

**结局（6个）**

| 类型 | 触发条件 | 标题 | 文案 |
|------|---------|------|------|
| 数值 | SIGNAL_STRENGTH ≤ 0 | 信号中断 | 信号消失了。\n像Vorkuta-5本身，没有留下任何标记。\n（Asya 还在塔上。）\n（致敬《Z.A.T.O.》……） |
| 组合 | SIGNAL_STRENGTH ≤ 0 + CONNECTION ≥ 10 | 听见了一部分 | 信号断之前，你回了十次以上。\n她不知道。\n但那十次是真实存在过的。\n（致敬《Z.A.T.O.》……） |
| 特殊 | SIGNAL_STRENGTH ≥ 80（脱离） | 有人在听 | 信号维持住了。\nAsya 继续发送了很久。\n她不知道有人在听。\n（致敬《Z.A.T.O.》……） |
| 特殊 | 正确处理 ≥ 20 条 | 守在频率上 | 你回了二十条以上。\n这个频率被你撑了一分钟。\n（没有人相信这件事发生过。）\n（致敬《Z.A.T.O.》……） |
| 兜底 | — | 杂音 | 这一分钟过去了。\nAsya 还在塔上。1986年的 Vorkuta-5 还在继续消失。\n（致敬《Z.A.T.O.》……） |

> 完整文案见 STORYLINE-SPACETIME_RADIO.md F 节。「……」处补全致敬句：「一款关于消失与爱的游戏。去玩吧。」

**伪人乱入触发后的兜底结局**（worldlineId: null 时显示）
```
标题：频率占用
文案：你回复了。\n那个频率现在属于别的东西了。\n（不要在不认识的频率上回复陌生信号。）
```
无「故事继续」按钮，只显示「再来一局」。

---

### F. DEMO 序列更新

```js
// 旧
let demoSequence = ['MAIN', 'SCAM_TARGET', 'DATA_AUCTION'];

// 新（方案一：加 intrusion 标记）
let demoSequence = [
  { id: 'MAIN' },
  { id: 'SCAM_TARGET' },
  { id: 'MAIN', intrusion: 'SCAM_TARGET' },  // 第15秒触发苏联女孩乱入
  { id: 'SPACETIME_RADIO' },
];
```

`startGame()` 改为读 `demoSequence[demoIndex].intrusion` 覆盖 intrusionConfig：
```js
intrusionConfig = INTRUSIONS[currentStep.intrusion || storylineId] || null;
```

> 方案一改动最小，逻辑清晰，不影响正常游戏，程序直接采用。

---

## 不做（本轮）

- 伪人乱入后的1分钟故事线（ANDROID_INTRUSION worldlineId:null，兜底结算即可）
- SPACETIME_RADIO 的独立解锁入口（仅从 SPACETIME_CHAIN 世界线进入）

---

## 实现顺序建议

1. `跳过按钮` — 不依赖文案，立即可做（见 0530-demo-worldline.md 第1条）
2. `D-3 新手引导` — buildQueue() 前3条固定（见 VERSIONS.md D-3）
3. `D-5 结算叙述句` — endGame() 替换数字表（见 VERSIONS.md D-5）
4. `乱入机制` — INTRUSIONS + 双视觉 CSS + 4卡序列 reply 逻辑（本文 A–D）
5. `SPACETIME_RADIO 集成` — 故事线数据 + 结局（本文 E）
6. `DEMO 演示模式` — 开始页按钮 + demoSequence 串联（本文 F + 0530-demo-worldline.md）

---

## 验证

1. SCAM_TARGET 第15秒：暗绿扫描线，弹出第1张 Asya 卡
2. 连按「收到」×4：依次推进4张乱入卡
3. 第4张按「收到」→ 立即结算，结算页有「故事继续」按钮（进入 SPACETIME_RADIO）
4. 任意卡按「等等/切断/还在」→ 视觉恢复，正常结算，无乱入按钮
5. SPACETIME_RADIO 第15秒：红色抖动，A·M 卡，按钮 opacity:0.3
6. 点任意按钮 → 坏结局「频率占用」，无「故事继续」；不操作到t=5 → 正常结算
7. SPACETIME_RADIO 故事线：开场「1986年。Vorkuta-5。」，按钮「等等/收到/切断/还在」，顶部「信号 1格」
8. 结局渲染：信号中断 / 听见了一部分 / 有人在听 / 守在频率上 / 杂音（5个，按触发条件验收）
9. DEMO 路演：MAIN（跳过）→ SCAM_TARGET → MAIN（第15秒触发乱入）→ SPACETIME_RADIO
