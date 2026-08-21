# HTTP 前后端接口协议

## 1. 适用范围

HTTP 只用于前端发起操作和查询历史记录；硬盘状态、任务进度和统计数据由 [WebSocket前后端通讯协议.md](WebSocket前后端通讯协议.md) 实时推送。

导出、导入和失败重试均为后端自动执行，不提供启动或重试接口。

## 2. 通用规则

- 请求和响应均使用 JSON；
- 成功响应：`{"data": ...}`；
- 失败响应：`{"error":{"code":"错误码","message":"错误说明"}}`；
- 分页查询参数：`page` 从 `1` 开始，默认 `1`；`page_size` 默认 `20`，最大 `100`；
- 分页响应：`{"data":{"items":[],"page":1,"page_size":20,"total":0}}`。

## 3. Edge 接口

### 3.1 查询导出记录

`GET /api/export-records`

可选查询参数：`disk_id`、`source_bucket`、`source_key`、`page`、`page_size`。

每条记录返回：`disk_id`、来源桶、来源对象路径、ETag、原文件大小、盘上 ZIP 路径、导出状态、写入时间、封盘确认时间。

## 4. Center 接口

### 4.1 格式化并初始化

`POST /api/disks/{device_name}/format-and-initialize`

请求：`{"confirmed":true}`。

仅用于非 ext4 硬盘。后端格式化 ext4、生成新的 `disk_id`、写入 `disk_info.json` 并注册；成功后返回新的 `disk_id`。

### 4.2 初始化并注册

`POST /api/disks/{device_name}/initialize-and-register`

请求：`{"confirmed":true}`。

仅用于已是 ext4、但没有有效 `disk_info.json` 的硬盘。后端清空盘内数据、生成新的 `disk_id`、写入 `disk_info.json` 并注册；成功后返回新的 `disk_id`。

### 4.3 查询导入记录

`GET /api/import-records`

可选查询参数：`disk_id`、`edge_code`、`source_bucket`、`source_key`、`page`、`page_size`。

每条记录返回：`disk_id`、来源 Edge 编码、来源桶、来源对象路径、ETag、原文件大小、归档桶、归档对象路径、导入时间。

### 4.4 查询已注册 Edge

`GET /api/edge-sites`

返回已注册站点的站点名称、站点编码和创建时间；不返回 Key。

### 4.5 注册 Edge

`POST /api/edge-sites`

请求：

```json
{
  "edge_name": "beijing",
  "edge_code": "EDGE-001",
  "key": "abc123"
}
```

`edge_code` 已存在时返回冲突错误。注册成功后不允许修改站点名称、编码或 Key；响应中不返回 Key。

## 5. 常用错误码

| HTTP 状态 | `code` | 含义 |
|---|---|---|
| `400` | `INVALID_REQUEST` | 请求字段缺失或不符合当前硬盘状态 |
| `404` | `DISK_NOT_FOUND` | 当前未发现指定硬盘 |
| `409` | `EDGE_CODE_EXISTS` | Edge 编码已注册 |
| `409` | `DISK_STATE_CONFLICT` | 硬盘状态不允许当前操作 |
| `500` | `INTERNAL_ERROR` | 后端执行失败，查看 `message` |
