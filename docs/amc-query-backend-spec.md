# AMC Audience Query Builder — 后端交付规格说明书

> 本文档面向后端开发，定义 Query 模板、全部参数、枚举值及其对应的 SQL 替换片段。
> 后端只需按此规格实现"参数 → SQL片段 替换 → 拼接"即可。

---

## 0. 文档阅读指南

本文档结构如下：

```
第1章  全局架构          ← 整条SQL的骨架、模块间如何拼接
第2章  参数总表          ← 所有参数一览，一眼看全
第3章  CTE SQL模板       ← 两种模板(广告行为/自然行为)的完整SQL
第4章  参数明细          ← 每个参数的枚举值 → SQL替换值 映射表
第5章  完整示例          ← 3个端到端的用户操作 → 最终SQL 案例
```

---

## 1. 全局 SQL 架构

### 1.1 最终输出的 SQL 结构

无论用户配置了几个行为模块，最终输出的 SQL **始终**是以下结构：

```sql
WITH
  seg_1 AS ( {行为模块1的CTE} ),
  seg_2 AS ( {行为模块2的CTE} ),
  ...
  seg_N AS ( {行为模块N的CTE} )
SELECT user_id FROM seg_1
{集合运算符1}
SELECT user_id FROM seg_2
{集合运算符2}
...
SELECT user_id FROM seg_N
```

### 1.2 集合运算符 — 参数 `${SET_OP}`

模块之间的组合关系，从左到右依次计算：

| 前端显示 | 枚举值 | SQL 替换值 | 语义 |
|---------|--------|-----------|------|
| 且（交集） | `AND` | `INTERSECT` | 同时满足两个行为 |
| 或（并集） | `OR` | `UNION` | 满足任一行为 |
| 非（差集） | `NOT` | `EXCEPT` | 满足前者但排除后者 |

### 1.3 单模块时的退化

如果只有1个行为模块，没有集合运算，SQL退化为：

```sql
WITH seg_1 AS ( {CTE} )
SELECT user_id FROM seg_1
```

---

## 2. 参数总表

### 2.1 模块级参数（每个行为模块各自拥有）

| # | 参数名 | 参数ID | 类型 | 适用范围 | 说明 |
|---|--------|--------|------|---------|------|
| 1 | 行为事件 | `${EVENT_TYPE}` | 单选枚举 | 全部模块 | 决定使用哪个SQL模板 + 哪些后续参数可用 |
| 2 | 广告类型 | `${AD_TYPE}` | 多选枚举 | 广告行为类 | 决定查询DSP表/SA表/Both |
| 3 | Advertiser | `${ADV}` | 单选/输入 | 广告行为类(DSP部分) | DSP advertiser过滤 |
| 4 | Entity | `${ENTITY}` | 单选/输入 | 广告行为类(SA部分) | SA entity过滤 |
| 5 | Campaign | `${CAMPAIGN}` | 多值输入 | 广告行为类 | Campaign名称列表 |
| 6 | 行为时间 | `${TIME_FILTER}` | 日期范围 | 全部模块 | 固定日期段 或 相对天数 |
| 7 | 行为属性类型 | `${ATTR_TYPE}` | 单选枚举 | 自然行为类 | 筛选哪个属性列 |
| 8 | 属性匹配方式 | `${MATCH_TYPE}` | 单选枚举 | 自然行为类 | 精准/模糊匹配 |
| 9 | 属性值 | `${ATTR_VALUES}` | 多值输入 | 自然行为类 | 用户输入的属性值列表 |
| 10 | 统计指标 | `${METRIC_TYPE}` | 单选枚举 | 全部模块 | 次数/金额 |
| 11 | 统计条件 | `${METRIC_OP}` | 单选枚举 | 全部模块 | 比较运算符 |
| 12 | 统计值 | `${METRIC_VAL}` | 数值输入 | 全部模块 | 用户填写的数值 |

### 2.2 全局参数

| 参数名 | 参数ID | 说明 |
|--------|--------|------|
| 模块间集合运算 | `${SET_OP}` | 每两个模块之间有一个，共 N-1 个 |

---

## 3. CTE SQL 模板

根据 `${EVENT_TYPE}` 所属类别，使用不同的 CTE 模板。

### 3.1 模板A — 广告行为类（曝光 / 有效曝光 / 点击）

```sql
seg_{N} AS (
  SELECT user_id
  FROM (
    -- ▼ DSP数据块（当选中了DSP类广告类型时包含此块）
    SELECT user_id, ${METRIC_COL} AS metric_value
    FROM ${TABLE_DSP}
    WHERE user_id_type = 'adUserId'
      AND ${METRIC_COL} > 0
      ${TIME_FILTER}
      ${ADV}
      ${CAMPAIGN}

    UNION ALL

    -- ▼ SA数据块（当选中了SA类广告类型时包含此块）
    SELECT user_id, ${METRIC_COL} AS metric_value
    FROM ${TABLE_SA}
    WHERE user_id_type = 'adUserId'
      AND ${METRIC_COL} > 0
      ${TIME_FILTER}
      ${ENTITY}
      ${CAMPAIGN}
  )
  GROUP BY 1
  HAVING ${HAVING_CLAUSE}
)
```

**模板A 动态裁剪规则：**

| 用户选择的广告类型 | 包含的数据块 |
|------------------|-------------|
| 仅DSP类 (PVA, OLV) | 只保留 DSP数据块，删除 `UNION ALL` + SA数据块 |
| 仅SA类 (SP, SB, SD) | 只保留 SA数据块，删除 DSP数据块 + `UNION ALL` |
| 混合 (如 PVA + SP) | 保留完整 UNION ALL |

### 3.2 模板B — 自然行为类（搜索 / 详情页浏览 / 加购 / 心愿单 / 购买）

```sql
seg_{N} AS (
  SELECT user_id
  FROM ${TABLE}
  WHERE user_id_type = 'adUserId'
    ${TIME_FILTER}
    ${ATTR_FILTER}
  GROUP BY 1
  HAVING ${HAVING_CLAUSE}
)
```

---

## 4. 参数明细 — 枚举值与 SQL 替换映射

### 4.1 `${EVENT_TYPE}` — 行为事件

决定性参数，所有后续参数的可用性取决于此。

| 前端显示 | 枚举值 | SQL模板 | 后续可用参数 |
|---------|--------|---------|-------------|
| 曝光 | `impression` | 模板A | AD_TYPE, ADV, ENTITY, CAMPAIGN, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 有效曝光 | `viewable_impression` | 模板A | AD_TYPE, ADV, ENTITY, CAMPAIGN, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 点击 | `click` | 模板A | AD_TYPE, ADV, ENTITY, CAMPAIGN, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 搜索 | `search` | 模板B | ATTR_TYPE, MATCH_TYPE, ATTR_VALUES, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 详情页浏览 | `detail_page_view` | 模板B | ATTR_TYPE, MATCH_TYPE, ATTR_VALUES, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 购物车加购 | `add_to_cart` | 模板B | ATTR_TYPE, MATCH_TYPE, ATTR_VALUES, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 心愿单加购 | `add_to_wishlist` | 模板B | ATTR_TYPE, MATCH_TYPE, ATTR_VALUES, TIME_FILTER, METRIC_OP, METRIC_VAL |
| 购买 | `purchase` | 模板B | ATTR_TYPE, MATCH_TYPE, ATTR_VALUES, TIME_FILTER, **METRIC_TYPE**, METRIC_OP, METRIC_VAL |

> 注：只有"购买"有 METRIC_TYPE 选择（次数/金额），其余行为的统计指标固定为"次数"。

---

### 4.2 `${TABLE_DSP}` / `${TABLE_SA}` / `${TABLE}` — 数据表名（由 EVENT_TYPE 自动决定）

后端根据 EVENT_TYPE 自动映射，前端不需要传此参数。

| EVENT_TYPE | ${TABLE_DSP} | ${TABLE_SA} | ${TABLE} |
|-----------|-------------|------------|---------|
| `impression` | `dsp_impressions_for_audiences` | `sponsored_ads_traffic_for_audiences` | — |
| `viewable_impression` | `dsp_impressions_for_audiences` | `sponsored_ads_traffic_for_audiences` | — |
| `click` | `dsp_clicks_for_audiences` | `sponsored_ads_traffic_for_audiences` | — |
| `search` | — | — | `amazon_organic_search_for_audiences` |
| `detail_page_view` | — | — | `detail_page_views_for_audiences` |
| `add_to_cart` | — | — | `add_to_carts_for_audiences` |
| `add_to_wishlist` | — | — | `add_to_wishlists_for_audiences` |
| `purchase` | — | — | `conversions_all_for_audiences` |

---

### 4.3 `${METRIC_COL}` — 指标列名（由 EVENT_TYPE 自动决定）

| EVENT_TYPE | ${METRIC_COL} | 说明 |
|-----------|--------------|------|
| `impression` | `impressions` | SUM(impressions) |
| `viewable_impression` | `viewable_impressions` | SUM(viewable_impressions) |
| `click` | `clicks` | SUM(clicks) |

> 自然行为类不使用 METRIC_COL，统一用 COUNT(1) 或 SUM(total_product_sales)。

---

### 4.4 `${AD_TYPE}` — 广告类型（多选）

仅广告行为类可用。用户可多选，选择结果决定 UNION ALL 结构。

| 前端显示 | 枚举值 | 所属广告来源 | 影响 |
|---------|--------|------------|------|
| PVA (Programmatic Video Ads) | `PVA` | DSP | 包含 DSP 数据块 |
| OLV (Online Video) | `OLV` | DSP | 包含 DSP 数据块 |
| SP (Sponsored Products) | `SP` | SA | 包含 SA 数据块 |
| SB (Sponsored Brands) | `SB` | SA | 包含 SA 数据块 |
| SD (Sponsored Display) | `SD` | SA | 包含 SA 数据块 |

**SQL 影响逻辑：**

```
选中的AD_TYPE列表
    │
    ├── 包含任一 DSP类 (PVA/OLV)  →  needDSP = true  →  包含DSP数据块
    ├── 包含任一 SA类 (SP/SB/SD)  →  needSA  = true  →  包含SA数据块
    │
    ├── needDSP && needSA   →  DSP数据块 UNION ALL SA数据块
    ├── needDSP && !needSA  →  仅DSP数据块（无UNION ALL）
    └── !needDSP && needSA  →  仅SA数据块（无UNION ALL）
```

---

### 4.5 `${ADV}` — Advertiser 过滤（DSP数据块专用）

| 用户操作 | SQL 替换值 |
|---------|-----------|
| 未填写 | `''`（空字符串，不生成任何SQL片段） |
| 填写了 advertiser_id | `AND advertiser = '{advertiser_id}'` |

**示例：** 用户填写 `ENTITY123` → `AND advertiser = 'ENTITY123'`

---

### 4.6 `${ENTITY}` — Entity 过滤（SA数据块专用）

| 用户操作 | SQL 替换值 |
|---------|-----------|
| 未填写 | `''`（空字符串） |
| 填写了 entity_id | `AND entity = '{entity_id}'` |

---

### 4.7 `${CAMPAIGN}` — Campaign 过滤

| 用户操作 | SQL 替换值 |
|---------|-----------|
| 未选择任何campaign | `''`（空字符串，不过滤） |
| 选择了1个或多个campaign | `AND campaign IN ('{camp1}','{camp2}',...)` |

**示例：** 用户输入了3个campaign →

```sql
AND campaign IN ('O-Brand-AW-IM_Smart Home-DP-CTR','O-Brand-AW-LS_Christmas-DP-CTR','O-Brand-AW-LS_DIY-DP-CTR')
```

---

### 4.8 `${TIME_FILTER}` — 行为时间

| 用户操作 | 子类型 | SQL 替换值 |
|---------|--------|-----------|
| 选择固定时间段 | `fixed` | `AND date_partition BETWEEN '{startDate}' AND '{endDate}'` |
| 选择相对时间 (过去N天) | `relative` | `AND date_partition >= DATE_FORMAT(DATE_ADD('day', -{N}, CURRENT_DATE), '%Y%m%d')` |

**示例：**
- 固定: 2024-01-01 ~ 2024-03-31 → `AND date_partition BETWEEN '20240101' AND '20240331'`
- 相对: 过去30天 → `AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')`

> **注意：** date_partition 格式为 `yyyyMMdd`（无分隔符），需确认AMC实例的实际格式。

---

### 4.9 `${ATTR_TYPE}` — 行为属性类型（自然行为类专用）

不同的行为事件可选的属性类型不同：

| EVENT_TYPE | 可选的 ATTR_TYPE 枚举值 |
|-----------|----------------------|
| `search` | `search_term` |
| `detail_page_view` | `product_line`, `asin` |
| `add_to_cart` | `product_line`, `asin` |
| `add_to_wishlist` | `product_line`, `asin` |
| `purchase` | `product_line`, `asin` |

各 ATTR_TYPE 对应的数据库列名：

| 前端显示 | 枚举值 | 对应表列名 |
|---------|--------|----------|
| 搜索词 | `search_term` | `search_term` |
| 产品线 | `product_line` | `product_line` |
| ASIN | `asin` | `asin` |

---

### 4.10 `${MATCH_TYPE}` — 属性匹配方式

| 前端显示 | 枚举值 | 适用 ATTR_TYPE | 说明 |
|---------|--------|---------------|------|
| 精准匹配 | `exact` | 全部 | 值完全相等 |
| 模糊匹配 | `fuzzy` | 仅 `search_term` | 包含关键词 |

---

### 4.11 `${ATTR_FILTER}` — 属性筛选（由 ATTR_TYPE + MATCH_TYPE + ATTR_VALUES 组合生成）

此为组合参数。后端根据 4.9、4.10、4.12 三个参数组合生成最终SQL片段：

| ATTR_TYPE | MATCH_TYPE | 用户输入值 | SQL 替换值 |
|-----------|------------|-----------|-----------|
| `search_term` | `exact` | `['keyword1','keyword2']` | `AND search_term IN ('keyword1','keyword2')` |
| `search_term` | `fuzzy` | `['keyword1','keyword2']` | `AND (search_term LIKE '%keyword1%' OR search_term LIKE '%keyword2%')` |
| `asin` | `exact` | `['B0XXX','B0YYY']` | `AND asin IN ('B0XXX','B0YYY')` |
| `product_line` | `exact` | `['LineA','LineB']` | `AND product_line IN ('LineA','LineB')` |
| 任意 | 任意 | `[]`（空） | `''`（空字符串，不过滤） |

---

### 4.12 `${ATTR_VALUES}` — 属性值（用户多值输入）

- 类型：字符串数组
- 前端允许用户输入多个值（逗号分隔或tag输入）
- 传给后端的格式：`string[]`
- 后端在生成SQL时需对每个值加单引号包裹，并做SQL注入转义

---

### 4.13 `${METRIC_TYPE}` — 统计指标类型

| EVENT_TYPE | 前端显示 | 枚举值 | 说明 |
|-----------|---------|--------|------|
| 全部行为 | 次数 | `count` | 行为发生的次数 |
| 仅 `purchase` | 金额 | `amount` | 购买总金额 |

---

### 4.14 `${HAVING_CLAUSE}` — 统计聚合（由 METRIC_TYPE + METRIC_OP + METRIC_VAL 组合生成）

此为组合参数。后端根据行为类型和指标类型生成：

**广告行为类（模板A）：**

| METRIC_TYPE | SQL 替换值 |
|------------|-----------|
| `count` | `SUM(metric_value) ${METRIC_OP} ${METRIC_VAL}` |

> metric_value 是模板A中 SELECT 的别名，已经映射了正确的指标列。

**自然行为类（模板B）：**

| METRIC_TYPE | SQL 替换值 |
|------------|-----------|
| `count` | `COUNT(1) ${METRIC_OP} ${METRIC_VAL}` |
| `amount` (仅purchase) | `SUM(total_product_sales) ${METRIC_OP} ${METRIC_VAL}` |

---

### 4.15 `${METRIC_OP}` — 统计条件（比较运算符）

| 前端显示 | 枚举值 | SQL 替换值 |
|---------|--------|-----------|
| >= | `gte` | `>=` |
| > | `gt` | `>` |
| = | `eq` | `=` |
| < | `lt` | `<` |
| <= | `lte` | `<=` |
| != | `neq` | `!=` |

---

### 4.16 `${METRIC_VAL}` — 统计值

- 类型：正整数 或 正浮点数（金额时）
- 前端校验：> 0
- SQL替换：直接输出数值，如 `3`, `100.5`

---

## 5. 完整端到端示例

### 示例1：曝光未购买（你的原始示例）

**用户操作：**

| 配置项 | 模块1 | 模块2 |
|-------|-------|-------|
| 行为事件 | 曝光 (`impression`) | 购买 (`purchase`) |
| 广告类型 | PVA + SP (DSP+SA) | — |
| Advertiser | — | — |
| Entity | — | — |
| Campaign | 4个campaign | — |
| 时间 | 默认 | 默认 |
| 属性 | — | — |
| 统计指标 | 次数 >= 1 | 次数 >= 1 (total_product_sales > 0) |
| **集合运算** | — **NOT** → | |

**参数值：**

```
模块1:
  EVENT_TYPE   = impression
  AD_TYPE      = [PVA, SP]        → needDSP=true, needSA=true → UNION ALL
  ADV          = ''               → (空)
  ENTITY       = ''               → (空)
  CAMPAIGN     = ['O-Divoom-Digital Picture Frame-AW-IM_Smart Home-DP-CTR',
                  'O-Divoom-Digital Picture Frame-AW-LS_Christmas-DP-CTR',
                  'O-Divoom-Digital Picture Frame-AW-LS_Interested in DIY-DP-CTR',
                  'O-Divoom-Digital Picture Frame-AW-LS_Video Games-DP-CTR']
  TIME_FILTER  = ''               → (空，不限时间)
  METRIC_OP    = gte              → >=
  METRIC_VAL   = 1

模块2:
  EVENT_TYPE   = purchase
  TABLE        = conversions_all_for_audiences
  ATTR_FILTER  = ''               → (空，不限属性)
  TIME_FILTER  = ''               → (空)
  METRIC_TYPE  = count
  METRIC_OP    = gte              → (体现为 total_product_sales > 0)
  METRIC_VAL   = 1

SET_OP[0]      = NOT              → EXCEPT
```

**生成的SQL：**

```sql
WITH
seg_1 AS (
  SELECT user_id
  FROM (
    SELECT user_id, impressions AS metric_value
    FROM dsp_impressions_for_audiences
    WHERE user_id_type = 'adUserId'
      AND impressions > 0
      AND campaign IN ('O-Divoom-Digital Picture Frame-AW-IM_Smart Home-DP-CTR','O-Divoom-Digital Picture Frame-AW-LS_Christmas-DP-CTR','O-Divoom-Digital Picture Frame-AW-LS_Interested in DIY-DP-CTR','O-Divoom-Digital Picture Frame-AW-LS_Video Games-DP-CTR')
    UNION ALL
    SELECT user_id, impressions AS metric_value
    FROM sponsored_ads_traffic_for_audiences
    WHERE user_id_type = 'adUserId'
      AND impressions > 0
      AND campaign IN ('O-Divoom-Digital Picture Frame-AW-IM_Smart Home-DP-CTR','O-Divoom-Digital Picture Frame-AW-LS_Christmas-DP-CTR','O-Divoom-Digital Picture Frame-AW-LS_Interested in DIY-DP-CTR','O-Divoom-Digital Picture Frame-AW-LS_Video Games-DP-CTR')
  )
  GROUP BY 1
  HAVING SUM(metric_value) >= 1
),
seg_2 AS (
  SELECT user_id
  FROM conversions_all_for_audiences
  WHERE user_id_type = 'adUserId'
    AND total_product_sales > 0
  GROUP BY 1
)
SELECT user_id FROM seg_1
EXCEPT
SELECT user_id FROM seg_2
```

---

### 示例2：搜索过关键词 + 浏览过详情页 + 未购买

**用户操作：**

| 配置项 | 模块1 | 模块2 | 模块3 |
|-------|-------|-------|-------|
| 行为事件 | 搜索 (`search`) | 详情页浏览 (`detail_page_view`) | 购买 (`purchase`) |
| 属性类型 | 搜索词 | ASIN | — |
| 匹配方式 | 模糊匹配 | 精准匹配 | — |
| 属性值 | `['digital frame','pixel art']` | `['B0AAAAAA','B0BBBBBB']` | — |
| 时间 | 过去30天 | 过去30天 | 过去30天 |
| 统计指标 | 次数 >= 2 | 次数 >= 1 | 次数 >= 1 |
| **集合运算** | — **AND** → | — **NOT** → | |

**生成的SQL：**

```sql
WITH
seg_1 AS (
  SELECT user_id
  FROM amazon_organic_search_for_audiences
  WHERE user_id_type = 'adUserId'
    AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')
    AND (search_term LIKE '%digital frame%' OR search_term LIKE '%pixel art%')
  GROUP BY 1
  HAVING COUNT(1) >= 2
),
seg_2 AS (
  SELECT user_id
  FROM detail_page_views_for_audiences
  WHERE user_id_type = 'adUserId'
    AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')
    AND asin IN ('B0AAAAAA','B0BBBBBB')
  GROUP BY 1
  HAVING COUNT(1) >= 1
),
seg_3 AS (
  SELECT user_id
  FROM conversions_all_for_audiences
  WHERE user_id_type = 'adUserId'
    AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')
  GROUP BY 1
  HAVING COUNT(1) >= 1
)
SELECT user_id FROM seg_1
INTERSECT
SELECT user_id FROM seg_2
EXCEPT
SELECT user_id FROM seg_3
```

---

### 示例3：高价值点击用户（仅SA广告）

**用户操作：**

| 配置项 | 模块1 |
|-------|-------|
| 行为事件 | 点击 (`click`) |
| 广告类型 | SP + SB (仅SA) |
| Campaign | `['SP-Brand-Auto','SB-Brand-Video']` |
| 时间 | 2024-01-01 ~ 2024-06-30 |
| 统计指标 | 次数 >= 5 |

**参数 → SQL 映射过程：**

```
EVENT_TYPE = click
  → 模板A
  → TABLE_SA = sponsored_ads_traffic_for_audiences
  → METRIC_COL = clicks

AD_TYPE = [SP, SB]
  → 全部属于SA → needDSP=false, needSA=true
  → 裁剪DSP数据块，仅保留SA数据块（无UNION ALL）

CAMPAIGN = ['SP-Brand-Auto','SB-Brand-Video']
  → AND campaign IN ('SP-Brand-Auto','SB-Brand-Video')

TIME_FILTER = fixed, 2024-01-01 ~ 2024-06-30
  → AND date_partition BETWEEN '20240101' AND '20240630'

HAVING = count, gte, 5
  → SUM(metric_value) >= 5
```

**生成的SQL（注意：无UNION ALL，因为只选了SA类型）：**

```sql
WITH
seg_1 AS (
  SELECT user_id
  FROM (
    SELECT user_id, clicks AS metric_value
    FROM sponsored_ads_traffic_for_audiences
    WHERE user_id_type = 'adUserId'
      AND clicks > 0
      AND date_partition BETWEEN '20240101' AND '20240630'
      AND campaign IN ('SP-Brand-Auto','SB-Brand-Video')
  )
  GROUP BY 1
  HAVING SUM(metric_value) >= 5
)
SELECT user_id FROM seg_1
```

---

## 6. 前端 → 后端 API 请求体格式（建议）

```jsonc
{
  "modules": [
    {
      "id": "mod_1",
      "eventType": "impression",           // ${EVENT_TYPE}
      "adFilter": {                         // 广告行为类专用
        "adTypes": ["PVA", "SP"],           // ${AD_TYPE}
        "advertiser": "",                   // ${ADV}
        "entity": "",                       // ${ENTITY}
        "campaigns": ["camp_a", "camp_b"]   // ${CAMPAIGN}
      },
      "timeRange": {                        // ${TIME_FILTER}
        "type": "relative",
        "relativeDays": 30
      },
      "metricFilter": {                     // ${HAVING_CLAUSE}
        "metricType": "count",              // ${METRIC_TYPE}
        "operator": "gte",                  // ${METRIC_OP}
        "value": 3                          // ${METRIC_VAL}
      }
    },
    {
      "id": "mod_2",
      "eventType": "purchase",
      "attributeFilter": {                  // 自然行为类专用
        "attributeType": "asin",            // ${ATTR_TYPE}
        "matchType": "exact",               // ${MATCH_TYPE}
        "values": ["B0XXX"]                 // ${ATTR_VALUES}
      },
      "timeRange": {
        "type": "relative",
        "relativeDays": 30
      },
      "metricFilter": {
        "metricType": "count",
        "operator": "gte",
        "value": 1
      }
    }
  ],
  "connections": [
    { "operation": "NOT" }                  // ${SET_OP} 模块1和模块2之间
  ]
}
```

---

## 7. 参数联动关系图（前端渲染控制）

```
${EVENT_TYPE} 变更
    │
    ├── 广告行为类 (impression / viewable_impression / click)
    │   ├── 显示: AD_TYPE, ADV, ENTITY, CAMPAIGN
    │   ├── 隐藏: ATTR_TYPE, MATCH_TYPE, ATTR_VALUES
    │   └── METRIC_TYPE 锁定为 count
    │
    └── 自然行为类 (search / detail_page_view / add_to_cart / add_to_wishlist / purchase)
        ├── 隐藏: AD_TYPE, ADV, ENTITY, CAMPAIGN
        ├── 显示: ATTR_TYPE, MATCH_TYPE, ATTR_VALUES
        │   │
        │   └── ${EVENT_TYPE} 决定可选的 ATTR_TYPE:
        │       ├── search         → [search_term]
        │       └── 其余自然行为    → [product_line, asin]
        │
        │   └── ${ATTR_TYPE} 决定可选的 MATCH_TYPE:
        │       ├── search_term    → [exact, fuzzy]
        │       └── product_line/asin → [exact]（锁定）
        │
        └── METRIC_TYPE:
            ├── purchase → [count, amount]（可选）
            └── 其余     → [count]（锁定）
```

---

## 8. 安全与校验

### 8.1 SQL注入防护

所有用户输入值（CAMPAIGN、ATTR_VALUES、ADV、ENTITY）在拼接SQL前必须：
- 转义单引号：`'` → `''`
- 去除分号、注释符等危险字符
- **推荐使用参数化查询** 如果AMC API支持

### 8.2 前端校验规则

| 校验项 | 规则 |
|-------|------|
| 模块数量 | 最少1个，最多5个 |
| EVENT_TYPE | 必选 |
| AD_TYPE（广告行为时） | 至少选1个 |
| METRIC_VAL | 必须 > 0 |
| TIME_FILTER | 开始日期 <= 结束日期；相对天数 > 0 |
| ATTR_VALUES | 非空时每个值不能为空字符串 |
