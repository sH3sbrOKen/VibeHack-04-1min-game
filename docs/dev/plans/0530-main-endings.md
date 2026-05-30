# 0530-main-endings — 主线结局数据合并

## 目标

把策划交付的 STORYLINE-MAIN.md 中 10 个结局写入 index.html 的 `STORYLINES.MAIN.endings` 数组。

## 改动范围

**仅改 1 处**：`index.html` 的 `STORYLINES.MAIN.endings` 数组（当前只有 1 个兜底结局，替换为完整 10 个）。

## 数据映射

从 STORYLINE-MAIN.md 表格直接转为 JS 对象：

- F.1 数值结局 × 6：condition 检查对应 stats 阈值，priority 80
- F.2 组合结局 × 2：condition 检查多个 stats，priority 90
- F.3 特殊结局 × 1：condition 检查 handled ≥ 30，priority 85
- F.4 兜底结局 × 1：condition 返回 true，priority 1

文案中的 `\n` 转为 JS 字符串换行符，原样保留。

## 验证

1. 浏览器打开 index.html 玩一局
2. 结算页应显示结局卡（不再只是「普通的早晨」）
3. 故意全部乱回 → 应触发数值结局或组合结局
4. 底部显示「结局收集 1/10」
