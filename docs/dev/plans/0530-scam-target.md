# 0530-scam-target — 被盯上一分钟故事线

## 目标

把 STORYLINE-SCAM_TARGET.md 注册进 index.html，可从主线解锁并进入游玩。世界线跳转暂不实现（缺 WORLDLINES.md）。

## 改动

**仅改 index.html 一处**：在 `STORYLINES` 对象中新增 `SCAM_TARGET` 条目。

## 数据映射

### 进入条件
策划写的是「对SPAM类型的错误回应占比 ≥ 所有类型中最高」。转为代码：`replyLog` 中按 senderType 分组统计错误数，SPAM 类错误数最多且 ≥ 1。用 `unlock.type:'behavior'` + `fn` 实现。

### 脱离条件
两个分支，用 `escape.type:'custom'` + `fn` 实现：
- `replied === 0` → 脱离（消失术）
- 所有回复均为 UNSUBSCRIBE → 脱离（较真）
- 其他 → 未脱离

### 结局
3个结局，全部行为判定：
- 消失术（priority 90）：`replied === 0`
- 较真（priority 85）：所有回复均为 UNSUBSCRIBE 且 `replied > 0`
- 名单保留（priority 1）：兜底

### 发信人/规则/数值/卡池/模板
从 md 表格直接转 JS 对象，无额外逻辑。

## 不做的

- 世界线跳转（缺 WORLDLINES.md）
- 验收标准第1条"4种回复都用到"的豁免（已确认是设计意图）

## 验证

1. 玩主线，故意对 SPAM 消息全部回错 → 结算页应出现「被盯上一分钟 已解锁」+ 进入按钮
2. 进入后看到开场旁白「你的号码进了一个名单...」
3. 60秒完全不操作 → 结局「消失术」
4. 全部点退订 → 结局「较真」
5. 混合操作 → 结局「名单保留」
