# 0530-custom-import — 玩家导入自定义消息记录

> 路演前必须完成
> 截止：2026-05-30 18:00
> 工作量：小（1.5~2h）

---

## 功能描述

玩家在开始页点击「导入我的消息」→ 粘贴自己的聊天记录 → 解析为消息卡 → 跑现有1分钟引擎 → 正常结算。

无世界线关联、无解锁系统、无乱入。纯粹「用自己的消息体验一分钟」。

---

## 输入格式

每行一条消息，支持：

```
张三：明天开会记得准时
外卖小哥：您的订单已送达，请下楼取餐
18888888888：您有一笔贷款待领取
妈妈：在吗
```

- 有「：」→ 冒号前为发信人，冒号后为正文
- 无「：」→ 发信人显示「未知」，整行为正文
- 空行跳过
- 最少3条，最多30条（超出截断，提示用户）
- 支持中文全角冒号「：」和半角「:」

---

## 改动范围（仅 index.html）

### 1. 开始页新增入口按钮

在 `#start-screen` 的「开始」按钮下方加一行：

```html
<button class="btn-secondary" onclick="showImportModal()">导入我的消息</button>
```

### 2. 新增导入 modal（HTML）

```html
<div id="import-modal" style="display:none">
  <div class="modal-overlay" onclick="hideImportModal()"></div>
  <div class="modal-box">
    <div class="modal-title">导入消息记录</div>
    <div class="modal-hint">每行一条，格式：发信人：消息内容</div>
    <textarea id="import-textarea" placeholder="张三：明天开会&#10;外卖小哥：您的订单已送达&#10;18888888888：您有一笔贷款待领取"></textarea>
    <div id="import-error" class="import-error"></div>
    <div class="modal-actions">
      <button onclick="hideImportModal()">取消</button>
      <button class="btn-primary" onclick="startCustomGame()">开始</button>
    </div>
  </div>
</div>
```

### 3. 新增 CSS

```css
.modal-overlay { position:fixed;inset:0;background:rgba(0,0,0,.5);z-index:99 }
.modal-box { position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);
  background:#fff;border-radius:12px;padding:24px;width:320px;z-index:100;
  display:flex;flex-direction:column;gap:12px }
.modal-title { font-size:16px;font-weight:600 }
.modal-hint { font-size:12px;color:#888 }
#import-textarea { height:160px;resize:none;border:1px solid #ddd;
  border-radius:8px;padding:10px;font-size:13px;line-height:1.6 }
.import-error { font-size:12px;color:#e55 }
.modal-actions { display:flex;gap:8px;justify-content:flex-end }
```

### 4. 新增解析函数 parseCustomMessages(text)

```js
function parseCustomMessages(text) {
  const lines = text.split('\n').map(l => l.trim()).filter(Boolean);
  if (lines.length < 3) return { error: '至少需要3条消息' };
  const cards = lines.slice(0, 30).map(line => {
    const sep = line.indexOf('：') !== -1 ? '：' : ':';
    const idx = line.indexOf(sep);
    if (idx > 0 && idx < line.length - 1) {
      return { senderName: line.slice(0, idx).trim(), body: line.slice(idx + 1).trim() };
    }
    return { senderName: '未知', body: line };
  });
  return { cards };
}
```

### 5. 新增 createCustomStoryline(cards)

把解析出的卡数组包装成引擎能用的故事线对象：

```js
function createCustomStoryline(cards) {
  return {
    id: '__CUSTOM__',
    name: '我的消息',
    intro: '一分钟。\n你的手机里有这些消息。\n（它们都在等你。）',
    statCfg: {},          // 无数值，不显示数值条
    defaults: {},
    rules: [],
    raw: cards.map((c, i) => ({
      t: 'CUSTOM',
      n: c.senderName,
      b: c.body,
      correct: 'RECEIVED',  // 任意回复不扣分
      e: {},
    })),
    tmpl: {},
    genCount: 0,          // 全用 raw 卡
    ui: {
      batteryText: '消息',
      buttons: { WAIT: '等等', RECEIVED: '已读', UNSUBSCRIBE: '不回', LOVE: '在的' },
    },
    endings: [
      {
        id: 'CUSTOM_E01',
        title: '看完了',
        icon: '📱',
        text: '这一分钟过去了。\n那些消息还在。\n（有几条你其实没想好怎么回。）',
        priority: 1,
        condition: () => true,
      },
    ],
  };
}
```

### 6. 新增 showImportModal / hideImportModal / startCustomGame

```js
function showImportModal() {
  document.getElementById('import-modal').style.display = 'block';
  document.getElementById('import-error').textContent = '';
}
function hideImportModal() {
  document.getElementById('import-modal').style.display = 'none';
}
function startCustomGame() {
  const text = document.getElementById('import-textarea').value;
  const result = parseCustomMessages(text);
  if (result.error) {
    document.getElementById('import-error').textContent = result.error;
    return;
  }
  if (result.cards.length === 30 && text.split('\n').filter(Boolean).length > 30) {
    document.getElementById('import-error').textContent = '超过30条，已截取前30条。';
    // 不阻断，继续
  }
  const sl = createCustomStoryline(result.cards);
  STORYLINES['__CUSTOM__'] = sl;
  hideImportModal();
  startGame('__CUSTOM__');
}
```

### 7. rebuildStatChips() 兼容处理

`__CUSTOM__` 故事线 `statCfg` 为空对象，`rebuildStatChips({})` 需在清空 bar 后立即隐藏并返回。

插入位置：`index.html` 第1331行，`bar.innerHTML = ''` 之后，`Object.entries(cfg)` 之前：

```js
function rebuildStatChips(cfg) {
  const bar = document.getElementById('stats-bar');
  bar.innerHTML = '';
  // ↓ 新增2行
  if (Object.keys(cfg).length === 0) { bar.style.display = 'none'; return; }
  bar.style.display = '';
  // ↑ 新增结束
  Object.entries(cfg).forEach(([k, c]) => {
    // 以下原有逻辑不变
  });
```

---

## 不做（本轮）

- 保存/加载自定义记录（localStorage）
- 自定义结局
- 格式容错（如微信导出格式）
- 自定义回复按钮文案

---

## 验证

1. 开始页出现「导入我的消息」按钮
2. 输入少于3条 → 报错提示
3. 粘贴5条「张三：xxx」格式 → 正常开始游戏
4. 无冒号行 → 发信人显示「未知」
5. 数值条隐藏
6. 游戏结束 → 显示兜底结局「看完了」
7. 点任意回复按钮 → 不触发负数值/crash

---

## Token 估算

小。单 HTML 改动，新增约 100 行（含 HTML/CSS/JS），无引擎改动。
