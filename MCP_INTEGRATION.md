# MCP 集成说明 · 薄香肌俱乐部

本项目为 **MCP 客户端侧 Skill**：不实现新的 MCP Server，全部能力来自麦当劳官方 MCP
（`https://mcp.mcd.cn`，Streamable HTTP，`Authorization: Bearer <token>`，限流 600 次/分钟/Token）。

## 使用的 MCP Server

| Server | 地址 | 用途 |
|---|---|---|
| mcd-mcp（麦当劳官方） | https://mcp.mcd.cn | 全部业务数据：营养、门店、菜单、价格、券、订单、活动、积分 |

## 工具调用映射

### 核心链路（膳食修行部 · 配餐）

| 步骤 | 工具 | 说明 |
|---|---|---|
| 1. 时间校准 | `now-time-info` | 判定当前餐段（早餐/正餐/深夜审判触发） |
| 2. 营养库 | `list-nutrition-foods` | 约 160 个餐品的热量/蛋白质/脂肪/碳水/钠/钙，配餐求解的数据底座 |
| 3a. 门店（到店） | `query-nearby-stores` | `searchType=2` 按位置搜索，`beType` 1=到店取餐 / 5=得来速；返回 storeCode/beCode/预约时段 |
| 3b. 门店（外送） | `delivery-query-addresses` → `delivery-query-stores` | 外送场景，`beType=2` |
| 4. 实时菜单 | `query-meals` | 该店当前可售餐品；与营养库做交集验证，剔除售罄/未上市项 |
| 5. 餐品详情 | `query-meal-detail` | 套餐组成与可替换项（如换零度可乐） |
| 6. 查券 | `query-store-coupons` | 该店可用优惠券列表 |
| 7. 权威核价 | `calculate-price` | 服务端权威价格（含券、配送费），禁止客户端自行算价 |
| 8. 下单（确认后） | `create-order` | **仅在用户明确确认后调用**；返回订单与支付链接 |

关键枚举（已实测确认）：`beType` 1=到店取餐 / 2=麦乐送 / 5=得来速 / 6=企业团餐；`orderType` 1=到店 / 2=外送。

### 入会审查处（测评复盘）

| 工具 | 用途 |
|---|---|
| `order-list` | 拉取近期到店/外送订单，复盘营养结构与消费习惯，计入双轴评分 |
| `query-order` | 单笔订单详情 |
| `query-my-account` | 积分账户（可用/即将过期积分），作为"权益认知"加分项参考 |

### 认知研究院

| 工具 | 用途 |
|---|---|
| `campaign-calendar` | 当月营销活动日历，"每日认知早餐"的真实活动来源 |
| `now-time-info` | 日期种子，金句轮转 |

### 权益与券（审判与省钱加分项）

| 工具 | 用途 |
|---|---|
| `available-coupons` / `query-my-coupons` | 麦麦省可领/已有券；检测到可领未领时触发认知拷打 |
| `auto-bind-coupons` | 一键领券（**写操作，需用户确认**） |
| `query-lottery-info` / `draw-lottery` / `query-my-prizes` | 积分抽奖（**写操作，需用户确认**） |

## 典型调用流程图

```
用户："帮我配个减脂餐，700kcal，到店，王府井"
  │
  ├─ now-time-info                    → 当前餐段判定
  ├─ list-nutrition-foods             → 营养数据库（160 项）
  ├─ query-nearby-stores(searchType=2, beType=1, keyword=王府井)
  │     → storeCode / beCode / 营业时间 / 预约时段
  ├─ query-meals(storeCode, beType=1, orderType=1)
  │     → 实时菜单 ∩ 营养库 = 可推荐集合
  ├─ query-store-coupons(storeCode, beType, orderType)
  │     → 可用券列表
  ├─ calculate-price(items, 券)        → 权威核价（用券/不用券两组）
  │
  └─ 输出：餐品 + 营养汇总 + 券后价 + 店长评语
       └─ 用户确认 → create-order（否则流程结束）
```

## 业务价值

1. **把"健康焦虑"变成"可执行的下一餐"**：营养数据与实时菜单交叉验证，推荐即可买，买了即修行。
2. **决策提效**：做题家检测机制限制无限对比，把"吃什么"的决策时间压到秒级。
3. **券价值发现**：`available-coupons` + `query-store-coupons` 联动，让用户账户里沉睡的券被用起来。
4. **活动触达**：`campaign-calendar` 让官方活动自然嵌入每日对话场景。

## 实测注意事项（开发走查记录）

1. **价格单位为分**：`calculate-price` 返回的 price/subtotal 均为分（如 3850 = ¥38.50），展示时除以 100。
2. **门店打烊**：非营业时段调用 `query-meals` 返回 `code 600057`；可携带 `reservationDate`（格式 `yyyy-MM-dd HH:mm`，须落在门店 `reservationTimeOptions` 时段内）以预约单模式拉取菜单。
3. **SSE 响应**：长载荷拆分为多条 `data:` 行，拼接时直接相连（勿插入换行），否则 JSON 损坏。
4. **响应格式**：多数工具返回"字段说明 + ## Original Response + JSON"混合格式，程序化处理时从 `## Original Response` 标记后提取 JSON。
5. **名称对齐**：菜单名与营养库名不完全一致（实测 103 项菜单仅 18 项精确同名），需归一化匹配，策略见 references/nutrition-guide.md。

## 安全与合规

- 写操作（下单/领券/抽奖/取消/建地址）一律二次确认，调用前复述影响。
- 仓库不含真实 Token，配置仅环境变量占位符。
- 不缓存用户订单/积分等个人数据，全部实时调用、即用即弃。
