# 古籍拓片缺损修补API

纯后端零依赖Node服务，使用 `data/db.json` 持久化拓片、缺损项和修补批次。

## 启动

```bash
PORT=3020 node server.js
```

## 主要接口

- `GET /health`
- `GET /rubbings`
- `POST /rubbings`
- `GET /rubbings/:id/damages`
- `POST /rubbings/:id/damages`
- `GET /damages?status=&type=`
- `PATCH /damages/:id`
- `GET /batches`
- `POST /batches`
- `GET /batches/:id`
- `POST /batches/:id/complete`
- `POST /batches/:id/transfers` — 把本批次（源）的单条缺损移交到另一未结批次
- `GET /transfers?damageId=&batchId=&rubbingId=` — 只读移交记录

## 闭环示例

```bash
curl http://127.0.0.1:3020/damages?status=pending
curl -X POST http://127.0.0.1:3020/batches \
  -H 'Content-Type: application/json' \
  -d '{"name":"六月小批修补","damageIds":["damage_demo_1","damage_demo_2"]}'
```

## 缺损移交闭环

同一拓片内，未结批次之间可以移交单条缺损。源批次移除该缺损，目标批次接收，缺损创建时间与已有修补结果保留。

```bash
curl -X POST http://127.0.0.1:3020/batches/<源批次id>/transfers \
  -H 'Content-Type: application/json' \
  -d '{"targetBatchId":"<目标批次id>","damageId":"damage_demo_1","reason":"虫蛀需并入专项批次"}'
curl 'http://127.0.0.1:3020/transfers?batchId=<批次id>'
```

以下情况整单返回 `409` 且数据不变：

- 源批次或目标批次已结项
- 缺损已修复（`status=repaired`）
- 跨拓片移交（目标批次归属另一拓片）
- 目标批次已包含该缺损
- 缺损不属于源批次

批次列表（`GET /batches`、`GET /batches/:id`）的缺损与统计均按实际归属（`damageIds`）计算，并内嵌只读 `transfers` 记录（`direction=in/out`）。并发重复移交仅一次成功；旧批次缺少移交字段时按空数组处理，全部结果落盘保留。
