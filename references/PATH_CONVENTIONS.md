# 路径 / 参数 / 响应命名规范

## 一、路径命名规范

### 基本原则

| 原则 | 正确示例 | 错误示例 |
|------|---------|---------|
| 资源路径用复数名词 | `/logical-models` | `/logical-model` |
| 子资源嵌套在父资源下 | `/logical-models/{tableName}/versions` | `/logical-model-versions` |
| 路径参数用 `{}` 包裹 | `/{id}` | `?id=` |
| 版本路径放在资源后 | `/{id}/versions/{version}` | `/versions/{id}/{version}` |
| 差异比对用 query string | `/{id}/versions/diff?from=&to=` | `/{id}/diff-version` |

### 路径结构速查

| 业务场景 | 路径 |
|---------|------|
| 列表查询（分页） | `GET /resources` |
| 详情查询 | `GET /resources/{id}` |
| 子资源列表 | `GET /resources/{id}/sub-resources` |
| 版本历史 | `GET /resources/{id}/versions` |
| 指定版本详情 | `GET /resources/{id}/versions/{version}` |
| 版本差异比对 | `GET /resources/{id}/versions/diff?from=&to=` |
| 批量模式（特殊） | `GET /resources/_batch?filter=` |
| 树形结构根节点 | `GET /resources/tree` |

### 批量/特殊模式

用下划线前缀 `_batch` 表示非标准资源操作：

```
GET /resources/_batch/sub?catalogIdPath=xxx&direction=upstream
```

### 过滤参数命名

| 参数类型 | 命名 | 示例 |
|---------|------|------|
| 精确匹配 | `fieldName` | `jobInsId=xxx` |
| 范围查询（起止） | `startField` / `endField` | `startTime=` / `endTime=` |
| 递归过滤 | `fieldPath` | `catalogIdPath=cat_001` |
| 区间 | `minField` / `maxField` | `minScore=` / `maxScore=` |
| 分页 | `pageNum` / `pageSize` | `pageNum=2&pageSize=20` |

---

## 二、参数命名规范

### 通用参数

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页最大 500，默认 20 |

### 参数 in 类型选择

| in 类型 | 使用场景 |
|--------|---------|
| `query` | 过滤、分页、排序等可选参数 |
| `path` | 必选的资源标识（如 `{id}`） |
| `header` | 认证token、租户ID等全局参数 |
| `body` | POST 请求体（避免使用，用 JSON 替代） |

### 参数描述规范

```yaml
- name: startTime
  in: query
  schema:
    type: string
    format: date-time
  description: 开始时间（ISO8601，如 2026-05-01T00:00:00Z）
```

---

## 三、响应结构规范

### 通用响应包装（必须）

```json
{
  "code": "success",
  "msg": "Query success",
  "data": { ... }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| code | string | 状态码：`success` / `fail` |
| msg | string | 状态消息 |
| data | object/array | 业务数据 |

### 分页响应结构

```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [...],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 156,
      "pages": 8
    }
  }
}
```

### 错误响应结构

```json
{
  "code": "fail",
  "msg": "具体错误原因：tenantId is required"
}
```

---

## 四、Schema 命名规范

| Schema 类型 | 命名格式 | 示例 |
|-----------|---------|------|
| 单个资源 | `<Resource>` | `LogicalModel` |
| 列表响应 | `<Resource>ListResponse` | `LogicalModelListResponse` |
| 详情响应 | `<Resource>DetailResponse` | `LogicalModelDetailResponse` |
| 分页结构 | `PaginationInfo` | — |
| 通用响应 | `ApiResponse` | — |
| 错误响应 | `ErrorResponse` | — |

---

## 五、operationId 命名规范

| 操作 | operationId 格式 | 示例 |
|------|----------------|------|
| 列表查询 | `query<Resource>s` | `queryLogicalModels` |
| 详情查询 | `query<Resource>ById` | `queryLogicalModelById` |
| 创建 | `create<Resource>` | `createLogicalModel` |
| 更新 | `update<Resource>` | `updateLogicalModel` |
| 删除 | `delete<Resource>` | `deleteLogicalModel` |
| 版本历史 | `query<Resource>Versions` | `queryLogicalModelVersions` |
| 差异比对 | `diff<Resource>Versions` | `diffLogicalModelVersions` |
| 血缘 | `query<Resource>Lineage` | `queryLogicalModelLineage` |

---

## 六、Tag 命名规范

Tag 与路径分组一一对应，用复数名词：

```yaml
tags:
  - name: logical-models
  - name: data-quality
  - name: tmf-sid-catalogs
```