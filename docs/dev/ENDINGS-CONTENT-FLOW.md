# ENDINGS-CONTENT-FLOW — 结局文案批量录入流程

> 适用于100个结局的低成本配置方案。
> 文案和逻辑分离：文案用 Excel 本地填写，逻辑由程序配。

---

## 分工

| 角色 | 工具 | 任务 |
|------|------|------|
| 文案 | Excel（本地 .xlsx） | 按格式填结局文案，保存文件 |
| 程序 | Claude Code | 读 .xlsx → 生成 endings[] 对象，挂进对应故事线 |
| PM | 当前窗口 | 审查计划，不介入实现 |

---

## 文案表格格式（Excel）

文件名：`endings-draft.xlsx`，放在项目根目录。
Sheet 名：`endings`（只用这一个 sheet）。
第一行为列名，从第二行开始填内容。

| 列名 | 必填 | 说明 | 示例 |
|------|------|------|------|
| `id` | 是 | 全局唯一，格式 `故事线缩写_E序号` | `MAIN_E11` |
| `storyline` | 是 | 挂进哪个故事线 | `MAIN` |
| `title` | 是 | 结局标题，4-8字 | `余额归零` |
| `icon` | 是 | 单个 emoji | `💸` |
| `text` | 是 | 3行正文，行间用 `\n` 写字面量，每行不超过20字 | `本月意外支出超过两百元。\n大部分你不知道买了什么。\n（充了个会员，平台已跑路。）` |
| `priority` | 是 | 数字，越高越优先；组合结局90，单项80，兜底1 | `80` |
| `condition_desc` | 是 | 中文描述触发条件，程序据此写代码 | `INFO_LEAK>=3 且 SPAM_SUBSCRIBED>=5` |
| `note` | 否 | 备注，不进游戏 | `和E03组合版` |

> `text` 字段在 Excel 单元格内直接输入 `\n` 两个字符（不是真正换行），程序读取后会转换为 JS 字符串换行。

---

## 文案操作流程

1. 用 Excel 打开（或新建）`endings-draft.xlsx`
2. Sheet 改名为 `endings`，第一行填列名
3. 按格式逐行填写结局，`text` 字段用 `\n` 分隔三行正文
4. 保存文件（Ctrl+S），**不需要导出 CSV**
5. 通知程序：`endings-draft.xlsx 已就绪，请导入`

---

## 程序操作流程（程序看这里）

### 一次性准备：安装依赖 + 写转换脚本

```bash
# 在项目根目录执行，只需一次
npm install xlsx --save-dev
```

在项目根目录创建 `tools/xlsx-to-endings.js`：

```js
// 用法：node tools/xlsx-to-endings.js endings-draft.xlsx
const XLSX = require('xlsx');

const file = process.argv[2] || 'endings-draft.xlsx';
const wb = XLSX.readFile(file);
const ws = wb.Sheets['endings'];
if (!ws) { console.error('找不到 sheet: endings'); process.exit(1); }

const rows = XLSX.utils.sheet_to_json(ws, { defval: '' });
const byStoryline = {};

rows.forEach(row => {
  if (!row.id || !row.storyline) return;
  const entry = `      {id:'${row.id}',title:'${row.title}',text:'${row.text}',icon:'${row.icon}',priority:${row.priority},condition:/* ${row.condition_desc} */ null},`;
  if (!byStoryline[row.storyline]) byStoryline[row.storyline] = [];
  byStoryline[row.storyline].push(entry);
});

for (const [sl, entries] of Object.entries(byStoryline)) {
  console.log(`\n// ── ${sl} endings ──`);
  console.log('    endings: [');
  entries.forEach(e => console.log(e));
  console.log('    ],');
}
```

### 每次录入步骤

1. `node tools/xlsx-to-endings.js endings-draft.xlsx > endings-output.txt`
2. 打开 `endings-output.txt`，逐条把 `condition: null` 替换为实际 JS 函数（依据 `condition_desc`）
3. 将整个 `endings: [...]` 块粘贴进 `index.html` 对应故事线
4. 浏览器验证：触发各结局，确认文案和触发条件匹配
5. 回复 PM：「已导入 N 条结局，来源：endings-draft.xlsx」

### condition 写法参考

| condition_desc 示例 | 对应代码 |
|---------------------|---------|
| `DELIVERY_LOST>=3` | `(s)=>s.DELIVERY_LOST>=3` |
| `BOSS_PATIENCE<=0` | `(s)=>s.BOSS_PATIENCE<=0` |
| `handled>=30` | `(s,h)=>h>=30` |
| `escapeType==='silent'` | `(s,h,r,_,et)=>et==='silent'` |
| `INFO_LEAK>=3 且 SPAM_SUBSCRIBED>=5` | `(s)=>s.INFO_LEAK>=3&&s.SPAM_SUBSCRIBED>=5` |
| 兜底（总是触发） | `()=>true` |

函数签名：`(stats, handled, replied, replyLog, escapeType) => boolean`

---

## 待审区

<!-- 程序导入完成后在此确认：已导入 N 条，来源：endings-draft.xlsx -->

---

## 当前已配置结局数

| 故事线 | 数量 | 状态 |
|--------|------|------|
| MAIN | 10 | ✅ 已配置 |
| SCAM_TARGET | 4 | ✅ 已配置 |
| DATA_AUCTION | 4 | ✅ 已配置 |
| SPACETIME_RADIO | 5 | ✅ 已配置 |
| 其余故事线 | — | ⏳ 待填写 |
