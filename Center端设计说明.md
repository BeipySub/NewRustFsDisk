# Center 端设计说明

## 1. 职责

Center 端负责运输盘初始化、Edge 站点管理、封盘导入、导入记录查询和导入成功后的运输盘重新初始化。

数据流：

```text
已封盘运输盘 → Center 校验与导入 → Center RustFS
```

Center 不向 Edge 发起网络请求；只处理插入本机的运输盘。

## 2. 已确定规则

- 一个 Edge 对应一个中控 RustFS 归档桶，桶名为 `edge_code`；
- 导入时桶存在则复用，不存在则新建；
- 目标对象键为 `source_bucket/source_key`；运输盘文件路径以 [运输硬盘通讯协议.md](运输硬盘通讯协议.md) 为准，Center 按清单中的来源桶和来源路径保持 Edge 的存储结构；
- 运输盘必须为 `SEALED` 才能导入；
- 找不到运输盘中 `edge_code` 对应的站点配置和固定 Key 时，拒绝导入；
- 导入成功的判断：全部对象完成解压、SHA256 校验、写入 RustFS 且对象账本落库；
- 导入成功后自动重新初始化运输盘，生成新的 `disk.disk_id`，删除 `manifest.zip` 和 `data/`；
- 文件系统不是 ext4 时，操作员确认后格式化为 ext4；
- Center 仅管理本机插入的硬盘；公共挂载规则见 [开发设计说明.md](开发设计说明.md)。

## 3. Module 划分

```text
Center 主程序
├── 本地硬盘挂载与目录扫描 Module（两端共用）
├── Edge 站点 Module
├── 运输盘初始化 Module
├── 导入任务 Module
│   ├── 运输盘协议 Module（两端共用）
│   ├── Center RustFS Module
│   └── 对象账本 Module
├── 任务与状态 Module
└── WebSocket 推送 Module
```

### 3.1 Edge 站点 Module

**职责**：维护站点名称、`edge_code` 和固定 Key。

**规则**：`edge_code` 唯一，且是该站点的 Center 归档桶名。

### 3.2 运输盘初始化 Module

**职责**：识别文件系统、格式化 ext4、创建或重新创建 `disk_info.json`。

**结果**：初始化后运输盘状态为 `READY`，`disk.disk_id` 为新 UUID，`edge` 为 `null`。

### 3.3 导入任务 Module

**职责**：读取已封盘运输盘，校验、解密并归档对象。

**任务状态**：`待导入 → 导入中 → 导入成功 / 导入失败`。

**多盘规则**：扫描到多块运输盘时，仅 `SEALED` 状态的盘进入导入队列；创建任务时记录 `queued_at`，按 `queued_at` 从早到晚逐块导入，同一时间只导入一块盘；Center 重启后继续按该顺序执行。

**失败规则**：运输盘保持 `SEALED`；下一轮扫描自动重试导入。

### 3.4 对象账本 Module

**职责**：记录对象已成功归档，防止重复写入。

**对象唯一标识**：`edge_code + source_bucket + source_key + etag + size`。

## 4. 导入流程

1. 本地硬盘挂载与目录扫描 Module 发现运输盘；多块盘同时存在时，`SEALED` 状态的盘创建导入任务并记录 `queued_at`，按该时间进入导入队列。
2. 读取 `disk_info.json`；仅 `state = SEALED` 的盘进入待导入状态。
3. 查询 `disk_ids`：必须存在盘中 `disk.disk_id` 且状态为 `FREE` 的记录，否则拒绝导入。
4. 使用 `edge.edge_code` 查找站点配置和固定 Key；找不到则拒绝导入。
5. 查找同名归档桶：存在则复用，不存在则创建。
6. 校验 `manifest.zip` 的 SHA256 是否等于 `edge.manifest_sha256`。
7. 按运输硬盘通讯协议的 ZIP 密码规则解密 `manifest.zip`，再以流式方式读取其中的 `manifest.jsonl`，逐行解析，不将明文清单写入硬盘。
8. 逐条处理对象：
   - 用对象唯一标识查询对象账本；已归档则跳过写入，但仍计入已处理对象数和已处理数据量；
   - 解密对应对象 ZIP，读取协议规定的原对象名文件；
   - 校验解压后的 SHA256 等于 `plain_sha256`；
   - 写入 Center RustFS 的 `edge_code` 桶，目标键为 `source_bucket/source_key`；
   - 写入对象账本。
9. 运输盘内全部对象成功后，将任务标记为导入成功，并将该 `disk_id` 更新为 `USED`。
10. 自动重新初始化运输盘，使其恢复 `READY`，供 Edge 再次使用。

### 4.1 数据库写入时机

1. 初始化或注册运输盘：新增 `disk_ids`，状态为 `FREE`。
2. 发现 `SEALED` 运输盘：创建或读取 `import_tasks`，状态为“待导入”。
3. 开始导入：更新任务为“导入中”。
4. 每个清单对象处理完成后（包括已归档而跳过的对象）：更新任务已处理数量和大小；实际写入 Center RustFS 成功的对象再新增 `import_objects`。
5. 运输盘内全部对象完成：更新任务为“导入成功”，将对应 `disk_id` 更新为 `USED`。
6. 任一步失败：更新任务为“导入失败”和错误信息；运输盘保持 `SEALED`，下一轮扫描自动重试。

## 5. 数据库

数据库表结构、字段和写入顺序见 [Center端数据库设计.md](Center端数据库设计.md)。

## 6. 页面

| 页面 | 首期内容 |
|---|---|
| Edge 站点管理 | 新增、编辑、查看站点名称、编码和固定 Key |
| 首页 | 包含运输盘列表、硬盘卡片和运输盘详情；实时导入记录展示任务进度、当前文件名和预计完成时间，并显示已导入对象总数和已导入数据总量（按解压后原文件大小计算） |
| 导入记录 | 按 `disk_id`、Edge 或对象查询归档结果 |

硬盘卡片展示当前硬盘状态。硬盘不是 ext4 时，展示“格式化并初始化”按钮；操作员确认后，Center 格式化为 ext4、初始化并注册该硬盘。硬盘已经是 ext4 但没有有效 `disk_info.json` 时，展示“初始化并注册”按钮：删除盘内现有内容，写入新的 `disk_info.json`，不再格式化。

首页通过两端共用的 WebSocket 页面通讯 Module 接收硬盘发现、插拔和导入任务变化，并实时刷新导入记录与统计数据。具体事件和字段见 [WebSocket前后端通讯协议.md](WebSocket前后端通讯协议.md)。

### 6.1 首页展示数据

| 首页数据 | 查询来源 |
|---|---|
| 实时导入记录 | `import_tasks` 的任务状态、进度、当前文件名和预计完成时间 |
| 已导入对象总数 | `import_objects` 记录数 |
| 已导入数据总量 | `SUM(import_objects.size)`，即解压后原文件大小 |
| 指定运输盘已导入数据 | 按 `import_objects.disk_id` 查询 |

## 7. 部署配置

| 配置 | 用途 |
|---|---|
| `database` | Center 数据库连接信息 |
| `rustfs` | Center RustFS 连接信息 |

公共挂载、扫描和 Docker 配置见 [开发设计说明.md](开发设计说明.md)；Center 额外需要读取文件系统类型、格式化 ext4 和重新挂载的权限。

## 8. 与运输盘协议的关系

Center 端只按 [运输硬盘通讯协议.md](运输硬盘通讯协议.md) 读取和写入运输盘。盘上文件格式、ZIP 密码和状态规则以该协议为准。
