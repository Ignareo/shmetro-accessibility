# AGENTS.md — 本仓库的 AI 协作约定

## 仓库定位

- fork 工作流（哪些改动回馈上游、PR 分支怎么开）见 `FORK_MAINTENANCE.md`，
  动手改 git 状态前先读它。
- 优化路线图与已完成的论证数据见 `PLAN.md`。

## 代码结构

- `amap_accessibility_common.py` — 高德客户端基类：数字签名、每凭证 QPS 限流、
  重试、transit 方案选择（只过滤 taxi 段）。
- `metro_accessibility_common.py` — 通用管线：catalog 解析、resolve、crawl、
  输出与 frontend JSON。新增通用功能改这里。
- `<城市>metro_accessibility.py` — 每城变体，只提供目录来源与
  `StationResolveRules`。多数城市走 `build_standard_parser` +
  `run_city_accessibility_main`；上海因历史原因是独立 main，新增 CLI 开关时
  **两个 parser 都要加**。
- `shmetro_accessibility_legacy.py` — 旧版爬虫，仅用于生成 `stations_all.csv`。

## 数据与口径

- 默认库 `output/amap_transit.db`（WAL）：`station_amap`（目录+POI 解析）、
  `route_times`（有向对结果）、`crawl_params`（date/time/strategy 口径，
  改口径必须 `--reset-crawl-cache`）、`route_flags`（审计输出）。
- 节点是线别化的：同名不同线 = 不同节点；物理站聚合只发生在前端输出层
  （按站名取 min）。
- `--group-reps` 下节点级 CSV/ranking 会大片 missing，属设计行为；
  前端 `stations.json`（组级）是权威输出。

## 修改约定

- 改 crawl/输出逻辑后验证：`python -m py_compile` 全部变体 +
  每个脚本 `--help` 冒烟（argparse 会对 help 文本做 `%` 插值，裸 `%` 会崩）
  + `--compute-only` 重建输出不报错；需要真实请求的验证用 `--max-routes` 小样本。
- 不要读取或提交 `.env`。本地站点列表数据文件（根目录 `.txt` /
  MetroMan `.html`）已从仓库移除（6ce094f），站点目录以 `station_amap` 表为准。
- 质量工具：`analyze_group_spread.py`（组代表元精度论证）、
  `audit_routes.py`（离线审计：三角形不等式/方向不对称/组内偏移）。
