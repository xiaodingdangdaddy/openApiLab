# 元数据管理查询接口 - API 接口文档

> 本文档基于 OpenAPI 3.1 规范编写（v2.1），包含所有接口的请求参数、响应格式、字段说明。

---

## 一、接口总览

| 方法 | 路径 | 说明 | 已有接口映射 |
|------|------|------|------------|
| GET | `/logical-models` | 逻辑模型列表（分页） | 内部封装 |
| GET | `/logical-models/{tableName}` | 逻辑模型详情（含物理模型） | `queryLogicalInfoByName` |
| GET | `/logical-models/{tableName}/lineage` | 逻辑模型血缘关系 | `queryLogicalModelsLineage` |
| GET | `/logical-models/{tableName}/versions` | 逻辑模型版本历史 | BFF 新增 |
| GET | `/logical-models/{tableName}/versions/{version}` | 指定版本详情 | BFF 新增 |
| GET | `/logical-models/{tableName}/versions/diff` | 版本差异比对 | BFF 新增 |
| GET | `/data-quality/results` | DQ 核查结果列表（分页） | 内部封装 |
| GET | `/data-quality/results/{jobInsId}` | DQ 核查结果详情 | `queryJobInstanceDetail` |
| GET | `/tmf-sid-catalogs` | TMF SID 目录树 | `dataCatalogTree` |
| GET | `/tmf-sid-catalogs/{catalogId}/models` | 目录下逻辑模型列表 | `model/list` |

---

## 二、统一规范

### 2.1 认证方式

所有接口均通过 `Authorization: Bearer <token>` Header 进行认证。

### 2.2 统一响应格式

```json
{
  "code": "success",
  "msg": "Query success",
  "data": { ... }
}
```

### 2.3 分页结构

```json
{
  "data": {
    "list": [...],
    "pagination": { "pageNum": 1, "pageSize": 20, "total": 100, "pages": 5 }
  }
}
```

---

## 三、接口详情

### 3.1 查询逻辑模型列表

**GET** `/api/v1/logical-models`

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| status | query | 否 | string | `draft` / `release` |
| billType | query | 否 | string | 单据类型 |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页最大 500 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [{ "tableName": "dim_user", "labelCh": "用户维度表", "status": "release" }],
    "pagination": { "pageNum": 1, "pageSize": 20, "total": 150, "pages": 8 }
  }
}
```

---

### 3.2 查询逻辑模型详情

**GET** `/api/v1/logical-models/{tableName}`

通过 `tableName` 查询包含物理模型映射、列定义等完整详情。

**响应示例：**
```json
{
  "code": "success",
  "data": {
    "tableName": "dim_user",
    "labelCh": "用户维度表",
    "status": "release",
    "relatedPhysicalModels": [{ "physicalModelName": "hive_dim_user", "physicalColumns": [...] }],
    "logicalColumns": [{ "columnName": "user_id", "labelCh": "用户ID", "relatedPhyColumnMapping": [...] }]
  }
}
```

---

### 3.3 查询逻辑模型血缘

**GET** `/api/v1/logical-models/{tableName}/lineage`

返回递归嵌套的前向血缘 / 后向影响链路。

---

### 3.4 查询逻辑模型版本历史

**GET** `/api/v1/logical-models/{tableName}/versions`

查询版本快照历史，按版本号倒序。

**响应示例：**
```json
{
  "code": "success",
  "data": {
    "tableName": "dim_user",
    "list": [
      { "version": 3, "changeType": "COLUMN_ADDED", "changeSummary": "新增 email_sensitive 列", "releasedAt": "2024-04-20T10:00:00Z", "releasedBy": "user@example.com" },
      { "version": 2, "changeType": "PHYSICAL_MODEL_ADDED", "changeSummary": "新增 Hive 物理表映射", "releasedAt": "2024-04-10T08:00:00Z" },
      { "version": 1, "changeType": "CREATE", "changeSummary": "初始创建", "releasedAt": "2024-03-01T09:00:00Z" }
    ],
    "pagination": { "pageNum": 1, "pageSize": 20, "total": 3 }
  }
}
```

**changeType 枚举值（实际生效的）：**

| 值 | 含义 |
|----|------|
| CREATE | 初始创建 |
| COLUMN_ADDED | 逻辑列新增 |
| PHYSICAL_MODEL_ADDED | 关联物理模型新增 |
| PHYSICAL_MODEL_DELETED | 关联物理模型删除 |
| PHYSICAL_COLUMN_ADDED | 物理表列新增 |

---

### 3.5 查询逻辑模型指定版本详情

**GET** `/api/v1/logical-models/{tableName}/versions/{version}`

返回指定版本的完整模型快照。

---

### 3.6 比对逻辑模型版本差异

**GET** `/api/v1/logical-models/{tableName}/versions/diff?from=1&to=3`

**响应示例：**
```json
{
  "code": "success",
  "data": {
    "tableName": "dim_user",
    "fromVersion": 1,
    "toVersion": 3,
    "changedFields": [
      { "path": "logicalColumns", "to": { "columnName": "email_sensitive" }, "changeType": "ADDED" }
    ],
    "summary": { "columnsAdded": 1, "physicalModelsAdded": 0, "physicalModelsDeleted": 0 }
  }
}
```

---

### 3.7 查询 DQ 核查结果列表

**GET** `/api/v1/data-quality/results`

| 参数 | 必选 | 说明 |
|------|------|------|
| tenantId | 是 | 租户ID |
| catalogIdPath | 否 | 目录ID（递归，该目录及子目录下所有模型） |
| tableCode | 否 | 模型编码 |
| startDataTime | 否 | 数据开始日期（YYYY-MM-DD） |
| endDataTime | 否 | 数据结束日期（YYYY-MM-DD） |
| status | 否 | 作业状态（success/failed/warning） |
| minScore | 否 | 最低质量评分（0-100） |
| maxScore | 否 | 最高质量评分（0-100） |
| ruleType | 否 | 规则类型（accuracy/completeness/uniqueness） |
| owner | 否 | 模型责任人 |
| pageNum / pageSize | 否 | 分页参数 |

---

### 3.8 查询 DQ 核查结果详情

**GET** `/api/v1/data-quality/results/{jobInsId}`

返回包含四类核查明细的完整结果：表间、单表、字段级、SQL级。

---

### 3.9 查询 TMF SID 目录树

**GET** `/api/v1/tmf-sid-catalogs?familyCatalogTypeId=FC001&dataTypeId=1`

**dataTypeId：** 1=Domain, 2=ABE, 3=BE, 4=业务对象, 10=逻辑模型

---

### 3.10 查询目录下逻辑模型列表

**GET** `/api/v1/tmf-sid-catalogs/{catalogId}/models?familyCatalogId=FC001&themeId=T001&searchWord=user`

支持关键词搜索，返回高亮字段 `highlightFields`。

---

## 四、版本触发规则（逻辑模型）

### 触发版本 +1 的变化

| 变化类型 | 是否 +1 版本 |
|---------|------------|
| 逻辑列新增 | ✅ |
| 关联物理模型新增 | ✅ |
| 关联物理模型删除 | ✅ |
| 物理表列新增 | ✅ |

### 系统不允许的操作（不涉及版本变化）

| 操作 | 说明 |
|------|------|
| 逻辑列删除 | 系统不允许 |
| 物理列删除 | 系统不允许 |
| 列重命名 | 系统不允许 |
| 列数据类型变化 | 系统不允许 |
| 列映射表达式变化 | 系统不允许 |
| `status` 状态发布变化 | 系统不允许 |
| `openFlag` 开放标识变化 | 系统不允许 |
| 别名/描述修改 | 静默保存 |

---

## 五、数据质量说明

不做版本变化追踪。每次 DQ 核查运行生成唯一的 `jobInsId`，即一个核查结果快照。直接使用 `score` 字段作为模型质量评分指标。
