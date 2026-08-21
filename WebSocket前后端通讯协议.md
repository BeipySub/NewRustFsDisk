# WebSocket 前后端通讯协议

## 1. 适用范围

Edge 和 Center 后端都使用本协议与各自前端页面通讯。

- Edge：硬盘插拔、硬盘空间、导出任务实时状态、已导出对象总数和已导出数据总量；
- Center：硬盘插拔、硬盘空间、导入任务实时状态、已导入对象总数和已导入数据总量。

前端连接本机部署的后端：`GET /ws`。

WebSocket 仅用于后端向前端推送状态；格式化、初始化和注册等操作由 HTTP 接口触发。

## 2. 通用规则

- 连接成功或断线重连后，后端先发送一次完整 `snapshot`；
- 本地硬盘每秒扫描一次；硬盘列表变化时发送完整 `disks_updated`；
- 任务每完成一个对象或任务状态变化时发送 `task_updated`；导出或导入进行中，每秒额外发送一次 `task_updated`；
- 导出或导入进行中，每秒发送一次 `disks_updated`，刷新硬盘实际已用、剩余和可写空间；
- Edge 的已导出对象总数或已导出数据总量变化时发送 `edge_summary_updated`；
- Center 的已导入对象总数或已导入数据总量变化时发送 `center_summary_updated`；
- 前端收到 `disks_updated` 后，以完整列表刷新硬盘卡片区域；因此多块硬盘同时插入会一次展示全部，拔出后对应卡片自动消失。

所有消息使用 JSON：

```json
{
  "event": "事件名称",
  "data": {}
}
```

## 3. 硬盘列表

### 3.1 `snapshot`

连接或重连时发送：

```json
{
  "event": "snapshot",
  "data": {
    "side": "EDGE 或 CENTER",
    "disks": [],
    "tasks": [],
    "summary": {}
  }
}
```

Edge 的 `summary`：

```json
{
  "exported_object_count": 1250,
  "exported_bytes": 1073741824000
}
```

Center 的 `summary`：

```json
{
  "imported_object_count": 1250,
  "imported_bytes": 1073741824000
}
```

### 3.2 `disks_updated`

硬盘插入、拔出、格式化、初始化、封盘状态变化或空间变化时发送完整硬盘列表：

```json
{
  "event": "disks_updated",
  "data": {
    "disks": [
      {
        "device_name": "sdb1",
        "mount_path": "/mnt/rustfs-transfer/sdb1",
        "filesystem": "ext4",
        "disk_id": "550e8400-e29b-41d4-a716-446655440000",
        "transport_state": "READY",
        "edge_code": null,
        "object_count": 0,
        "total_bytes": 1073741824000,
        "used_bytes": 0,
        "free_bytes": 1073741824000,
        "reserve_bytes": 8589934592,
        "writable_bytes": 1065151889408,
        "disk_info": {},
        "actions": ["初始化并注册"]
      }
    ]
  }
}
```

字段说明：

| 字段 | 含义 |
|---|---|
| `device_name` / `mount_path` | 当前硬盘设备名和挂载位置，用于前端定位硬盘卡片 |
| `filesystem` | 当前文件系统；未格式化或不是 ext4 时前端展示格式化操作 |
| `disk_id` | 盘内存在有效 `disk_info.json` 时的 `disk.disk_id`；否则为 `null` |
| `transport_state` | `READY`、`WRITING`、`SEALED`；未初始化时为 `null` |
| `edge_code` / `object_count` | 当前盘来源 Edge 和封盘对象数；无封盘信息时为 `null` 或 `0` |
| `total_bytes` / `used_bytes` / `free_bytes` | 硬盘实际总容量、已用空间和剩余空间 |
| `reserve_bytes` / `writable_bytes` | Edge 导出时的运输协议预留空间与当前可写空间；Center 可为 `null` |
| `disk_info` | 完整 `disk_info.json` 内容；未初始化时为 `null` |
| `actions` | 当前硬盘允许前端展示的操作，例如 `格式化并初始化`、`初始化并注册`；前端通过对应 HTTP 接口执行 |

## 4. 任务状态

### 4.1 `task_updated`

Edge 导出或 Center 导入任务变化时发送：

```json
{
  "event": "task_updated",
  "data": {
    "task_type": "EXPORT 或 IMPORT",
    "disk_id": "550e8400-e29b-41d4-a716-446655440000",
    "state": "当前任务状态",
    "object_count": 1250,
    "processed_count": 500,
    "total_bytes": 1073741824000,
    "processed_bytes": 429496729600,
    "current_file_name": "source-bucket/2026/08/a.mp4",
    "current_file_size": 1073741824,
    "current_file_transferred_bytes": 536870912,
    "speed_bytes_per_second": 104857600,
    "estimated_finished_at": "2026-08-20T12:30:00Z",
    "error_message": null
  }
}
```

| 后端 | `task_type` | 前端展示 |
|---|---|---|
| Edge | `EXPORT` | 导出状态、当前文件已传输字节数、已写入对象数和数据量、速度、错误 |
| Center | `IMPORT` | 导入状态、进度、当前文件名、速度、预计完成时间和错误 |

### 4.2 `center_summary_updated`

仅 Center 在成功导入对象后发送：

```json
{
  "event": "center_summary_updated",
  "data": {
    "imported_object_count": 1250,
    "imported_bytes": 1073741824000
  }
}
```

`imported_bytes` 按解压后的原文件大小累计。

### 4.3 `edge_summary_updated`

仅 Edge 在封盘确认导出后发送：

```json
{
  "event": "edge_summary_updated",
  "data": {
    "exported_object_count": 1250,
    "exported_bytes": 1073741824000
  }
}
```

`exported_bytes` 按原文件大小累计。

## 5. 前端处理

| 事件 | 处理方式 |
|---|---|
| `snapshot` | 初始化整个页面状态 |
| `disks_updated` | 用完整硬盘列表刷新硬盘卡片、运输盘列表和当前选中硬盘详情 |
| `task_updated` | 刷新对应导出或导入任务的进度、当前文件名、速度、预计完成时间和错误 |
| `edge_summary_updated` | 刷新 Edge 首页的已导出对象总数和已导出数据总量 |
| `center_summary_updated` | 刷新 Center 首页的已导入对象总数和已导入数据总量 |
