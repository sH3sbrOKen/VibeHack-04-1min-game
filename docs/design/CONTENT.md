# CONTENT — 信息卡数据库

> 上下文入口：[README.md](../../README.md)
>
> **数据一致性声明**：本文件为设计稿。运行时数据以 `index.html` 中的 `RAW` 数组为准。新卡流程：先写本文件 → 确认后同步到 RAW 数组。

---

## 〇、信息卡来源

游戏中的消息分两类来源：

### 手写卡（当前26张，目标40+）
由设计者逐条编写，有精确的正文、StatEffect 和陷阱/彩蛋标记。收录在本文件中，同步到 `RAW` 数组。手写卡是内容核心——陷阱卡、彩蛋卡、剧情关键卡必须手写。

### 程序化生成卡
由代码在运行时基于模板批量生成，补足至99+条。生成规则：
- 从 SenderType 注册表中随机选取发信人类型
- 从预设的正文模板库中选取模板，填入随机变量（姓名、数字、时间等）
- correctReply 由现有 BodyOverrideRule 引擎自动推导
- StatEffect 按 SenderType 的默认规则分配（不含手写卡的特殊数值）
- difficulty 根据当前 timeLeft 分配（早期 d1，中期 d2，后期 d3）
- **不生成**陷阱卡、彩蛋卡——这些只来自手写

> 程序化生成卡的模板和变量池定义在 index.html 的 `<script>` 中，不在本文件维护。

---

## 一、数据结构定义

### MessageCard（信息卡）

```
MessageCard {
  id          : 唯一编号（如 MSG_001）
  sender      : SenderType（发信人类型）
  body        : string（正文内容）
  correctReply: ReplyType（正确答案，由规则引擎推导，不硬编码）
  isTrap      : boolean（是否为陷阱卡）
  wrongEffect : StatEffect[]（答错触发的数值变化）
  difficulty  : 1 | 2 | 3
}
```

### SenderType（发信人类型注册表）

| SenderType | 显示名示例 | 默认正确回复 |
|-----------|---------|-----------|
| BOSS | 老板、王总、HR | RECEIVED |
| DELIVERY | 外卖员、骑手、饿了么 | WAIT |
| COURIER | 快递员、驿站、顺丰 | WAIT |
| PARTNER | 对象名字、宝、亲爱的 | LOVE |
| FAMILY | 妈、爸、姑姑、奶奶 | LOVE |
| FRIEND | 朋友名字、同学 | LOVE |
| SYSTEM | 招商银行、支付宝、系统 | RECEIVED |
| SPAM | 陌生号码、400开头、英文号 | UNSUBSCRIBE |
| SUBSCRIPTION | 爱奇艺、健身APP、XX会员 | UNSUBSCRIBE |

### ReplyType（回复类型）

| ReplyType | 按钮文字 | 位置 |
|----------|--------|-----|
| WAIT | 等等 | 左上 |
| RECEIVED | 收到 | 右上 |
| UNSUBSCRIBE | 退订 | 左下 |
| LOVE | 好的亲 | 右下 |

### StatEffect

```
StatEffect {
  stat  : StatKey
  delta : number（正数增加，负数减少）
}
```

---

## 二、正确答案推导规则

### 推导逻辑

```
function getCorrectReply(card):
  1. 遍历 BodyOverrideRule，检查 body 是否匹配
  2. 若匹配，返回 override 的 ReplyType
  3. 若无匹配，返回 card.sender.defaultReply
```

> **原则**：正确答案永远不要硬编码在消息文本里。程序读取 sender 的 defaultReply，再检查 BodyOverrideRule，有匹配则覆盖。新增消息只需填写 sender 和 body，规则引擎自动推导答案。

### BodyOverrideRule 列表

| 规则ID | 触发条件（正文含有） | 覆盖为 | 典型场景 |
|-------|----------------|------|--------|
| BODY_DELIVERED | "放你门口" / "已放前台" / "已投递" | RECEIVED | 快递已放，不需要等 |
| BODY_PHISHING | "点击链接" + "奖励/退款/验证" | UNSUBSCRIBE | 诈骗链接，发信人伪装成银行 |
| BODY_OTP_UNSOLICITED | "验证码" + 发信人为 SPAM | UNSUBSCRIBE | 诈骗验证码 |
| BODY_DEDUCT | "已自动续费" / "已扣费" | RECEIVED | 扣费通知，应知悉而非退订 |
| BODY_BOSS_WARM | 发信人为 BOSS + "辛苦了" / "没事" | RECEIVED | 老板温情消息，不能回好的亲 |
| BODY_FAMILY_MONEY | 发信人为 FAMILY + "转给你" / "够不够用" | RECEIVED | 家人给钱，应回收到 |

---

## 三、信息卡数据池（第一关，共42条）

### BOSS 组

```
[MSG_001]
sender: BOSS / body: 「今天的日报发了吗，我没收到」
correctReply: RECEIVED（sender默认）
isTrap: false / wrongEffect: BOSS_PATIENCE -15 / difficulty: 1

[MSG_002]
sender: BOSS / body: 「明天早会9点，准时」
correctReply: RECEIVED（sender默认）
isTrap: false / wrongEffect: BOSS_PATIENCE -15 / difficulty: 1

[MSG_003]
sender: BOSS / body: 「方案改好了吗，客户在催」
correctReply: RECEIVED（sender默认）
isTrap: false / wrongEffect: BOSS_PATIENCE -20 / difficulty: 1

[MSG_004]
sender: BOSS / body: 「辛苦了，好好休息」
correctReply: RECEIVED（body覆盖 BODY_BOSS_WARM）
isTrap: true（看起来应该回好的亲）/ wrongEffect: BOSS_PATIENCE -30 / difficulty: 2
```

---

### DELIVERY 组

```
[MSG_005]
sender: DELIVERY / body: 「您好！外卖到了，麻烦帮忙开一下门禁谢谢」
correctReply: WAIT（sender默认）
isTrap: false / wrongEffect: DELIVERY_LOST +1 / difficulty: 1

[MSG_006]
sender: DELIVERY / body: 「哥们我在楼下，开个门」
correctReply: WAIT（sender默认）
isTrap: false / wrongEffect: DELIVERY_LOST +1 / difficulty: 1

[MSG_007]
sender: DELIVERY / body: 「您的餐我放门口了，注意查收」
correctReply: RECEIVED（body覆盖 BODY_DELIVERED）
isTrap: true（已放不需要等）/ wrongEffect: DELIVERY_LOST +1 / difficulty: 2
```

---

### COURIER 组

```
[MSG_008]
sender: COURIER / body: 「您有件快递，方便下来签一下吗」
correctReply: WAIT（sender默认）
isTrap: false / wrongEffect: DELIVERY_LOST +1 / difficulty: 1

[MSG_009]
sender: COURIER / body: 「您的包裹已放前台，记得取，有效期3天」
correctReply: RECEIVED（body覆盖 BODY_DELIVERED）
isTrap: true（已放不需要等）/ wrongEffect: DELIVERY_LOST +1 / difficulty: 2
```

---

### SPAM 组

```
[MSG_010]
sender: SPAM / body: 「【贷款】您的信用额度已提升至20万，点击领取」
correctReply: UNSUBSCRIBE（sender默认）
isTrap: false / wrongEffect: INFO_LEAK +1 / difficulty: 1

[MSG_011]
sender: SPAM / body: 「恭喜您获得本月幸运用户资格，奖励待领取」
correctReply: UNSUBSCRIBE（sender默认）
isTrap: false / wrongEffect: INFO_LEAK +1 / difficulty: 1

[MSG_012]
sender: SPAM（伪装名「【招商银行】」）/ body: 「您尾号XXXX的卡出现异常，请立即点此链接验证」
correctReply: UNSUBSCRIBE（body覆盖 BODY_PHISHING）
isTrap: true（发信人看起来像银行）/ wrongEffect: INFO_LEAK +2 / difficulty: 3

[MSG_013]
sender: SPAM / body: 「验证码：847291，请勿告知他人」
correctReply: UNSUBSCRIBE（body覆盖 BODY_OTP_UNSOLICITED）
isTrap: true（验证码看起来正规）/ wrongEffect: INFO_LEAK +2 / difficulty: 3
```

---

### SUBSCRIPTION 组

```
[MSG_014]
sender: SUBSCRIPTION（爱奇艺）/ body: 「您的会员将于3天后到期，续费享6折」
correctReply: UNSUBSCRIBE（sender默认）
isTrap: false / wrongEffect: UNEXPECTED_CHARGE +12 / difficulty: 1

[MSG_015]
sender: SUBSCRIPTION（XX健身APP）/ body: 「您已自动续费12元，感谢支持」
correctReply: RECEIVED（body覆盖 BODY_DEDUCT）
isTrap: true（扣费通知应知悉，不是退订）/ wrongEffect: UNEXPECTED_CHARGE +12 / difficulty: 2
```

---

### SYSTEM 组

```
[MSG_016]
sender: SYSTEM（招商银行）/ body: 「您的账户消费158.00元，余额XXX元」
correctReply: RECEIVED（sender默认）
isTrap: false / wrongEffect: UNEXPECTED_CHARGE +10 / difficulty: 1

[MSG_017]
sender: SYSTEM（饿了么）/ body: 「您的订单已出餐，预计22分钟送达」
correctReply: RECEIVED（sender默认）
isTrap: false / wrongEffect: DELIVERY_LOST +1 / difficulty: 1
```

---

### PARTNER 组

```
[MSG_018]
sender: PARTNER / body: 「你昨晚睡哪了」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: INTIMACY -20 / difficulty: 1

[MSG_019]
sender: PARTNER / body: 「你怎么还不回我」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: INTIMACY -30 / difficulty: 2

[MSG_020]
sender: PARTNER / body: 「你手机是不是没电了哈哈」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: 无（彩蛋：任意回复触发对象会心一笑提示）/ difficulty: 1
```

---

### FAMILY 组

```
[MSG_021]
sender: FAMILY（妈）/ body: 「吃早饭了吗」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: INTIMACY -10 / difficulty: 1

[MSG_022]
sender: FAMILY（妈）/ body: 「钱够不够，不够说一声」
correctReply: RECEIVED（body覆盖 BODY_FAMILY_MONEY）
isTrap: true（家人给钱应回收到）/ wrongEffect: INTIMACY -15 / difficulty: 2

[MSG_023]
sender: FAMILY（爸）/ body: 「今年回来过节不」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: INTIMACY -10 / difficulty: 1
```

---

### FRIEND 组

```
[MSG_024]
sender: FRIEND / body: 「上次说要约的饭还记得吗」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: INTIMACY -5 / difficulty: 1

[MSG_025]
sender: FRIEND / body: 「我把你拉进一个群了你看下」
correctReply: LOVE（sender默认）
isTrap: false / wrongEffect: INTIMACY -5 / difficulty: 1
```

---

### 群消息组

```
[MSG_026]
sender: BOSS（群消息 @你）/ body: 「@你 周五的PPT你来准备一下」
correctReply: RECEIVED（sender默认）
isTrap: false / wrongEffect: BOSS_PATIENCE -20 / difficulty: 1
```

---

## 四、陷阱卡速查表

| ID | 发信人 | 正文关键词 | 看起来应该回 | 实际正确回 | 触发数值 |
|----|------|---------|-----------|---------|--------|
| MSG_004 | BOSS | 辛苦了 | LOVE | RECEIVED | BOSS_PATIENCE -30 |
| MSG_007 | DELIVERY | 放门口了 | WAIT | RECEIVED | DELIVERY_LOST +1 |
| MSG_009 | COURIER | 已放前台 | WAIT | RECEIVED | DELIVERY_LOST +1 |
| MSG_012 | SPAM伪装SYSTEM | 点此链接验证 | RECEIVED | UNSUBSCRIBE | INFO_LEAK +2 |
| MSG_013 | SPAM | 验证码 | RECEIVED | UNSUBSCRIBE | INFO_LEAK +2 |
| MSG_015 | SUBSCRIPTION | 已自动续费 | UNSUBSCRIBE | RECEIVED | UNEXPECTED_CHARGE +12 |
| MSG_022 | FAMILY | 够不够用 | LOVE | RECEIVED | INTIMACY -15 |

---

## 五、数值注册表

| StatKey | 显示名 | 方向 | 初始值 | 满值阈值 | 解锁支线 |
|--------|------|-----|------|--------|--------|
| DELIVERY_LOST | 外卖丢失率 | ↑坏 | 0 | 3 | STORYLINE_RIDER |
| BOSS_PATIENCE | 老板耐心值 | ↓坏 | 100 | 0 | STORYLINE_CRISIS |
| INFO_LEAK | 个人信息泄露指数 | ↑坏 | 0 | 3 | STORYLINE_SALES |
| INTIMACY | 亲密度 | ↓坏 | 100 | 0 | STORYLINE_BREAKUP |
| SPAM_SUBSCRIBED | 骚扰订阅量 | ↑坏 | 0 | 5 | STORYLINE_OPERATOR |
| UNEXPECTED_CHARGE | 意外扣费金额 | ↑坏 | 0 | 200 | STORYLINE_SUPPORT |

---

## 六、方案B — 变体卡设计稿（待审核）

> 每个 SenderType 最多 3 组种子卡 × 2 变体。只从非陷阱、非彩蛋的手写卡派生。
> 变体卡继承种子卡的 SenderType 和默认 StatEffect，difficulty 统一为种子卡同级。
> correctReply 由规则引擎自动推导，不硬编码。

### BOSS（3组 × 2变体 = 6张）

种子 MSG_001「今天的日报发了吗，我没收到」
- 「周报发了没，催了你好几次了」
- 「上次让你整理的那个资料呢」

种子 MSG_002「明天早会9点，准时」
- 「下午三点开会，别迟到」
- 「周五下午复盘，准时参加」

种子 MSG_003「方案改好了吗，客户在催」
- 「报价单好了吗，那边在等」
- 「昨天说的排期今天能出吗」

### DELIVERY（2组 × 2变体 = 4张）

种子 MSG_005「您好！外卖到了，麻烦帮忙开一下门禁谢谢」
- 「到了到了，帮忙开下楼下门」
- 「你好在单元门口了，开下门禁」

种子 MSG_006「哥们我在楼下，开个门」
- 「师傅在大堂了，下来拿一下呗」
- 「到你楼下了催催催快点要超时了」

### COURIER（1组 × 2变体 = 2张）

种子 MSG_008「您有件快递，方便下来签一下吗」
- 「有你一个包裹，今天能来取吗」
- 「快递到了，方便什么时候过来签收」

### SPAM（2组 × 2变体 = 4张）

种子 MSG_010「【贷款】您的信用额度已提升至20万，点击领取」
- 「【理财】恭喜获得专属体验金50000元」
- 「【保险】您有一份免费保障待领取」

种子 MSG_011「恭喜您获得本月幸运用户资格，奖励待领取」
- 「您的购物券即将过期，点击使用」
- 「恭喜中奖，回复TD退订」

### SUBSCRIPTION（1组 × 2变体 = 2张）

种子 MSG_014「您的会员将于3天后到期，续费享6折」
- 「您的月度会员明天到期，续费提醒」
- 「尊敬的用户，您的套餐流量即将用尽」

### SYSTEM（2组 × 2变体 = 4张）

种子 MSG_016「您的账户消费158.00元」
- 「您尾号3721的卡支出68.00元」
- 「信用卡本期账单已出，请及时还款」

种子 MSG_017「您的订单已出餐，预计22分钟送达」
- 「您的快递正在派送中，请保持电话畅通」
- 「您的订单已签收，请确认」

### PARTNER（2组 × 2变体 = 4张）

种子 MSG_018「你昨晚睡哪了」
- 「你昨天是不是没回家」
- 「怎么这个点才起」

种子 MSG_019「你怎么还不回我」
- 「又不看手机」
- 「你是不是在忙」

### FAMILY（2组 × 2变体 = 4张）

种子 MSG_021「吃早饭了吗」
- 「中午吃的什么」
- 「晚上记得按时吃饭」

种子 MSG_023「今年回来过节不」
- 「暑假有空回来吗」
- 「国庆能回来几天」

### FRIEND（2组 × 2变体 = 4张）

种子 MSG_024「上次说要约的饭还记得吗」
- 「周末有空吗出来坐坐」
- 「好久没见了约一下」

种子 MSG_025「我把你拉进一个群了你看下」
- 「有个活动群你要不要进」
- 「给你推了张名片你加一下」

### 方案B汇总

| SenderType | 种子数 | 变体数 | 小计 |
|-----------|-------|-------|-----|
| BOSS | 3 | 6 | 6 |
| DELIVERY | 2 | 4 | 4 |
| COURIER | 1 | 2 | 2 |
| SPAM | 2 | 4 | 4 |
| SUBSCRIPTION | 1 | 2 | 2 |
| SYSTEM | 2 | 4 | 4 |
| PARTNER | 2 | 4 | 4 |
| FAMILY | 2 | 4 | 4 |
| FRIEND | 2 | 4 | 4 |
| **合计** | **17** | **34** | **34** |

手写26 + 变体34 = **60张确定内容卡**。剩余由方案A模板生成补足至 99+。

---

## 七、方案A — 模板生成设计稿（待审核）

> 运行时从模板库随机选取模板，填入变量池中的随机值，生成新消息。
> correctReply 由 BodyOverrideRule 引擎自动推导。
> StatEffect 按 SenderType 默认规则分配（见下方默认表）。
> 不生成陷阱卡和彩蛋卡。

### SenderType 默认 StatEffect

| SenderType | 默认 StatEffect | 说明 |
|-----------|----------------|------|
| BOSS | BOSS_PATIENCE -15 | 没及时回老板 |
| DELIVERY | DELIVERY_LOST +1 | 外卖没等到 |
| COURIER | DELIVERY_LOST +1 | 快递没取到 |
| SPAM | INFO_LEAK +1 | 没退订骚扰 |
| SUBSCRIPTION | UNEXPECTED_CHARGE +12 | 没退订扣费 |
| SYSTEM | UNEXPECTED_CHARGE +10 | 没确认通知 |
| PARTNER | INTIMACY -10 | 没好好回对象 |
| FAMILY | INTIMACY -10 | 没好好回家人 |
| FRIEND | INTIMACY -5 | 没好好回朋友 |

### 模板库

每条模板格式：`正文模板 | 变量池`

**BOSS**（4条模板）
- 「{time}的{event}你准备好了吗」| time=[明天, 下周一, 周五] event=[汇报, 述职, 评审]
- 「{person}问你那个{thing}什么时候给」| person=[客户, 市场部, 那边] thing=[方案, 数据, 报告]
- 「你的{thing}我看了，重新改下」| thing=[周报, 邮件, PPT, 需求文档]
- 「{event}定了，{time}之前发给我」| event=[排期, 进度表, 预算] time=[今天下班前, 明天中午, 这周五]

**DELIVERY**（3条模板）
- 「你的{food}快到了，还有{min}分钟」| food=[咖啡, 便当, 奶茶, 沙拉] min=[5, 8, 12, 15]
- 「{greeting}，{item}到了，在{location}等你」| greeting=[你好, 亲] item=[外卖, 餐] location=[楼下, 小区门口, 大堂]
- 「麻烦快点下来，还有好几单要送」|（无变量，固定文本）

**COURIER**（3条模板）
- 「您有{count}个快递到了，{action}」| count=[一, 两] action=[方便过来取下吗, 今天能来吗]
- 「快递取件码{code}，在{location}」| code=[5823, 3947, 0816, 2741] location=[驿站, 快递柜, 门卫处]
- 「{name}，顺丰到付件，{amount}元」| name=[先生, 女士] amount=[18, 23, 45]

**SPAM**（4条模板）
- 「【{brand}】限时福利，{offer}」| brand=[XX商城, 拼多多, 京东] offer=[满100减50, 免费领取, 独家优惠]
- 「您有一笔{amount}元待退款，点击处理」| amount=[358, 129, 88]
- 「{org}提醒您，您的{thing}即将过期」| org=[社保局, 公积金中心, 车管所] thing=[医保, 驾照, 社保卡]
- 「回复1领取福利，回复TD退订」|（无变量）

**SUBSCRIPTION**（3条模板）
- 「【{service}】您的{plan}即将到期」| service=[视频VIP, 音乐会员, 云空间, 读书会员] plan=[月度会员, 年度套餐, 试用期]
- 「尊敬的用户，{notif}」| notif=[新功能已上线, 系统升级中, 积分即将清零]
- 「{service}邀请您参加会员日活动」| service=[美团, 饿了么, 淘宝, 京东]

**SYSTEM**（3条模板）
- 「您{action}{amount}元」| action=[消费, 收到转账, 充值] amount=[29.9, 66, 128, 199]
- 「{service}：{content}」| service=[支付宝, 微信支付, 银行] content=[还款提醒, 交易成功, 限额调整通知]
- 「您的{thing}已到账」| thing=[工资, 退款, 红包, 奖金]

**PARTNER**（4条模板）
- 「你{question}」| question=[在哪呢, 到家了没, 忙什么呢, 吃了吗]
- 「{mood}」| mood=[想你了, 无聊, 哼, 晚安]
- 「我跟你说个事」|（无变量）
- 「你看你朋友圈{time}那条」| time=[昨天发的, 上周, 前天]

**FAMILY**（4条模板）
- 「{relative}说{content}」| relative=[你姑, 你姨, 你舅, 你奶奶] content=[好久没见你了, 有空来家里吃饭, 让你打个电话]
- 「{question}」| question=[最近忙吗, 身体还好吧, 天冷了多穿点, 工作还顺利吗]
- 「给你转了{amount}块，{reason}」| amount=[200, 500, 1000] reason=[买点水果, 别省着, 过节快乐]
- 「听说{news}」| news=[你换工作了, 你搬家了, 你升职了, 你谈对象了]

**FRIEND**（4条模板）
- 「有个{thing}你感兴趣吗」| thing=[展, 活动, 比赛, 讲座]
- 「{request}」| request=[帮我砍一刀, 投个票, 帮忙点个赞, 帮我看看这个]
- 「你看{thing}了吗」| thing=[今天的热搜, 那个新闻, 朋友圈那条, 群里发的]
- 「{time}有空吗」| time=[今晚, 这周末, 下周, 明天中午]

### 方案A生成量估算

| SenderType | 模板数 | 变量组合数（估） | 说明 |
|-----------|-------|-------------|------|
| BOSS | 4 | ~36 | 3×3 + 3×3 + 4 + 3×3 |
| DELIVERY | 3 | ~25 | 4×4 + 2×2×3 + 1 |
| COURIER | 3 | ~14 | 2×2 + 4×3 + 2×3 |
| SPAM | 4 | ~16 | 3×3 + 3 + 3×3 + 1 |
| SUBSCRIPTION | 3 | ~13 | 4×4 + 3 + 4 |
| SYSTEM | 3 | ~18 | 3×4 + 3×3 + 4 |
| PARTNER | 4 | ~12 | 4 + 4 + 1 + 3 |
| FAMILY | 4 | ~22 | 4×4 + 4 + 3×3 + 4 |
| FRIEND | 4 | ~14 | 4 + 4 + 4 + 4 |
| **合计** | **32** | **~170** | 远超99+所需 |

### 总内容量汇总

| 来源 | 数量 | 特点 |
|------|------|------|
| 手写卡 | 26 | 陷阱卡、彩蛋卡、核心叙事 |
| B变体 | 34 | 基于手写卡换皮，质量可控 |
| A生成 | ~40（运行时按需） | 模板填槽，充当背景噪音 |
| **总计** | **~100** | 覆盖99+需求 |

> 运行时生成器逻辑：优先弹出手写卡和B变体（确定内容），穿插A生成卡（填充密度）。
> 手写卡/B变体耗尽后，全部由A生成卡填充直到60秒结束。

---

## 八、待补充

- 各支线关卡专属信息卡池（每关约20条）
- 方案A模板库可持续扩展，每个 SenderType 随时可加新模板
