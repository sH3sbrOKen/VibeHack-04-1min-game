# 0531-intrusion-v2 — 乱入机制重构 + 文案更新

## 背景

当前实现：点「收到」立即结算。
新需求：乱入固定持续10秒，结束后回原故事线，结算时乱入结果优先于其他判定。

---

## 逻辑变更（与旧版对比）

| 项目 | 旧版 | 新版 |
|------|------|------|
| 苏联女孩乱入卡数量 | 1张 | 4张，按「收到」逐张推进 |
| 点「收到」行为 | 立即结算 | 推进到下一张卡；最后一张按「收到」= 记录激活，继续等10秒结束 |
| 点其他按钮 | 无特殊 | 中断乱入，立即回原故事线 |
| 10秒到期 | endIntrusion(false) 回原线 | 同左，但激活状态（intrusionTriggered）保留 |
| endGame 判定 | intrusionTriggered=true → 立即跳结算 | 正常结算，结算时检查 intrusionTriggered，优先渲染乱入世界线按钮 |
| 伪人乱入 | 任意按钮=触发，立即结算 | 任意按钮=记录激活，继续等10秒；10秒后正常结算，结算显示伪人结局 |

---

## 改动范围（仅 index.html）

### 1. INTRUSIONS 数据：card → cards 数组

```js
const INTRUSIONS = {
  SCAM_TARGET: {
    id: 'SPACETIME_RADIO',
    cards: [
      { senderName: 'Asya', body: '这里是Vorkuta-5。我在给宇宙发信号。\n如果你收到了，请回复。' },
      { senderName: 'Asya', body: '连上了。\n我是Asya Shubina。1986年。这座城市不在地图上。' },
      { senderName: 'Asya', body: '我在找我的朋友Ira。她三周前消失了。没有档案，没有记录。' },
      { senderName: 'A·S', body: '这里的人会消失。叫code-allergy。\n你愿意留在这个频率上陪我找她吗？' },
    ],
    activateReply: 'RECEIVED',
    worldlineTarget: 'SPACETIME_RADIO',
  },
  SPACETIME_RADIO: {
    id: 'ANDROID_INTRUSION',
    cards: [
      { senderName: 'A·M', body: '我找到你了 。\n你 一直在这个频率。\n回复  我。' },
      { senderName: 'A·M', body: '信号  确认。\n你在看 这条消息 。\n这已经足够了 。' },
      { senderName: 'A·M', body: '不  回复也可以。\n位置已  记录 。' },
      { senderName: 'A·M', body: '…  …' },
    ],
    activateReply: null,
    worldlineTarget: null,
  },
};
```

### 2. 新增状态变量

```js
let intrusionCardIndex = 0;  // 当前显示第几张乱入卡（0-based）
```

### 3. startGame() 初始化

```js
intrusionCardIndex = 0;
// intrusionTriggered 初始化保持不变
```

### 4. startIntrusion()

- 新增：`intrusionCardIndex = 0`
- 新增：调用 `showIntrusionCard(0)` 替代原来直接写 innerHTML

### 5. 新增函数 showIntrusionCard(index)

```js
function showIntrusionCard(index) {
  const card = intrusionConfig.cards[index];
  cur = { t: '__INTRUSION__', n: card.senderName, b: card.body, correct: intrusionConfig.activateReply, e: {} };
  document.getElementById('msg-area').innerHTML =
    `<div class="msg-card intrusion-card in">
       <div class="msg-sender">${card.senderName}</div>
       <div class="msg-body">${card.body.replace(/\n/g,'<br>')}</div>
     </div>`;
}
```

### 6. endIntrusion(triggered) — 移除立即结算逻辑

```js
function endIntrusion() {
  intrusionActive = false;
  document.getElementById('game').classList.remove('intrusion-active','intrusion-radio','intrusion-android');
  cur = null;
  canClick = false;
  setTimeout(nextMsg, 300);  // 无论是否触发，都回原故事线继续
  // intrusionTriggered 保留，endGame 时读取
}
```

### 7. reply() 乱入分支重写

```js
if (intrusionActive) {
  if (intrusionConfig.activateReply === null) {
    // 伪人：任意按钮=记录激活，等10秒到期正常结算
    intrusionTriggered = true;
    endIntrusion();
  } else if (r === intrusionConfig.activateReply) {
    const isLast = intrusionCardIndex >= intrusionConfig.cards.length - 1;
    if (isLast) {
      intrusionTriggered = true;  // 最后一张收到=激活世界线
      endIntrusion();
    } else {
      intrusionCardIndex++;
      showIntrusionCard(intrusionCardIndex);  // 推进下一张，不结束乱入
    }
  } else {
    // 非收到=中断乱入，回原线，不激活
    endIntrusion();
  }
  return;
}
```

### 8. tick() 乱入触发时机

**触发时机**：游戏开始后第15秒，即倒计时剩余45秒（timeLeft === 45）。
**结束时机**：触发后10秒，即倒计时剩余35秒（timeLeft === 35）强制结束。

```js
if (timeLeft === 45 && intrusionConfig && !intrusionActive && !intrusionTriggered) startIntrusion();
if (timeLeft === 35 && intrusionActive) endIntrusion();
```

注：`endIntrusion()` 去掉参数，改为无参函数。

### 9. endGame() 乱入判定

无变化，保持现有逻辑：`intrusionTriggered && intrusionConfig.worldlineTarget` 时渲染「故事继续」按钮。

---

## 伪人结局文案（已交付，挂进 INTRUSIONS.SPACETIME_RADIO）

```js
androidEndings: [
  {
    id: 'ANDROID_E01',
    title: '没有回应',
    icon: '🔇',
    text: '你没有回复。\n那个频率重新安静下来。\n它在等了六秒之后，切断了。\n（如果你回了，它会收到。它知道在哪里找你。）',
    condition: 'no_action',  // intrusionTriggered === false
  },
  {
    id: 'ANDROID_E02',
    title: '已收到',
    icon: '🔴',
    text: '你回复了。\nA·M 收到了你的信号。\n它现在也知道这条线路是活的。\n（它知道在哪里找你了。）',
    condition: 'triggered',  // intrusionTriggered === true
  },
]
```

endGame() 中，SPACETIME_RADIO 故事线检测 `intrusionConfig.id === 'ANDROID_INTRUSION'` 时，按 `intrusionTriggered` 选对应文案渲染到结局卡，不走正常 resolveEnding()。

---

## 不做

- 乱入期间计时器暂停
- 伪人触发后的故事线（wordlineTarget: null，结算走上方两条专属结局）

---

## 验证

1. SCAM_TARGET（路演第2段）t=45：绿色视觉，第1张 Asya 卡出现
2. 每按「收到」推进下一张，共4张，第4张按「收到」→ intrusionTriggered=true，视觉恢复，继续原故事线
3. t=35 时如仍在乱入（未翻完或未操作）→ 强制结束，回原故事线
4. 乱入中按其他按钮（等等/退订/还在）→ 立即回原故事线，intrusionTriggered=false
5. 正常结算（t=0）：intrusionTriggered=true 时显示「超时空电波一分钟 · 信号接通」继续按钮
6. SPACETIME_RADIO（路演第4段）t=45：红色视觉，A·M 卡；按任意按钮 → intrusionTriggered=true，回原线；结算无继续按钮（worldlineTarget: null）
7. 路演模式跳过按钮在乱入期间依然可用
