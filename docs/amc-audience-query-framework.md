# AMC Audience Query Builder - Framework Design

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend UI Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Behavior  │  │ Behavior  │  │ Behavior  │  ...        │
│  │ Module 1  │──│ Module 2  │──│ Module 3  │             │
│  └──────────┘  └──────────┘  └──────────┘              │
│       │  AND/OR/NOT  │  AND/OR/NOT  │                   │
│  ┌──────────────────────────────────────────┐           │
│  │         BehaviorModuleManager             │           │
│  └──────────────────────────────────────────┘           │
└──────────────────────┬──────────────────────────────────┘
                       │ AudienceQueryState
┌──────────────────────▼──────────────────────────────────┐
│                 Query Builder Layer                       │
│  ┌────────────────┐  ┌────────────────┐                 │
│  │ BehaviorRegistry│  │ SQLTemplateEngine│                │
│  │ (行为注册表)     │  │ (SQL模板引擎)   │                │
│  └────────────────┘  └────────────────┘                 │
│  ┌────────────────────────────────────────┐             │
│  │         QueryAssembler                  │             │
│  │  (CTE组装 + Set Operation拼接)          │             │
│  └────────────────────────────────────────┘             │
└──────────────────────┬──────────────────────────────────┘
                       │ Final SQL String
┌──────────────────────▼──────────────────────────────────┐
│                    AMC API Layer                          │
│            Submit query → Get audience                    │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Core Data Model

### 2.1 行为事件分类

行为事件分为两大类，对应不同的 AMC 数据表和筛选维度：

```
行为事件
├── 广告行为类 (Ad-Related) ─── 有广告来源、广告类型、Campaign筛选
│   ├── 曝光 (Impression)
│   ├── 有效曝光 (Viewable Impression)
│   └── 点击 (Click)
│
└── 自然行为类 (Organic) ─── 有行为属性（搜索词/ASIN等）筛选
    ├── 搜索 (Search)
    ├── 详情页浏览 (Detail Page View)
    ├── 购物车加购 (Add to Cart)
    ├── 心愿单加购 (Add to Wishlist)
    └── 购买 (Purchase)
```

### 2.2 TypeScript Type Definitions

```typescript
// ============================================================
// 枚举与基础类型
// ============================================================

/** 行为事件类型 */
type BehaviorEventType =
  | 'impression'           // 曝光
  | 'viewable_impression'  // 有效曝光
  | 'click'                // 点击
  | 'search'               // 搜索
  | 'detail_page_view'     // 详情页浏览
  | 'add_to_cart'          // 购物车加购
  | 'add_to_wishlist'      // 心愿单加购
  | 'purchase';            // 购买

/** 广告来源 */
type AdSource = 'DSP' | 'SA';

/** 广告类型 */
type AdType = 'PVA' | 'OLV' | 'SP' | 'SB' | 'SD';

/** 广告类型 → 广告来源 映射 */
const AD_TYPE_SOURCE_MAP: Record<AdType, AdSource> = {
  PVA: 'DSP',
  OLV: 'DSP',
  SP: 'SA',
  SB: 'SA',
  SD: 'SA',
};

/** 集合运算符（行为模块之间的组合关系）*/
type SetOperation = 'AND' | 'OR' | 'NOT';
// AND → SQL INTERSECT（交集：同时满足）
// OR  → SQL UNION（并集：满足任一）
// NOT → SQL EXCEPT（差集：排除）

/** 比较运算符 */
type ComparisonOperator = '>=' | '>' | '=' | '<' | '<=' | '!=';

/** 属性匹配方式 */
type MatchType = 'exact' | 'fuzzy';

/** 统计指标类型 */
type MetricType = 'count' | 'amount';

/** 时间范围类型 */
type TimeRangeType = 'fixed' | 'relative';

// ============================================================
// 筛选条件数据结构
// ============================================================

/** 时间范围 */
interface TimeRange {
  type: TimeRangeType;
  // 固定时间段
  startDate?: string;  // '2024-01-01'
  endDate?: string;    // '2024-03-31'
  // 相对时间段
  relativeDays?: number;  // 过去N天
}

/** 广告筛选条件 - 仅广告行为类使用 */
interface AdFilter {
  adTypes: AdType[];          // 选中的广告类型
  campaigns: string[];        // Campaign名称列表
  advertiser?: string;        // DSP advertiser筛选
  entity?: string;            // SA entity筛选
}

/** 行为属性筛选 - 仅自然行为类使用 */
interface AttributeFilter {
  attributeType: 'search_term' | 'product_line' | 'asin';
  matchType: MatchType;
  values: string[];           // 属性值列表
}

/** 统计指标筛选 */
interface MetricFilter {
  metricType: MetricType;     // count=次数, amount=金额
  operator: ComparisonOperator;
  value: number;
}

// ============================================================
// 行为模块 - 核心数据结构
// ============================================================

/** 单个行为模块的完整配置 */
interface BehaviorModule {
  id: string;                           // 唯一标识 (uuid)
  eventType: BehaviorEventType;         // 行为事件类型
  timeRange: TimeRange;                 // 行为时间
  adFilter?: AdFilter;                  // 广告筛选（广告行为类）
  attributeFilter?: AttributeFilter;    // 属性筛选（自然行为类）
  metricFilter: MetricFilter;           // 统计指标筛选
}

/** 行为模块之间的连接关系 */
interface BehaviorConnection {
  operation: SetOperation;  // AND / OR / NOT
}

/** 完整的受众查询状态 */
interface AudienceQueryState {
  modules: BehaviorModule[];          // 行为模块列表（有序）
  connections: BehaviorConnection[];  // 模块间连接（长度 = modules.length - 1）
}
```

---

## 3. Behavior Registry（行为注册表）

行为注册表定义每种行为事件对应的 AMC 数据表、可用筛选项和 SQL 生成规则。

### 3.1 AMC 表映射

```
行为事件              AMC 表                                        指标列
────────────────────────────────────────────────────────────────────────────
曝光 (impression)
  └─ DSP             dsp_impressions_for_audiences                 impressions
  └─ SA              sponsored_ads_traffic_for_audiences            impressions

有效曝光 (viewable_impression)
  └─ DSP             dsp_impressions_for_audiences                 viewable_impressions
  └─ SA              sponsored_ads_traffic_for_audiences            viewable_impressions

点击 (click)
  └─ DSP             dsp_clicks_for_audiences                      clicks
  └─ SA              sponsored_ads_traffic_for_audiences            clicks

搜索                  amazon_organic_search_for_audiences           1 (COUNT)
详情页浏览            detail_page_views_for_audiences               1 (COUNT)
购物车加购            add_to_carts_for_audiences                    1 (COUNT)
心愿单加购            add_to_wishlists_for_audiences                1 (COUNT)
购买                  conversions_all_for_audiences                 total_product_sales / 1
```

### 3.2 Registry Definition

```typescript
interface TableConfig {
  tableName: string;
  metricColumn: string;               // SUM的目标列
  additionalFilters?: string[];       // 表级固定过滤条件
}

interface BehaviorConfig {
  label: string;
  category: 'ad' | 'organic';

  // 广告行为类：按广告来源分表
  tables?: Record<AdSource, TableConfig>;
  // 自然行为类：单表
  table?: TableConfig;

  // 可用筛选维度
  availableFilters: {
    adSource?: boolean;          // 广告来源筛选
    adType?: boolean;            // 广告类型筛选
    campaign?: boolean;          // Campaign筛选
    attribute?: {                // 行为属性筛选
      types: ('search_term' | 'product_line' | 'asin')[];
      matchTypes: MatchType[];
    };
  };

  // 可用统计指标
  availableMetrics: MetricType[];
}

const BEHAVIOR_REGISTRY: Record<BehaviorEventType, BehaviorConfig> = {

  impression: {
    label: '曝光',
    category: 'ad',
    tables: {
      DSP: {
        tableName: 'dsp_impressions_for_audiences',
        metricColumn: 'impressions',
        additionalFilters: ['impressions > 0'],
      },
      SA: {
        tableName: 'sponsored_ads_traffic_for_audiences',
        metricColumn: 'impressions',
        additionalFilters: ['impressions > 0'],
      },
    },
    availableFilters: { adSource: true, adType: true, campaign: true },
    availableMetrics: ['count'],
  },

  viewable_impression: {
    label: '有效曝光',
    category: 'ad',
    tables: {
      DSP: {
        tableName: 'dsp_impressions_for_audiences',
        metricColumn: 'viewable_impressions',
        additionalFilters: ['viewable_impressions > 0'],
      },
      SA: {
        tableName: 'sponsored_ads_traffic_for_audiences',
        metricColumn: 'viewable_impressions',
        additionalFilters: ['viewable_impressions > 0'],
      },
    },
    availableFilters: { adSource: true, adType: true, campaign: true },
    availableMetrics: ['count'],
  },

  click: {
    label: '点击',
    category: 'ad',
    tables: {
      DSP: {
        tableName: 'dsp_clicks_for_audiences',
        metricColumn: 'clicks',
        additionalFilters: ['clicks > 0'],
      },
      SA: {
        tableName: 'sponsored_ads_traffic_for_audiences',
        metricColumn: 'clicks',
        additionalFilters: ['clicks > 0'],
      },
    },
    availableFilters: { adSource: true, adType: true, campaign: true },
    availableMetrics: ['count'],
  },

  search: {
    label: '搜索',
    category: 'organic',
    table: {
      tableName: 'amazon_organic_search_for_audiences',
      metricColumn: '1',  // COUNT(1)
    },
    availableFilters: {
      attribute: {
        types: ['search_term'],
        matchTypes: ['exact', 'fuzzy'],
      },
    },
    availableMetrics: ['count'],
  },

  detail_page_view: {
    label: '详情页浏览',
    category: 'organic',
    table: {
      tableName: 'detail_page_views_for_audiences',
      metricColumn: '1',
    },
    availableFilters: {
      attribute: {
        types: ['product_line', 'asin'],
        matchTypes: ['exact'],
      },
    },
    availableMetrics: ['count'],
  },

  add_to_cart: {
    label: '购物车加购',
    category: 'organic',
    table: {
      tableName: 'add_to_carts_for_audiences',
      metricColumn: '1',
    },
    availableFilters: {
      attribute: {
        types: ['product_line', 'asin'],
        matchTypes: ['exact'],
      },
    },
    availableMetrics: ['count'],
  },

  add_to_wishlist: {
    label: '心愿单加购',
    category: 'organic',
    table: {
      tableName: 'add_to_wishlists_for_audiences',
      metricColumn: '1',
    },
    availableFilters: {
      attribute: {
        types: ['product_line', 'asin'],
        matchTypes: ['exact'],
      },
    },
    availableMetrics: ['count'],
  },

  purchase: {
    label: '购买',
    category: 'organic',
    table: {
      tableName: 'conversions_all_for_audiences',
      metricColumn: '1',  // COUNT时用1, SUM金额时用 total_product_sales
    },
    availableFilters: {
      attribute: {
        types: ['product_line', 'asin'],
        matchTypes: ['exact'],
      },
    },
    availableMetrics: ['count', 'amount'],  // 购买多一个"金额"指标
  },
};
```

---

## 4. SQL Query Generation Strategy

### 4.1 核心思路：CTE段 + 集合运算

```
最终SQL结构:

WITH
  seg_1 AS ( ... ),      ← 行为模块1生成的CTE
  seg_2 AS ( ... ),      ← 行为模块2生成的CTE
  seg_3 AS ( ... ),      ← 行为模块3生成的CTE
  ...
  combined AS (          ← 集合运算组合
    SELECT user_id FROM seg_1
    INTERSECT
    SELECT user_id FROM seg_2
    EXCEPT
    SELECT user_id FROM seg_3
  )
SELECT user_id FROM combined
```

**为什么选择 CTE + Set Operation 而非 JOIN：**

| 方案 | 优点 | 缺点 |
|------|------|------|
| CTE + Set Operation | SQL清晰、引擎优化好、支持任意组合 | 无 |
| 多层 JOIN | - | 多表JOIN性能差、NULL处理复杂 |
| 多次查询+中间存储 | - | AMC不支持保存中间结果 |
| 嵌套子查询 | - | 可读性差、难以动态生成 |

### 4.2 每个 CTE 段的 SQL 模板

#### 模板A：广告行为类（曝光/有效曝光/点击）

支持多广告来源 UNION ALL：

```sql
seg_{id} AS (
  SELECT user_id
  FROM (
    -- DSP 数据源（如果选中了DSP相关的广告类型）
    SELECT user_id, {metric_column} AS metric_value
    FROM {dsp_table}
    WHERE user_id_type = 'adUserId'
      AND {metric_column} > 0
      {time_filter}
      {advertiser_filter}
      {campaign_filter}

    UNION ALL

    -- SA 数据源（如果选中了SA相关的广告类型）
    SELECT user_id, {metric_column} AS metric_value
    FROM {sa_table}
    WHERE user_id_type = 'adUserId'
      AND {metric_column} > 0
      {time_filter}
      {entity_filter}
      {campaign_filter}
  )
  GROUP BY 1
  HAVING SUM(metric_value) {operator} {value}
)
```

**动态规则：**
- 如果用户只选了 DSP 类型（PVA/OLV）→ 去掉 SA 的 UNION ALL 部分
- 如果用户只选了 SA 类型（SP/SB/SD）→ 去掉 DSP 的 UNION ALL 部分
- 如果都选了 → 保留完整 UNION ALL

#### 模板B：自然行为类（搜索/浏览/加购/购买）

```sql
seg_{id} AS (
  SELECT user_id
  FROM {table}
  WHERE user_id_type = 'adUserId'
    {time_filter}
    {attribute_filter}
  GROUP BY 1
  HAVING {aggregate_function} {operator} {value}
)
```

### 4.3 筛选条件 → SQL Fragment 映射

```typescript
// ── 时间筛选 ──
function buildTimeFilter(timeRange: TimeRange): string {
  if (timeRange.type === 'fixed') {
    return `AND date_partition BETWEEN '${timeRange.startDate}' AND '${timeRange.endDate}'`;
  }
  // 相对时间：过去N天
  return `AND date_partition >= DATE_FORMAT(DATE_ADD('day', -${timeRange.relativeDays}, CURRENT_DATE), '%Y%m%d')`;
}

// ── Campaign 筛选 ──
function buildCampaignFilter(campaigns: string[]): string {
  if (campaigns.length === 0) return '';
  const list = campaigns.map(c => `'${c}'`).join(',');
  return `AND campaign IN (${list})`;
}

// ── Advertiser 筛选（DSP）──
function buildAdvertiserFilter(advertiser?: string): string {
  if (!advertiser) return '';
  return `AND advertiser = '${advertiser}'`;
}

// ── Entity 筛选（SA）──
function buildEntityFilter(entity?: string): string {
  if (!entity) return '';
  return `AND entity = '${entity}'`;
}

// ── 行为属性筛选 ──
function buildAttributeFilter(attr?: AttributeFilter): string {
  if (!attr || attr.values.length === 0) return '';

  const column = {
    search_term: 'search_term',
    product_line: 'product_line',
    asin: 'asin',
  }[attr.attributeType];

  if (attr.matchType === 'exact') {
    const list = attr.values.map(v => `'${v}'`).join(',');
    return `AND ${column} IN (${list})`;
  }

  // 模糊匹配：用 LIKE + OR
  const conditions = attr.values.map(v => `${column} LIKE '%${v}%'`).join(' OR ');
  return `AND (${conditions})`;
}

// ── HAVING 聚合 ──
function buildHaving(metricFilter: MetricFilter, metricColumn: string): string {
  if (metricFilter.metricType === 'count' && metricColumn === '1') {
    return `HAVING COUNT(1) ${metricFilter.operator} ${metricFilter.value}`;
  }
  if (metricFilter.metricType === 'amount') {
    return `HAVING SUM(total_product_sales) ${metricFilter.operator} ${metricFilter.value}`;
  }
  return `HAVING SUM(${metricColumn}) ${metricFilter.operator} ${metricFilter.value}`;
}
```

### 4.4 集合运算组装

```typescript
const SET_OP_SQL: Record<SetOperation, string> = {
  AND: 'INTERSECT',
  OR:  'UNION',
  NOT: 'EXCEPT',
};

function buildCombinedQuery(state: AudienceQueryState): string {
  const cteList: string[] = [];
  const segNames: string[] = [];

  // 1. 为每个行为模块生成 CTE
  state.modules.forEach((module, index) => {
    const segName = `seg_${index + 1}`;
    segNames.push(segName);
    cteList.push(`${segName} AS (\n${buildSegmentCTE(module)}\n)`);
  });

  // 2. 组装 WITH 子句
  const withClause = `WITH\n${cteList.join(',\n')}`;

  // 3. 组装集合运算
  let combinedSelect = `SELECT user_id FROM ${segNames[0]}`;
  state.connections.forEach((conn, index) => {
    const sqlOp = SET_OP_SQL[conn.operation];
    combinedSelect += `\n${sqlOp}\nSELECT user_id FROM ${segNames[index + 1]}`;
  });

  // 4. 最终输出
  return `${withClause}\n${combinedSelect}`;
}
```

### 4.5 完整生成示例

**用户操作：**
- 模块1: 曝光 >= 3次, DSP+SA, Campaign=('camp_a','camp_b'), 过去30天
- 模块2: 购买 >= 1次, ASIN=('B0xxx'), 过去30天
- 组合: 模块1 **NOT** 模块2（曝光但未购买）

**生成的 SQL：**

```sql
WITH
seg_1 AS (
  SELECT user_id
  FROM (
    SELECT user_id, impressions AS metric_value
    FROM dsp_impressions_for_audiences
    WHERE user_id_type = 'adUserId'
      AND impressions > 0
      AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')
      AND campaign IN ('camp_a','camp_b')
    UNION ALL
    SELECT user_id, impressions AS metric_value
    FROM sponsored_ads_traffic_for_audiences
    WHERE user_id_type = 'adUserId'
      AND impressions > 0
      AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')
      AND campaign IN ('camp_a','camp_b')
  )
  GROUP BY 1
  HAVING SUM(metric_value) >= 3
),
seg_2 AS (
  SELECT user_id
  FROM conversions_all_for_audiences
  WHERE user_id_type = 'adUserId'
    AND date_partition >= DATE_FORMAT(DATE_ADD('day', -30, CURRENT_DATE), '%Y%m%d')
    AND asin IN ('B0xxx')
  GROUP BY 1
  HAVING COUNT(1) >= 1
)
SELECT user_id FROM seg_1
EXCEPT
SELECT user_id FROM seg_2
```

---

## 5. Frontend Component Architecture

### 5.1 组件树

```
<AudienceQueryBuilder>                    ← 顶层容器
│
├── <BehaviorModuleList>                  ← 模块列表管理
│   │
│   ├── <BehaviorModule index={0}>        ← 第1个行为模块
│   │   ├── <EventTypeSelector />         ← 行为事件选择（曝光/点击/购买...）
│   │   ├── <TimeRangeSelector />         ← 时间范围
│   │   ├── <AdFilterPanel />             ← 广告筛选面板（条件渲染）
│   │   │   ├── <AdTypeSelect />          ← 广告类型多选
│   │   │   └── <CampaignInput />         ← Campaign输入
│   │   ├── <AttributeFilterPanel />      ← 属性筛选面板（条件渲染）
│   │   │   ├── <AttributeTypeSelect />   ← 属性类型
│   │   │   ├── <MatchTypeSelect />       ← 匹配方式
│   │   │   └── <AttributeValuesInput />  ← 属性值
│   │   └── <MetricFilterPanel />         ← 统计指标面板
│   │       ├── <MetricTypeSelect />      ← 次数/金额
│   │       ├── <OperatorSelect />        ← 比较运算符
│   │       └── <ValueInput />            ← 数值输入
│   │
│   ├── <SetOperationConnector />         ← 集合运算选择器 (AND/OR/NOT)
│   │
│   ├── <BehaviorModule index={1}>        ← 第2个行为模块
│   │   └── ...
│   │
│   └── <AddModuleButton />              ← 添加新模块按钮
│
├── <QueryPreview />                      ← SQL预览（可选，调试用）
└── <SubmitButton />                      ← 生成人群包
```

### 5.2 条件渲染规则

当用户选择不同的行为事件时，面板显示逻辑如下：

```
用户选择行为事件
    │
    ├── 广告行为类（曝光/有效曝光/点击）
    │   ├── ✅ 显示 <AdFilterPanel>
    │   │   ├── 广告类型多选 (PVA, OLV, SP, SB, SD)
    │   │   └── Campaign 输入
    │   ├── ❌ 隐藏 <AttributeFilterPanel>
    │   └── ✅ 显示 <MetricFilterPanel>
    │       └── 指标只有: 次数
    │
    ├── 搜索
    │   ├── ❌ 隐藏 <AdFilterPanel>
    │   ├── ✅ 显示 <AttributeFilterPanel>
    │   │   ├── 属性: 搜索词
    │   │   └── 匹配方式: 精准/模糊
    │   └── ✅ 显示 <MetricFilterPanel>
    │       └── 指标只有: 次数
    │
    ├── 详情页浏览/加购/心愿单
    │   ├── ❌ 隐藏 <AdFilterPanel>
    │   ├── ✅ 显示 <AttributeFilterPanel>
    │   │   ├── 属性: 产品线/ASIN
    │   │   └── 匹配方式: 精准
    │   └── ✅ 显示 <MetricFilterPanel>
    │       └── 指标只有: 次数
    │
    └── 购买
        ├── ❌ 隐藏 <AdFilterPanel>
        ├── ✅ 显示 <AttributeFilterPanel>
        │   ├── 属性: 产品线/ASIN
        │   └── 匹配方式: 精准
        └── ✅ 显示 <MetricFilterPanel>
            └── 指标: 次数 + 金额  ← 购买多一个金额指标
```

### 5.3 State Management

```typescript
// ── Actions ──
type QueryAction =
  | { type: 'ADD_MODULE' }
  | { type: 'REMOVE_MODULE'; moduleId: string }
  | { type: 'UPDATE_MODULE'; moduleId: string; updates: Partial<BehaviorModule> }
  | { type: 'SET_CONNECTION'; index: number; operation: SetOperation }
  | { type: 'CHANGE_EVENT_TYPE'; moduleId: string; eventType: BehaviorEventType };

// ── 当事件类型变更时，需要重置不适用的筛选 ──
function handleEventTypeChange(module: BehaviorModule, newType: BehaviorEventType): BehaviorModule {
  const config = BEHAVIOR_REGISTRY[newType];
  return {
    ...module,
    eventType: newType,
    // 切到广告行为类 → 清空属性筛选，初始化广告筛选
    adFilter: config.category === 'ad' ? { adTypes: [], campaigns: [] } : undefined,
    // 切到自然行为类 → 清空广告筛选，初始化属性筛选
    attributeFilter: config.category === 'organic'
      ? { attributeType: config.availableFilters.attribute!.types[0], matchType: 'exact', values: [] }
      : undefined,
    // 重置统计指标为第一个可用指标
    metricFilter: {
      metricType: config.availableMetrics[0],
      operator: '>=',
      value: 1,
    },
  };
}
```

---

## 6. Query Performance Optimization

### 6.1 AMC Query 优化策略

```
                        优化手段                          预期效果
────────────────────────────────────────────────────────────────────
1. 时间分区下推      date_partition 过滤放入每个CTE       减少80%+数据扫描
2. 按需UNION        只选DSP→去掉SA部分，反之亦然         减少不必要的表扫描
3. 早期聚合         GROUP BY + HAVING 在CTE内完成        减少中间结果集
4. 只SELECT user_id 不带多余列                          减少数据传输
5. 集合运算替代JOIN  INTERSECT/EXCEPT 代替 LEFT JOIN     引擎优化更好
```

### 6.2 用户只选单一广告来源时的优化

```typescript
function buildAdBehaviorCTE(module: BehaviorModule): string {
  const config = BEHAVIOR_REGISTRY[module.eventType];
  const adFilter = module.adFilter!;

  // 根据选中的广告类型，判断需要查询哪些数据源
  const needDSP = adFilter.adTypes.some(t => AD_TYPE_SOURCE_MAP[t] === 'DSP');
  const needSA = adFilter.adTypes.some(t => AD_TYPE_SOURCE_MAP[t] === 'SA');

  const parts: string[] = [];

  if (needDSP) {
    parts.push(buildSingleSourceQuery(config.tables!.DSP, module));
  }
  if (needSA) {
    parts.push(buildSingleSourceQuery(config.tables!.SA, module));
  }

  // 只有一个数据源 → 直接查询，无需子查询包装
  if (parts.length === 1) {
    return `SELECT user_id FROM ${parts[0]} GROUP BY 1\n${buildHaving(...)}`;
  }

  // 多个数据源 → UNION ALL
  return `SELECT user_id FROM (\n${parts.join('\nUNION ALL\n')}\n) GROUP BY 1\n${buildHaving(...)}`;
}
```

---

## 7. Directory Structure (Recommended)

```
src/
├── features/
│   └── audience-query/
│       ├── types/
│       │   └── index.ts                    # 所有TypeScript类型定义
│       ├── config/
│       │   └── behavior-registry.ts        # 行为注册表配置
│       ├── builder/
│       │   ├── sql-fragment-builders.ts    # SQL片段构建函数
│       │   ├── cte-builder.ts             # 单个CTE段构建
│       │   └── query-assembler.ts         # 完整查询组装
│       ├── components/
│       │   ├── AudienceQueryBuilder.tsx    # 顶层容器
│       │   ├── BehaviorModuleList.tsx      # 模块列表
│       │   ├── BehaviorModule.tsx          # 单个行为模块
│       │   ├── EventTypeSelector.tsx       # 行为事件选择
│       │   ├── TimeRangeSelector.tsx       # 时间范围
│       │   ├── AdFilterPanel.tsx           # 广告筛选面板
│       │   ├── AttributeFilterPanel.tsx    # 属性筛选面板
│       │   ├── MetricFilterPanel.tsx       # 统计指标面板
│       │   ├── SetOperationConnector.tsx   # 集合运算连接器
│       │   └── QueryPreview.tsx           # SQL预览
│       ├── hooks/
│       │   ├── useAudienceQuery.ts        # 状态管理hook
│       │   └── useBehaviorConfig.ts       # 行为配置hook
│       └── index.ts                        # 模块导出
```

---

## 8. Data Flow Summary

```
用户操作                    State变更                  SQL生成
─────────────────────────────────────────────────────────────────

选择行为事件      →   更新 eventType            →   选择SQL模板
                      重置不适用筛选

选择广告类型      →   更新 adFilter.adTypes     →   决定UNION ALL结构
                                                    (DSP/SA/Both)

输入Campaign      →   更新 adFilter.campaigns   →   生成 AND campaign IN (...)

选择时间范围      →   更新 timeRange            →   生成 AND date_partition ...

设置属性筛选      →   更新 attributeFilter      →   生成 AND asin/search_term ...

设置统计指标      →   更新 metricFilter         →   生成 HAVING ...

添加/删除模块     →   增减 modules[]            →   增减 CTE段数

切换集合运算      →   更新 connections[]        →   更改 INTERSECT/UNION/EXCEPT

点击提交          →   读取完整 state            →   调用 buildCombinedQuery()
                                                    → 输出完整SQL
```

---

## 9. Edge Cases & Validation

```typescript
// 前端验证规则
const VALIDATION_RULES = {
  // 至少1个行为模块
  minModules: 1,
  // 最多N个模块（防止query过长）
  maxModules: 5,
  // 每个模块必须选择行为事件
  requireEventType: true,
  // 广告行为必须选至少1个广告类型
  adBehaviorRequiresAdType: true,
  // 统计值必须 > 0
  metricValuePositive: true,
  // 时间范围必须有效
  timeRangeValid: true,
};
```
