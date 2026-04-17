# API Contracts: 磁盘容量预测

## 预测概览列表

```
GET /inspection/prediction/list
```

**Params**:
- `instanceIp` (optional) — 按实例 IP 筛选
- `mountPoint` (optional) — 按挂载点筛选
- `riskLevel` (optional) — 按风险等级筛选（high/medium/low）

**Response**: `R<List<DiskPredictionVo>>`

---

## 预测详情

```
GET /inspection/prediction/detail
```

**Params**:
- `instanceIp` (required) — 实例 IP
- `mountPoint` (required) — 挂载点

**Response**: `R<DiskPredictionVo>` — 含完整 predictions 列表

---

## 历史数据查询（趋势图用）

```
GET /inspection/prediction/history
```

**Params**:
- `instanceIp` (required) — 实例 IP
- `mountPoint` (required) — 挂载点
- `days` (optional, default 30) — 查询最近 N 天的数据

**Response**: `R<List<DiskUsageHistoryVo>>`

---

## 权限标识

| 接口 | 权限 |
|---|---|
| /list | `inspection:prediction:query` |
| /detail | `inspection:prediction:query` |
| /history | `inspection:prediction:query` |

所有接口需要 `@SaCheckPermission("inspection:prediction:query")`
