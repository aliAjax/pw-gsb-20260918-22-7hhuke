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
- `POST /batches/:id/transfer`：把源批次内单条缺损移交到同拓片的另一未结批次
- `GET /batches/:id/transfers`：只读查询某批次相关的移交记录（出/入均含）
- `GET /transfers?damageId=&batchId=`：只读查询全局移交记录
- `POST /batches/:id/complete`

## 缺损移交闭环

同一拓片内，可将一个**未结批次**中的单条缺损移交到另一个**未结批次**：源批次移除该缺损、目标批次接收，缺损的创建时间与已有修补结果（照片、备注、修复时间等）原样保留，只变更批次归属，并追加一条只读移交记录。

```bash
curl -X POST http://127.0.0.1:3020/batches/<源批次id>/transfer \
  -H 'Content-Type: application/json' \
  -d '{"damageId":"damage_demo_1","toBatchId":"<目标批次id>","reason":"师傅排期调整"}'
```

以下情况整单返回 `409 Conflict`，且不做任何变更：

- 源批次或目标批次已结项（`status !== "open"`）
- 缺损已修复（`status === "repaired"`）
- 缺损不属于源批次（含并发下已被移交走的情况）
- 目标批次已包含该缺损（重复移交）
- 跨拓片移交（目标批次按其当前成员锚定拓片；空批次可接收任意拓片的首条缺损）

批次列表/详情的 `total/repaired/pending/damages` 均按**实际归属**统计（同时在批次 `damageIds` 中且缺损 `batchId` 指向该批次），移交后源批次不再统计该缺损。并发重复移交经服务端串行化处理，仅一次成功。旧数据缺少 `transfers` 集合或批次缺少 `damageIds` 时按空处理，所有写入经临时文件原子落盘。


## 闭环示例

```bash
curl http://127.0.0.1:3020/damages?status=pending
curl -X POST http://127.0.0.1:3020/batches \
  -H 'Content-Type: application/json' \
  -d '{"name":"六月小批修补","damageIds":["damage_demo_1","damage_demo_2"]}'
```
