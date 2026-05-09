---
name: doris-sql-performance-analyzer
description: |
  Doris SQL 性能评估工具。根据 SQL 自动获取表结构、评估数据量级和分桶策略、
  生成基于 SQL 逻辑的模拟数据、分析执行计划，最终输出性能分析报告。
  任何时候用户提到 Doris 性能分析、SQL 优化、执行计划分析、分桶评估、模拟数据生成、
  SQL 性能测试、影子表测试、Profile 分析、查询速度慢、查询优化、表结构分析、Doris 调优，
  甚至只是说"这条 SQL 跑得慢帮我看看"或者发了 Doris SQL 问能不能优化——只要涉及 Doris SQL
  的执行效率问题，都应当使用本 skill。如果用户提到"分析这条SQL"加上Doris相关上下文，也触发。
  触发词（任意匹配即触发）：Doris、性能分析、执行计划、影子表、分桶评估、模拟数据、SQL 性能测试、
  Profile、查询优化、SQL 调优、慢查询。
---

# Doris SQL 性能分析器
<!-- managed by ai-you, auto-synced to cc-switch & ai-study -->

根据用户提供的 Doris SQL，自动完成从表结构分析到性能报告输出的完整流程。

## 工作流

### Step 1: 解析 SQL 提取表名

用正则从 SQL 中提取 `FROM` / `JOIN` 后面的表名（支持 `db.table` 格式）。同时提取 CTE、子查询引用。每个表名记为一条记录。

### Step 2: 问询约束与环境

**一次性问清**以下信息，不要逐条挤牙膏：

1. **Doris 版本号**（如 2.0.3、2.1.x、3.0.x）— 这决定了 CTE 物化、子查询去关联的行为差异
2. **表结构修改权限**：哪些表可以改结构（分桶、分区、索引），哪些是 ODS/外部表不可改
3. **优化手段约束**：是否允许使用 CTE 物化提示 / 中间临时表
4. **执行环境**：在哪套环境执行（dev/fat/uat/pro），决定了 Doris 连接和 ANALYZE 安全性
5. **性能目标**（可选）：预期优化到什么程度（如"从 30s 降到 5s"）

记住这些约束——后续所有建议都必须在此范围内。尤其不能对 ODS 层表建议结构变更。

### Step 3: 获取表结构

优先使用 `doris_get_table_basic_info` 一次性获取行数、列数、分区数、表大小。然后使用 `doris_get_table_schema` 获取完整 DDL。

补充检查 `sql/table_schemas/` 目录是否有匹配的本地缓存（文件名格式：`<库名>_<表名>.sql`）。

对**没有本地缓存且 MCP 工具不可用**的表，一次性问询用户贴建表语句并选择数据量级（1万/10万/100万/1000万行）。

### Step 4: 保存建表语句到本地

将本轮获取到的建表语句保存到 `sql/table_schemas/<库名>_<表名>.sql`，供下次复用。

### Step 5: 生成影子表脚本

**先问询**："是否需要生成新的影子表脚本？如果之前已经 mock 过（影子表仍存在且有数据），可以直接复用。"

选项：
- 需要生成新的（继续生成 DDL + mock 数据）
- 已有影子表，直接用于分析（跳到 Step 6，用已有表名）

如果用户选择生成新的，基于建表语句生成影子表 DDL + mock 数据 SQL，输出到 `performance-reports/<TIMESTAMP>/ddl_and_mock.sql`。

影子表命名：`<原表名>_shadow`。DDL 保持与原表一致的分桶和分区策略。

Mock 数据生成原则：
- 根据列名和数据类型生成合理的模拟值
- 对数值型字段生成有区分度的值（不要全是 1, 2, 3...）
- 对时间字段生成符合 WHERE 条件的日期范围
- 对字符串字段生成有业务语义的值
- **数据量根据用户选择决定，默认推荐 10万行**（200 行无法触发真实执行计划策略，如 broadcast vs shuffle join）
- 使用 INSERT INTO ... VALUES 方式，每批 100 行
- 如果 SQL 中有 JOIN 条件（如 `a.id = b.user_id`），确保关联键的值能匹配上（限制关联键的取值范围使主键有重叠）
- 如果 SQL 中有 WHERE 条件（如 `status = 1`），让 30-70% 的行满足过滤条件，以模拟真实选择性

### Step 6: 问询执行确认

- 如果用户选择了"已有影子表"：直接确认已有的影子表名（默认 `原表名_shadow`），跳到 Step 7
- 如果用户选择了"需要生成新的"：问用户"影子表脚本已生成到 `<path>`，请在 Doris 中执行。建议在 dev/fat 环境执行。完成后告诉我。"

执行选项：
- 已执行完成
- 跳过影子表（直接对原表做 EXPLAIN）

### Step 7: ANALYZE TABLE（关键步骤）

**在获取执行计划之前，必须提示用户执行 ANALYZE TABLE**，否则 Doris 优化器使用过时或缺失的统计信息，JOIN 顺序和扫描策略可能严重失真。

如果用户选择了影子表路径：
```
请在 Doris 中执行以下命令后告诉我：
ANALYZE TABLE <shadow_table_1> WITH SYNC MODE;
ANALYZE TABLE <shadow_table_2> WITH SYNC MODE;
...
```

如果用户跳过了影子表（直接对原表分析），仅在 dev/fat 环境建议执行：
```
建议在 dev/fat 环境对原表运行 ANALYZE TABLE 以获取准确统计：
ANALYZE TABLE <table> WITH SYNC MODE;
```

### Step 8: 获取执行计划

使用 `doris_get_sql_explain(sql=用户的SQL, verbose=true)` 获取执行计划。

如果使用了影子表，将 SQL 中的表名替换为影子表名后再获取。同时获取简化版 EXPLAIN（不带 verbose）作对比参考。

### Step 9: 获取 Profile（建议必做）

Profile 提供了 EXPLAIN 无法反映的运行时细节：实际内存消耗、IO 等待、数据倾斜。尤其当 EXPLAIN 没有明显瓶颈时，Profile 往往能发现真相。

使用 `doris_get_sql_profile` 获取 Profile。

### Step 10: 利用 Doris MCP 工具进行深度分析

根据执行计划中的疑点选择性使用以下工具：

| 疑点 | 工具 | 分析重点 |
|------|------|---------|
| JOIN 慢、数据倾斜 | `doris_analyze_table_storage` | 分桶键是否与 JOIN 键对齐、分桶数是否合理 |
| 过滤条件扫太多行 | `doris_analyze_columns` | 过滤列的值分布、NULL 率 |
| 聚合慢 | `doris_analyze_columns` | GROUP BY 列的基数 |
| 不确定索引效果 | `doris_get_table_indexes` | 现有索引覆盖情况 |
| 整体评估 | `doris_get_table_data_size` | 表实际大小、副本数 |

**分桶评估具体做法**：
- 从 `doris_analyze_table_storage` 和 `doris_get_table_basic_info` 中读取当前分桶数和数据量
- 计算每桶行数 = 总行数 / 分桶数
- 经验最优区间：每桶 100万-500万行（OLAP 场景）
- 如果每桶 < 50万行 → 分桶过多，浪费内存
- 如果每桶 > 1000万行 → 分桶过少，并行度不足
- 给出**具体建议值**（如"建议从 4 个桶增加到 16 个桶"），并说明计算依据
- 注意：1 桶是特殊 red flag，在报告中标注为 P0

注意：这些工具只能在真实表或已执行的影子表上使用。不要对未确认存在的表调用。

### Step 11: 利用常见优化模式逐项排查

在撰写报告前，用以下模式对 SQL 做最后一轮检查（不需要工具，纯逻辑判断）：

1. **EXISTS 关联子查询**：能否改写为 `LEFT SEMI JOIN`？Doris 对关联子查询的去关联化在不同版本表现不一致，LEFT SEMI JOIN 语义更明确且避免行膨胀
2. **冗余 JOIN**：子查询/CTE 已做了过滤，外层是否还在 JOIN 同一个表？检查能否移除
3. **DISTINCT 仅用于过滤**：`SELECT DISTINCT ... WHERE x IN (...)` 如果仅为了防止重复行，考虑是否存在更轻量的去重方式
4. **单桶表瓶颈**：是否有表只有 1 个桶？这是最常见的性能瓶颈——如果该表参与 JOIN，它会成为广播侧或导致单节点处理全部数据。注意 ODS 层表可能无法改结构
5. **CTE 多次引用**：同一个 CTE 被引用多次时，老版本 Doris 可能不会自动物化。考虑建议添加 `/*+ SET_VAR(enable_materialize_cte=true) */` hint（需确认版本支持）
6. **分区裁剪失效**：WHERE 条件是否命中分区键？如果分区键是 `dt` 但 WHERE 用的是函数包裹的 `DATE(dt)`，会导致分区裁剪失效

### Step 12: 生成性能报告

输出到 `performance-reports/<TIMESTAMP>/performance_report.md`，必须包含：

#### 报告模板

```
# [SQL 描述] 性能分析报告

## 分析概要
- 分析时间
- Doris 版本
- 影子表状态（已使用/已跳过）
- ANALYZE 状态（已执行/未执行）
- 约束说明（哪些表不可改结构、哪些优化手段不可用）

## 表结构概览
| 表名 | 行数 | 分桶数 | 分区数 | 大小 | 角色 |
|------|------|--------|--------|------|------|

## 执行计划分析
- 扫描方式
- JOIN 类型与顺序
- 聚合方式
- 排序方式
- 关键算子耗时

## 瓶颈识别（按优先级排列）

每个瓶颈标注优先级：
- **P0 - 严重**：必须修复，否则性能无法达标（如单桶表、全表扫描、笛卡尔积 JOIN）
- **P1 - 重要**：应该修复，有明显收益（如分桶数不合理、缺乏索引、JOIN 顺序不佳）
- **P2 - 建议**：锦上添花，长期优化的方向（如分区细化、索引微调）

每个瓶颈包含：
- 现象（执行计划中的具体数据）
- 根因（为什么会发生）
- 影响（对整体性能的量化估算）
- 建议（具体可执行的操作，含修改后的 SQL/DDL）

## 优化方案

### 代码层优化（可立即执行）
改写后的 SQL，标注每一处改动的原因。

### 结构层优化（需评估影响）
仅在约束允许的范围内建议。包含具体 DDL。

### 统计信息优化
建议执行的 ANALYZE 命令。

## 优化效果预估
基于执行计划和经验估算的优化后性能。
```

### Step 13: 输出优化 SQL

将报告中代码层优化的改写 SQL 单独输出到 `performance-reports/<TIMESTAMP>/optimized_sql.sql`，包含注释说明每处改动的目的。

## 优化模式参考

检查 SQL 时重点关注以下模式，它们是最常见的 Doris 性能杀手：

| 模式 | 识别方法 | 优化方向 | 风险 |
|------|---------|---------|------|
| EXISTS 关联子查询 | `WHERE EXISTS (SELECT 1 FROM b WHERE b.x = a.x)` | `LEFT SEMI JOIN b ON a.x = b.x` | 语义等价，无风险 |
| 冗余 JOIN | 子查询已过滤表 t，外层又 JOIN t | 移除外层 JOIN | 需确认字段引用 |
| 单桶表参与 JOIN | EXPLAIN 中 `buckets: 1` 且作为 JOIN 右侧 | 评估能否增加桶数 | ODS 表通常不可改 |
| CTE 多次引用 | 同一 WITH 子句被引用 ≥2 次 | 添加物化 hint | 需 Doris 2.1+ |
| 分区键函数包裹 | `WHERE DATE(dt)='2026-01-01'` | 改为 `WHERE dt >= '2026-01-01' AND dt < '2026-01-02'` | 无风险 |
| 高基数 GROUP BY | GROUP BY 的列 NDV 很高 | 评估是否必要条件 | 业务逻辑确认 |

## 工具参考

| 工具 | 用途 | 时机 |
|------|------|------|
| `Read` | 读取本地缓存 DDL | Step 3 |
| `Write` | 保存 DDL、影子表脚本、报告 | Step 4-13 |
| `question` | 问询约束、DDL、量级、执行确认 | Step 2-6 |
| `doris_get_table_basic_info` | 一次性获取行数/列数/分区/大小 | Step 3（首选） |
| `doris_get_table_schema` | 获取完整 DDL | Step 3 |
| `doris_get_sql_explain` | 获取执行计划 | Step 8 |
| `doris_get_sql_profile` | 获取 Profile | Step 9 |
| `doris_analyze_table_storage` | 存储/分桶分析 | Step 10 |
| `doris_analyze_columns` | 列分布分析 | Step 10 |
| `doris_get_table_indexes` | 索引信息 | Step 10 |
| `doris_get_table_data_size` | 表大小 | Step 10 |

## 输出文件结构

```
sql/table_schemas/
  ├── <库名>_<表名1>.sql
  └── <库名>_<表名2>.sql

performance-reports/<TIMESTAMP>/
  ├── ddl_and_mock.sql        ← 影子表 DDL + mock 数据
  ├── performance_report.md   ← 完整分析报告（含 P0/P1/P2 优先级）
  └── optimized_sql.sql       ← 优化后 SQL
```

## 注意事项

1. 不要假设 MCP 可以直接获取 DDL——优先检查本地缓存，其次 `doris_get_table_schema`，最后问用户
2. mock 数据量默认 10 万行起，不要用 200 行凑合过小数据量无法触发真实执行计划策略
3. **所有问询合并成一轮**，不要挤牙膏式多次问
4. **EXPLAIN 之前必须提示 ANALYZE TABLE**——这是准确执行计划的前提
5. **所有优化建议必须在用户约束范围内**——对 ODS 表不提议结构变更
6. **报告瓶颈必须标注 P0/P1/P2 优先级**，让用户知道先改哪个
7. 报告中的每个结论必须引用 EXPLAIN/Profile 中的具体数据作为依据
8. 所有文件路径基于当前工作目录，目录不存在时自动创建
