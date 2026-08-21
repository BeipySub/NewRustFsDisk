# Center 端数据库设计

## 1. 范围

Center 数据库保存站点配置、运输盘 `disk_id` 使用状态、导入任务和已归档对象。

运输盘当前状态不保存在数据库中，以盘上的 `disk_info.json` 为准。`disk.disk_id` 代表一次运输，不建立物理硬盘登记表。

## 2. 表关系

```text
edge_sites 1 ── N import_tasks 1 ── N import_objects
disk_ids   1 ── 0..1 import_tasks
```

## 3. `edge_sites`：Edge 站点

| 字段 | 类型 | 规则 | 含义 |
|---|---|---|---|
| `id` | UUID | 主键 | 站点记录编号 |
| `edge_name` | 字符串 | 必填 | 站点名称 |
| `edge_code` | 字符串 | 必填、唯一 | 站点编码，也是 Center RustFS 归档桶名 |
| `key` | 字符串 | 必填 | 中控分配给 Edge 的 Key |
| `created_at` | 时间 | 必填 | 创建时间 |
| `updated_at` | 时间 | 必填 | 最后修改时间 |

约束：`edge_code` 不能重复。

## 4. `import_tasks`：导入任务

一条记录代表一次运输盘导入任务。运输盘导入失败后重试，继续使用原任务记录。

| 字段 | 类型 | 规则 | 含义 |
|---|---|---|---|
| `disk_id` | UUID | 主键 | 盘上 `disk.disk_id`，即本次运输编号 |
| `edge_code` | 字符串 | 必填、索引 | 来源站点编码 |
| `state` | 字符串 | 必填 | `待导入`、`导入中`、`导入成功`、`导入失败` |
| `object_count` | 整数 | 必填 | 清单对象总数 |
| `processed_count` | 整数 | 必填，默认 0 | 已处理对象数 |
| `total_bytes` | 大整数 | 必填 | 清单中原文件总大小 |
| `processed_bytes` | 大整数 | 必填，默认 0 | 已处理的原文件大小 |
| `current_file_name` | 文本 | 可空 | 当前正在导入的来源文件路径 |
| `estimated_finished_at` | 时间 | 可空 | 根据当前导入速度计算的预计完成时间 |
| `error_message` | 文本 | 可空 | 最近一次失败原因 |
| `queued_at` | 时间 | 必填 | 发现 `SEALED` 盘并进入队列的时间 |
| `started_at` | 时间 | 可空 | 开始导入时间 |
| `finished_at` | 时间 | 可空 | 导入成功或失败时间 |
| `created_at` | 时间 | 必填 | 任务创建时间 |
| `updated_at` | 时间 | 必填 | 最后修改时间 |

约束：同一个 `disk_id` 只保留一条导入任务，用于查询本次运输的完整导入结果。

## 5. `import_objects`：对象账本

只在对象已成功写入 Center RustFS 后新增记录。导入重试时，已存在的对象账本记录直接跳过。

| 字段 | 类型 | 规则 | 含义 |
|---|---|---|---|
| `id` | UUID | 主键 | 对象账本编号 |
| `disk_id` | UUID | 必填、索引 | 首次导入该对象的运输编号 |
| `edge_code` | 字符串 | 必填 | 来源站点编码 |
| `source_bucket` | 字符串 | 必填 | 来源 RustFS 桶名 |
| `source_key` | 文本 | 必填 | 来源对象路径 |
| `etag` | 字符串 | 必填 | 对象 ETag |
| `size` | 大整数 | 必填 | 解压后原文件大小，用于统计数据量 |
| `plain_sha256` | 字符串 | 必填 | 解压后校验通过的 SHA256 |
| `target_bucket` | 字符串 | 必填 | Center 归档桶，等于 `edge_code` |
| `target_key` | 文本 | 必填 | Center 对象路径：`source_bucket/source_key` |
| `imported_at` | 时间 | 必填 | 成功归档时间 |

唯一约束：`edge_code + source_bucket + source_key + etag + size`。

索引：

- `disk_id`：查询某次运输已导入的数据；
- `edge_code`：查询某个站点的归档数据；
- `target_bucket + target_key`：查询 Center 中的归档位置。

## 6. `disk_ids`：运输盘 `disk_id` 管理

| 字段 | 类型 | 规则 | 含义 |
|---|---|---|---|
| `disk_id` | UUID | 主键 | 每次初始化或注册生成的运输编号；一个 ID 只能使用一次 |
| `device_path` | 字符串 | 可空 | 初始化时的块设备路径，例如 `/dev/sdb` |
| `disk_sn` | 字符串 | 可空、不校验 | 当前硬盘读取到的 SN；部分硬盘可能无法读取 |
| `vendor` | 字符串 | 可空 | 硬盘厂商 |
| `model` | 字符串 | 可空 | 硬盘型号 |
| `transport` | 字符串 | 可空 | 连接方式，例如 USB、SATA、NVMe |
| `size_bytes` | 大整数 | 必填、仅记录 | 硬盘总容量（字节） |
| `filesystem` | 字符串 | 可空 | 初始化时的文件系统，完成格式化后为 ext4 |
| `filesystem_uuid` | 字符串 | 可空 | 文件系统 UUID |
| `mount_path` | 字符串 | 可空 | 初始化时的挂载路径 |
| `hardware_info` | JSON | 可空 | Linux 当前可读取到的完整硬盘信息原始数据 |
| `status` | 字符串 | 必填 | `FREE`：已初始化可交给 Edge；`USED`：该 ID 的盘内数据已全部导入，禁止再次使用 |
| `initialized_at` | 时间 | 必填 | 该 `disk_id` 初始化时间 |
| `used_at` | 时间 | 可空 | 状态变为 `USED` 的时间 |

规则：

- 初始化或重新初始化时，新增一个 `disk_id`，状态为 `FREE`；
- 同一块物理硬盘可反复初始化，每次产生新的 `disk_id`；
- 除 `disk_id`、`size_bytes`、`status`、`initialized_at` 外，其余硬盘字段读取不到时允许为空；
- `disk_sn`、设备路径、容量、文件系统 UUID 等硬盘信息仅用于展示和查询，不作为注册、校验或导入依据；
- 全部对象导入完成后，将该 `disk_id` 更新为 `USED`；该 ID 不得再次导出或导入。
