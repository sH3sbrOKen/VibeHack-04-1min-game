# 0530-intrusion — 异世界乱入机制

## 目标

在任意故事线的第15秒，触发一次10秒的「异世界乱入」：
- 全屏视觉变成阴森科幻风格
- 出现专属乱入消息卡序列（最多4张）
- 玩家每次点「收到」→ 推进到下一张卡；最后一张点「收到」→ 乱入触发，endGame 后走乱入世界线
- 玩家点其他按钮 / 在任意卡不操作到 t=5 → 乱入结束，返回原故事线正常结算

**苏联女孩乱入：4张卡序列讲完背景，最后一张按「收到」跳入 SPACETIME_RADIO。**

乱入世界线优先级最高，高于所有其他世界线和解锁判定。

---

## 改动范围（仅 index.html）

### 1. 新增数据：INTRUSIONS 配置（1处新增）

```js
const INTRUSIONS = {
  // 苏联女孩乱入：SCAM_TARGET + DEMO第3条MAIN触发
  // 玩家依次按收到推进4张卡，最后一张按收到跳 SPACETIME_RADIO
  SCAM_TARGET: {
    id: 'SPACETIME_RADIO',
    cards: [
      {
        senderName: 'Asya',
        body: '这里是Vorkuta-5。我在给宇宙发信号。\n如果你收到了，请回复。',
      },
      {
        senderName: 'Asya',
        body: '连上了。\n我是Asya Shubina。1986年。这座城市不在地图上。',
      },
      {
        senderName: 'Asya',
        body: '我在找我的朋友Ira。她三周前消失了。没有档案，没有记录。',
      },
      {
        senderName: 'A·S',
        body: '这里的人会消失。叫code-allergy。\n你愿意留在这个频率上陪我找她吗？',
      },
    ],
    activateReply: 'RECEIVED',  // 每张卡按收到才推进；最后一张按收到=激活
    worldlineId: 'SPACETIME_CHAIN',
  },
  // 伪人乱入：SPACETIME_RADIO触发，正确做法是不回复（10秒内不点任何按钮）
  SPACETIME_RADIO: {
    id: 'ANDROID_INTRUSION',
    cards: [
      {
        senderName: 'A·M',
        body: '我找到你了。\n你一直在这个频率。\n回复我。',
      },
    ],
    activateReply: null,  // null = 点任何按钮都算激活（触发坏结局），不点才是正确做法
    worldlineId: null,    // 暂不做伪人1分钟故事线，触发后兜底结算
  },
};
```

> **字段变更说明**：`card`（单卡对象）→ `cards`（数组）。程序读取时用 `intrusionCardIndex` 追踪当前卡下标，每次 activateReply 命中时 index++；到最后一张时命中=激活乱入。

**DEMO 模式下 MAIN 乱入问题（⚠️ 程序自行决定实现方式）**：

DEMO 序列为 `MAIN → SCAM_TARGET → MAIN → SPACETIME_RADIO`，第1条和第3条都是 MAIN，但：
- 第1条 MAIN：制作人演示跳过按钮用，**不应触发乱入**
- 第3条 MAIN：**应触发乱入**，让观众看到苏联女孩乱入效果

当前 INTRUSIONS 按 storylineId 索引，无法区分第几次进入。程序决定如何实现（可考虑：DEMO 模式下用序列位置索引覆盖配置；或在 DEMO_WORLDLINE 中额外标记每条是否启用乱入；等等）。

### 2. 新增状态变量（4个）

```js
let intrusionActive = false;
let intrusionTriggered = false;
let intrusionConfig = null;       // 当前故事线的乱入配置，startGame 时赋值
let intrusionCardIndex = 0;       // 当前显示第几张乱入卡（0-based）
```

### 3. tick() 新增触发检测（1处）

```js
// 在 timeLeft === 15 时触发乱入
if (timeLeft === 15 && intrusionConfig && !intrusionTriggered) {
  startIntrusion();
}
// 在 timeLeft === 5 时强制结束乱入（仅未触发时执行，触发时已在 reply() 里提前 endGame）
if (timeLeft === 5 && intrusionActive && !intrusionTriggered) {
  endIntrusion(false);
}
```

### 4. 新增函数 startIntrusion()

- `intrusionActive = true`
- `intrusionCardIndex = 0`
- `#game` 加 CSS class `intrusion-active`（以及 `intrusion-radio` 或 `intrusion-android`，由 `intrusionConfig.id` 决定）
- 调用 `showIntrusionCard(0)` 渲染第一张卡

**新增函数 showIntrusionCard(index)**

- 清空 `#msg-area`，渲染 `intrusionConfig.cards[index]`（发信人名、正文，专属 class）
- 乱入卡有独立 CSS 样式（见第9条）

### 5. 新增函数 endIntrusion(triggered)

- 移除 `#game` 上的 `intrusion-active` class
- `intrusionActive = false`
- 清空乱入卡
- `triggered=false` → 正常 nextMsg()，游戏继续
- `triggered=true` → 立即 `clearInterval(tik); endGame()`，与跳过按钮行为一致
  - 无需屏蔽 nextMsg()，endGame 已在 clearInterval 后立刻触发，tick 不再执行

### 6. reply() 新增乱入分支（1处）

```js
// reply() 开头插入
if (intrusionActive) {
  if (intrusionConfig.activateReply === null) {
    // 伪人乱入：任意回复=激活坏结局
    intrusionTriggered = true;
    endIntrusion(true);
    return;
  }
  if (r === intrusionConfig.activateReply) {
    const isLastCard = intrusionCardIndex >= intrusionConfig.cards.length - 1;
    if (isLastCard) {
      // 最后一张卡按收到=激活乱入世界线
      intrusionTriggered = true;
      endIntrusion(true);
    } else {
      // 还有后续卡，推进到下一张
      intrusionCardIndex++;
      showIntrusionCard(intrusionCardIndex);
    }
  } else {
    // 按了错误按钮（非收到）=中止乱入，回原故事线
    endIntrusion(false);
  }
  return;
}
```

**行为汇总**：
- 苏联女孩：每张按「收到」→ 推进；按其他按钮 → 结束乱入回原线；最后一张按「收到」→ 激活
- 伪人：任意按钮 → 激活坏结局；不操作到 t=5 → 正常结算

### 7. endGame() 新增乱入世界线优先判定（1处，在现有世界线判定之前）

```js
if (intrusionTriggered && intrusionConfig) {
  // 替换原 resolveWorldlineNext() 结果
  showIntrusionWorldlineNext(intrusionConfig.worldlineId);
  return;  // 不走原世界线逻辑
}
```

注：`showIntrusionWorldlineNext` 直接在结算页渲染「故事继续」按钮，与现有 worldline 按钮样式一致。

### 8. startGame() 初始化乱入配置（1处）

```js
intrusionConfig = INTRUSIONS[storylineId] || null;
intrusionActive = false;
intrusionTriggered = false;
intrusionCardIndex = 0;
```

### 9. 新增 CSS：intrusion-active 视觉效果

乱入卡类型由 `intrusionConfig.id` 区分，程序在触发时给 `#game` 加不同 class：

**苏联女孩乱入**（`id: 'SPACETIME_RADIO'`）→ `#game.intrusion-active.intrusion-radio`
- 背景色 → `#0a0a0f`（近黑）
- 扫描线动画（repeating-linear-gradient + 伪元素）
- 所有文字色 → `#00ff9d`（终端绿）
- 消息卡背景 → `#0d1117`，边框 `1px solid #00ff9d`
- 闪烁/glitch keyframe（间歇性 `filter: hue-rotate`）

**伪人乱入**（`id: 'ANDROID_INTRUSION'`）→ `#game.intrusion-active.intrusion-android`
- 背景色 → `#0f0a0a`（近黑偏红）
- 所有文字色 → `#ff3333`（警报红）
- 消息卡背景 → `#1a0a0a`，边框 `1px solid #ff3333`，边框快速闪烁（opacity keyframe 0.2s）
- 无扫描线，改为屏幕整体轻微抖动（`@keyframes shake`，振幅2px）
- 四角按钮整体变暗（`opacity: 0.3`），暗示不应该按

---

## activateReply: null 的处理逻辑

当 `activateReply === null` 时（伪人乱入），回复行为语义反转：
- 玩家**点任何按钮** → `intrusionTriggered = true` → `endIntrusion(true)` → 走乱入世界线（当前为 null，直接兜底结算，显示专属结局文案）
- 玩家**不操作到 t=5** → `endIntrusion(false)` → 正常结算，无乱入影响

与 `activateReply: 'RECEIVED'` 的区别：触发条件从「点特定按钮」变为「点任意按钮」。程序判断分支：
```js
if (intrusionActive) {
  if (intrusionConfig.activateReply === null) {
    // 任意回复都算触发
    intrusionTriggered = true;
  } else if (r === intrusionConfig.activateReply) {
    intrusionTriggered = true;
  }
  endIntrusion(intrusionTriggered);
  return;
}
```

---

## 不做的

- 乱入期间计时器暂停（主计时器继续跑，保持压迫感）
- 乱入消息卡的答错数值惩罚（乱入不影响数值，只影响世界线走向）
- 多条乱入卡（一次乱入只出一张卡）
- 伪人乱入触发后的1分钟故事线（低优先级，暂缓）

---

## 验证

1. 主线第15秒：全屏变暗绿色，出现 Asya 第1张乱入卡
2. 点「收到」→ 依次推进卡2、卡3、卡4（共4张）
3. 第4张按「收到」→ 立即结算，结算页出现乱入世界线「故事继续」按钮
4. 任意卡点「等等/切断/还在」→ 视觉恢复，正常结算，无乱入按钮
5. 不操作到 t=5 → 视觉恢复，正常结算，无乱入按钮
6. SCAM_TARGET 第15秒同样触发，行为与主线一致
7. SPACETIME_RADIO 第15秒：出现伪人乱入卡，点任意按钮 → 坏结局；不操作 → 正常结算
8. 乱入触发后，原故事线的世界线判定（escapeType）不影响结算页的乱入按钮
