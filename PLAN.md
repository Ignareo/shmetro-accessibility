# 地铁可达性算法优化计划

> 架构：catalog → resolve → crawl → 输出/前端聚合。
> 数据口径示例：上海库 530 线别节点 / 416 同名站组 / 全量有向对 280,370（快照 2026-10-09，
> 当时已爬 6 万+，**增量爬取进行中，数字随时漂移**；耗时基准按实跑 QPS 重估）。

## 代价模型

- resolve 阶段一次性成本（结果持久化，命中即跳过）。
- crawl 阶段 O(n²) API 调用，是全周期成本主体。
- route_times 主键不含日期/时间维度：口径缓存校验见方向 3 之 B4（已实现）。

## 方向 1：官方地铁数据源 → 本地图路由（先 spike，再承诺建图）

风险（review 修正后）：

- 唯一数据源证据是 2017 年知乎文章，接口现状未验证 → 先做半天级 spike：
  接口是否存活、是否有站间运行时间/换乘时间/首末班字段、ToS 限制。
- 上海无官方 GTFS；r5r/OTP 只是算法参照（依赖 GTFS+OSM），不能直接套用，工作量之前被低估。
- 本方向为上海专用，不进其它城市 variant 的公共路径。

验收判据：

- 校验集 = "纯地铁 done 子集"：按 summary 过滤。**当前口径有污染（A3，已证实）**：
  674 条 done 含磁浮段、2,571 条含公交段（其中 407 条纯公交）；代码只过滤 taxi，
  `contains_maglev` 是死代码。校验前必须先按 summary 清洗。
- 图模型 vs 实测：P90 误差 <2 分钟且 P99 <3 分钟（含等车时间建模说明）。
- spike 通过前不投入建图。

参照：[r5r travel_time_matrix](https://ipeagit.github.io/r5r/reference/travel_time_matrix.html)、
[gtfsrouter](https://github.com/UrbanAnalyst/gtfsrouter/issues/73)、
[OpenTripPlanner TravelTime](https://opentripplanner.readthedocs.io/en/latest/sandbox/TravelTime/)、
[Helsinki 矩阵数据集](https://pmc.ncbi.nlm.nih.gov/articles/PMC11315881/)。

## 方向 2：组代表元爬取（已完成，含 review 修正）

### 精度论证（analyze_group_spread.py，与实现同口径：POI 聚类 + 代表 rep→rep）

- 旧结论作废：`best - first_best` 恒 ≤0（符号写反），"100% 拿到组内最优"无效。
- 生产口径（n=48,495 已覆盖簇对）：代表误差 ≤0.5min 占 99.94%，max 5.9min。
- 多观测子集（n=15,651，唯一能看见尾巴的诚实样本）：≤0.5min 占 99.83%；
  >3min 尾巴仅涉及 1 个起点簇（上海南站）。
- 样本偏差提示：多观测簇对仅占已覆盖簇对约 1/3，偏向先爬完的线路，外推保留余量。

### 实现（已合入并验证）

- `--group-reps`：同名组先按已解析 POI 距离聚类，**>300m 拆簇**（A4，已证实同名≠同站：
  2/14 号线浦东南路相距 657m、2/17 号线国家会展中心相距 692m；误差尾巴全部来自这类组）。
- 代表元选取受 `--from-lines`/`--to-lines` 约束（B2：范围内站点以该线路节点出发，
  已验证 8号线范围内人民广场选 08- 节点）。
- 簇对只要任一成员对已有最终结果即视为已覆盖，与存量节点级数据自然互补。
- 输出口径（B3）：group-reps 下节点级 CSV/ranking 非代表节点大面积 missing/NaN，
  属设计行为；**前端 stations.json（组级）为权威输出**；与 `--from-lines`/`--to-lines`
  组合时覆盖判定仍是簇级（范围外线成员已覆盖的簇会跳过），运行时会打印提示。
- 收尾新增调用：节点级 ~208k vs 簇级 ~125k（−40%，快照随爬取漂移）。
- pairs 固定种子打乱，避免 capped run 固定顺序只推进同一条前沿。

## 方向 3：零 API 成本的数据质量算法（已完成，含 review 修正）

### 审计（audit_routes.py，全库数秒（70k done 约 2.4s）；结果存 route_flags 表 + output/route_audit.md）

- `triangle_slow ≥12min`（excess 分布 P99≈10.3，阈值在尾巴上）：约 400 个，**集中在
  02-浦东1号2号航站楼、02-龙阳路** → 同原点集中出现 = 该站 POI 疑似配到
  远处出口，优先复核清单。
- `triangle_fast ≤−20min`：40+ 个。初版阈值 −12 时 700+ 个，但其中绝大多数是"两段换乘
  开销 vs 直达"的正常差距（解读错误，已收紧到只看极端值）。
- `direction_asym ≥12min 且 ≥30%`：6 个（已移出"≥3 中间站"门槛，稀疏覆盖下不再漏检；
  含浦东机场↔曹杨路，反向 +23min）。
- `group_offset ≥4min`：0 个。

### no_valid_route 构成（A1 修正，已证实）

- 当前 101 个 **全部 reason=no_transits**；98 个是同名站跨线对内点对（AMap 对同站
  换乘返回无可乘方案，属步行场景），跨名对仅 3 个且均短途（浦东南路→商城路等）。
- "max_trans=5 卡死远郊对"的判断**错误，已撤销**：放宽 max_trans 没有对象。
- `--recrawl-no-valid-route` 开关保留给未来真出现的跨名无解对，默认行为不变；
  机制说明：`route_result_is_final` 本把 no_valid_route 视为最终态，该开关通过
  `_result_final_for_run` 重新纳入待爬并覆写，无需手动删行。

### 对称回填 `--fill-symmetric`

- 前端缺失行用已爬反向对时间兜底，meta.json 披露 `estimated_pairs`（快照 34,786 条）；
  ranking/CSV 保持纯实测口径。抽样验证回填值正确。

### 口径缓存校验（B4，已实现并验证；review 二轮加固）

- `crawl_params` 表记录 (date, time, strategy)；route_times 无此维度，冲突时：
  - 正常爬取：必须 `--reset-crawl-cache`（删除旧 done/no_valid 行）否则直接退出；
  - `--compute-only`：警告并用存量口径标注输出，不中断；
  - `--resolve-only`：不涉及爬取缓存，直接放行，也不会触发缓存删除；
  - 存量库无口径行（迁移真空）：打印警告，用请求参数标注，直到某次爬取钉定真实口径。
- 存量库首次以新代码爬取时自动钉定口径；校验/删除逻辑已抽成
  `guard_service_params` 单一助手，shmetro 与 common 共用，防止两份漂移。

## resolve 阶段（review 修正：不做替代，降级为审计数据源）

- 模板搜索的价值正是 README 强调的"线别化出口 POI"精度；且 530/530 全 resolved、
  一次性仅数百次调用——网格化全量抓取"替代"方案撤销。
- 降级定位：错配审计的数据源。方向 3 已先证明存在错配（triangle_slow 聚集在浦东
  机场站），后续如需复核可用全量 POI 网格抓取交叉验证，不必先投入。

## 工程小项（现状）

- ✅ pairs 固定种子打乱（真问题：固定顺序 → 前沿推进 + error 对反复占队首 + 覆盖向
  先爬线路倾斜；原"每次只爬同一片"表述不准确——已完成对本来就会跳过）。
- ✅ QPS 默认 3.01 → 3.0（避免偶发 CUQPS 重试）。
- ✅ 重试退火 jitter（防多 worker 同步重试风暴）。
- ⬇️ executemany 批量写库：WAL + synchronous=NORMAL 下写入非瓶颈，优先级下调，暂不做。
- 待办：接 `chinese-calendar` 自动规避节假日。
- 待核实：新计费页下 `AlternativeRoute=8` 的多备选是否按次加权（A3 旁证：多备选
  实际只服务 taxi 过滤，磁浮过滤为死代码）。

## 参考

- 高德路径规划 2.0：https://lbs.amap.com/api/webservice/guide/api/newroute
- 高德基础服务计费：https://lbs.amap.com/pages/base_service_price
- 上海地铁官网数据解析（待 spike 验证）：https://zhuanlan.zhihu.com/p/40988487
