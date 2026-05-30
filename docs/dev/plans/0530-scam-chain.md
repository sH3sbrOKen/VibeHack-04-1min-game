# 0530-scam-chain — 多分支脱离 + SCAM_CHAIN 世界线 + DATA_AUCTION

## 目标

1. escape 系统从 boolean 升级为具名分支（支持单故事线多脱离路径）
2. WORLDLINES 按分支名路由下一故事线
3. SCAM_TARGET escape 拆为 silent / defiant 两条分支
4. 注册 DATA_AUCTION 故事线
5. 注册 SCAM_CHAIN 世界线

## 改动范围（仅 index.html）

### 1. escapeType 变量（1处新增）
`escaped = false` → `escapeType = null`（null=未脱离，字符串=脱离类型）

### 2. checkEscape → resolveEscapeType（2处：声明1 + endGame调用1）
返回 `null | string`，不再返回 boolean。checkEscape 仅在 endGame 调用一次，无其他引用。

### 3. WORLDLINES 结构（1处升级）
nodes 由 `{ escaped, notEscaped }` 改为 `{ [escapeType]: nextId, null: nextId }`：
```js
SCAM_TARGET: { 'silent':'MAIN', 'defiant':'DATA_AUCTION', null:'MAIN' }
DATA_AUCTION: { 'silent':'MAIN', null:'MAIN' }
```

### 4. resolveWorldlineNext（1处改动）
用 `escapeType` 作 key 查表，`null` 对应未脱离分支。

### 5. endGame 中两处引用 escaped → escapeType（2处）
- `checkEscape()` → `resolveEscapeType()`
- 传给世界线结局判定

### 6. STORYLINES 新增两条（1处）
- SCAM_TARGET.escape.fn 更新为返回 `'silent' | 'defiant' | null`
- DATA_AUCTION 全量数据

### 7. WORLDLINES 注册 SCAM_CHAIN（1处）

## 不做的
- SCAM_E02「较真」结局触发后的视觉提示（目前只显示结局卡，等世界线跳转自动处理）

## 验证
1. SCAM_TARGET 60秒不操作 → 结局「消失术」→ 回主线
2. SCAM_TARGET 全部退订 → 结局「较真」→ 结算页出现「名单拍卖一分钟 继续」按钮
3. SCAM_TARGET 混乱回复 → 结局「名单保留」→ 回主线
4. 进入 DATA_AUCTION → 开场旁白「你的号码正在被竞价」
5. DATA_AUCTION 不操作 → 结局「流拍」→ 回主线
