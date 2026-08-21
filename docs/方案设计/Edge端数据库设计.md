# Edge 端数据库设计

## 1. 目标

Edge 数据库只保存导出任务和导出账本，用于队列恢复、写盘恢复、导出记录查询和页面统计。

运输盘当前状态不保存在数据库中，以盘上的 `disk_info.json` 为准。`disk.disk_id` 代表一次运输，不建立物理硬盘登记表。

站点名称、站点编码、Key 和 RustFS 连接信息由首次部署配置提供，不写入数据库。

## 1.1 启动时表结构检查

Edge 主程序连接数据库后，按本文的 `export_tasks`、`export_objects` 模型检查数据表、字段、唯一约束和索引。

- 缺失结构由程序自动创建；空数据库可直接完成初始化；
- 已有表只补齐缺失结构，不删除数据、不重建表；
- 已存在字段与本文类型或约束冲突时，启动失败并报告冲突项，不自动修改已有字段。

表结构检查完成前，不启动硬盘扫描和导出任务。

## 2. 表关系

```text
export_tasks  1 ── N  export_objects
```

一条 `export_tasks` 记录对应一个 `disk_id` 的一次导出。刚初始化、但还没有待导出对象的运输盘没有导出任务。

## 3. `export_tasks`：导出任务

扫描到 `READY` 运输盘时创建导出任务；若扫描后没有待导出对象，删除该任务，运输盘保持 `READY`。

| 字段 | 类型 | 规则 | 含义 |
|---|---|---|---|
| `disk_id` | UUID | 主键 | 盘上 `disk.disk_id`，即本次运输编号 |
| `state` | 字符串 | 必填 | `待导出`、`导出中`、`等待恢复`、`已封盘`、`导出失败` |
| `object_count` | 整数 | 必填，默认 `0` | 本次运输盘已写入对象数 |
| `exported_bytes` | 大整数 | 必填，默认 `0` | 本次运输盘已写入对象大小 |
| `current_file_name` | 文本 | 可空 | 当前正在导出的来源对象路径 |
| `error_message` | 文本 | 可空 | 最近一次失败原因 |
| `queued_at` | 时间 | 必填 | `READY` 盘进入导出队列的时间 |
| `started_at` | 时间 | 可空 | 首次开始写盘时间 |
| `finished_at` | 时间 | 可空 | 封盘成功或最近一次导出失败时间 |

约束：同一个 `disk_id` 最多保留一条导出任务。导出失败后重试，继续使用原任务记录；封盘成功后任务状态为“已封盘”。

## 4. `export_objects`：导出账本

每条记录代表一个对象版本在某个 `disk_id` 中的写入记录。

| 字段 | 类型 | 规则 | 含义 |
|---|---|---|---|
| `id` | UUID | 主键 | 导出对象记录编号 |
| `disk_id` | UUID | 必填、索引 | 写入该对象的运输编号 |
| `source_bucket` | 字符串 | 必填 | 来源 RustFS 桶名 |
| `source_key` | 文本 | 必填 | 来源对象路径 |
| `etag` | 字符串 | 必填 | 对象 ETag |
| `size` | 大整数 | 必填 | 原对象大小 |
| `plain_sha256` | 字符串 | 必填 | 导出时计算的原对象 SHA256 |
| `zip_path` | 文本 | 必填 | 盘上 ZIP 相对路径，格式由运输硬盘通讯协议定义 |
| `state` | 字符串 | 必填 | `WRITTEN`：已写入当前盘；`CONFIRMED`：该盘已封盘，确认导出 |
| `written_at` | 时间 | 必填 | 对象 ZIP 完整写入时间 |
| `confirmed_at` | 时间 | 可空 | 封盘确认时间 |

唯一约束：`source_bucket + source_key + etag + size`。

对象版本标识为：`source_bucket + source_key + etag + size`。`CONFIRMED` 记录用于后续 S3 全量扫描时跳过已导出对象；当前 `disk_id` 的 `WRITTEN` 记录用于恢复写盘时跳过已写入当前盘的对象。

## 5. 写入与恢复规则

1. `READY` 盘进入队列时，创建 `export_tasks`，状态为“待导出”。
2. 确认存在待导出对象后，任务状态改为“导出中”。
3. 每个对象 ZIP 完整写入运输盘且 ETag、大小复核一致后，新增 `export_objects`，状态为 `WRITTEN`；同时更新任务进度。
4. 硬盘拔出时，任务状态改为“等待恢复”。同一 `disk_id` 重新插入后，按该盘 `WRITTEN` 记录恢复。
5. 恢复时，盘上 ZIP 没有对应账本记录则删除 ZIP；账本记录没有对应盘上 ZIP 则删除账本记录。
6. 封盘成功后，将当前 `disk_id` 的全部 `WRITTEN` 记录更新为 `CONFIRMED`，并更新任务为“已封盘”。
7. 对象读取失败时，任务更新为“导出失败”；硬盘仍挂载时，60 秒后自动继续使用该任务和现有 `WRITTEN` 记录。

## 6. 页面查询来源

| 页面数据 | 查询来源 |
|---|---|
| 导出队列和实时导出记录 | `export_tasks` |
| 当前已写入数量、数据量、文件名、错误 | `export_tasks` |
| 已确认导出对象总数 | `export_objects` 中 `state = CONFIRMED` 的记录数 |
| 已确认导出数据总量 | `SUM(export_objects.size)`，仅统计 `state = CONFIRMED` |
| 指定运输盘的导出数据 | 按 `export_objects.disk_id` 查询 |

## 7. 与其他文档的关系

- 运输盘目录、`disk_info.json`、对象 ZIP 路径和封盘规则见 [运输硬盘通讯协议.md](运输硬盘通讯协议.md)；
- Edge 导出流程、队列和失败恢复见 [Edge端设计说明.md](Edge端设计说明.md)；
- WebSocket 推送字段见 [WebSocket前后端通讯协议.md](WebSocket前后端通讯协议.md)。
