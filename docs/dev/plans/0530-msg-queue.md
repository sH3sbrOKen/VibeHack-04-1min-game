# 0530-msg-queue — 消息自动堆叠队列

> 路演前必须完成
> 工作量：中（已实现，本文为事后计划供审核）

---

## 功能描述

消息不等玩家操作，按时间间隔自动入队堆叠。玩家点按钮处理队首那张；未处理时消息持续积压，后面卡片缩小变暗。类比真实手机消息体验。

---

## 改动范围（仅 index.html）

### 1. 新增状态变量

```js
let msgQueue = [];          // 待处理消息队列
let msgPushInterval = 3;    // 保留备用，实际由 tick() 内逻辑控制
```

### 2. tick() 自动推消息

每秒触发，非乱入状态下按剩余时间决定推送频率：

| 剩余时间 | 推送间隔 |
|---------|---------|
| > 30s   | 每3秒   |
| 15-30s  | 每2秒   |
| < 15s   | 每1秒   |

队列为空时立即补充（保证屏幕始终有消息）。队列上限6张，超出不再推。

### 3. 新增函数

**`pickCard()`** — 原 `nextMsg()` 的取卡逻辑抽离：优先 guidanceQueue，再按时间段从 pool1/2/3 取。

**`pushNextCard()`** — 取一张卡 push 进 `msgQueue`，播放 sfx-msg，调 `renderMsgArea()`，更新 `cur` 和 `canClick`。

**`renderMsgArea()`** — 把 `msgQueue` 前4张渲染为堆叠卡列表；超出4张时显示「+N条」提示。

**`nextMsg()`** — 保留为 `pushNextCard()` 的别名。经全文检索确认，`nextMsg()` 在 index.html 中只有一处调用点：`endIntrusion()` 内的 `setTimeout(nextMsg, 300)`。该处语义是「乱入结束后推一张新卡」，与 `pushNextCard()` 完全一致，别名替换后行为不变。

### 4. reply() 改动

- 操作对象从单张 DOM `.msg-card` 改为 `msgQueue[0]`（`.active-card`）
- 处理完后 200ms 延迟移除队首，更新 `cur` 为下一张，重渲染
- 不再调 `setTimeout(nextMsg, delay)`（推消息已由 tick() 驱动）

### 5. 乱入边界处理

- `startIntrusion()`：清空 `msgQueue`，乱入卡独占屏幕
- `endIntrusion()`：清空 `msgQueue` + 清空 msg-area，300ms 后调 `pushNextCard()` 重新开始

### 6. startGameCore() 改动

- 初始化 `msgQueue = []`
- 末尾改为调两次 `pushNextCard()`（开局立即显示2张卡）

### 7. CSS 堆叠样式

`msg-area` 改为纵向 flex + `gap: 8px`。

| class | 效果 |
|-------|------|
| `.active-card` | 正常显示，opacity 1，scale 1 |
| `.stacked-card`（第2张） | opacity 0.55，scale 0.96，pointer-events none |
| `.stacked-card`（第3张） | opacity 0.30，scale 0.93 |
| `.stacked-card`（第4张） | opacity 0.15，scale 0.90 |
| `.msg-more` | 小字提示"+N条" |

---

## 不做（本轮）

- 滑动/拖拽消除卡片
- 消息堆积超阈值时的视觉告警（如背景变色）
- 乱入期间消息继续堆积（乱入结束后重新推）

---

## 验证

1. 开局立即出现2张堆叠卡，后面的卡透明缩小
2. 不操作时每3秒自动新增一张卡
3. 点任意按钮处理队首，200ms后下一张升为 active
4. 剩余15秒内推送变为每秒一张
5. 乱入开始时普通消息全部消失，乱入结束后重新开始堆叠
6. 队列6张时不再新增

---

## Token 估算

中。改动约80行（新增3个函数 + CSS + 状态变量），涉及 tick/reply/startIntrusion/endIntrusion/startGameCore 5处修改。
