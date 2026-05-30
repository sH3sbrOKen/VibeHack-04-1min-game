# WORLDLINES — 世界线定义

---

## SCAM_CHAIN — 诈骗深渊线

**起点**：SCAM_TARGET（被盯上一分钟）
**终点**：DATA_AUCTION（名单拍卖一分钟）

### 跳转规则

SCAM_TARGET
  ├── 脱离（60秒不操作）→ 回到主线
  ├── 未脱离（混乱回复）→ 停留，兜底结局，回到主线
  └── 特殊脱离（全部UNSUBSCRIBE）→ DATA_AUCTION

DATA_AUCTION
  ├── 未脱离 → （世界线结束，回到主线）
  └── 脱离 → （世界线结束，回到主线）

---

## SPACETIME_CHAIN — 超时空电波线

**起点**：乱入触发（MAIN 或 SCAM_TARGET 第15秒苏联女孩乱入，玩家点「收到」）
**终点**：SPACETIME_RADIO

### 跳转规则

乱入触发
  └── 触发（玩家点收到）→ SPACETIME_RADIO

SPACETIME_RADIO
  ├── 未脱离 → 世界线结束，回到主线
  └── 脱离 → 世界线结束，回到主线

---

## DEMO — 路演演示世界线

**用途**：黑客松路演演示专用，开始页「演示模式」按钮启动。
**顺序固定**，由现有故事线构成，玩家无需触发解锁。

### 演示序列

sequence:
  1. MAIN（演示跳过按钮，制作人可直接跳过）
  2. SCAM_TARGET（体验诈骗公司盯梢）
  3. MAIN（正常玩到第15秒苏联女孩乱入，点收到触发 SPACETIME_CHAIN）
  4. SPACETIME_RADIO（苏联女孩一分钟）
