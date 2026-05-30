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
