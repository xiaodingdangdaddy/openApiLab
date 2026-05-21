# 元数据管理查询接口 - API 接口文档

> 本文档基于 OpenAPI 3.1 规范编写（v2.0），包含所有接口的请求参数、响应格式、字段说明。

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
| GET | `/data-quality/results/{jobInsId}/history` | 同一模型历史结果 | BFF 新增 |
| GET | `/data-quality/results/{jobInsId}/diff` | 核查结果差异比对 | BFF 新增 |
| GET | `/tmf-sid-catalogs` | TMF SID 目录树 | `dataCatalogTree` |
| GET | `/tmf-sid-catalogs/{catalogId}/models` | 目录下逻辑模型列表 | `model/list` |

---

## 二、统一规范

### 2.1 认证方式

所有接口均通过 `Authorization: Bearer <token>` Header 进行认证。

```
Authorization: Bearer 6eab8b...d7dd
```

### 2.2 通用请求 Header

| Header | 值 | 必选 | 说明 |
|--------|------|------|------|
| Authorization | Bearer + token | 是 | 认证令牌 |
| tenantId | 租户ID | 部分接口在 query 中传 | |

### 2.3 统一响应格式

```json
{
  "code": "success",
  "msg": "Query success",
  "data": { ... }
}
```

**code 取值：**
- `success`：成功
- `fail`：失败

### 2.4 分页结构

```json
{
  "code": "success",
  "data": {
    "list": [ ... ],
    "pagination": { "pageNum": 1, "pageSize": 20, "total": 100, "pages": 5 }
  }
}
```

---

## 三、接口详情

### 3.1 查询逻辑模型列表

**GET** `/api/v1/logical-models`

**请求参数：**

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

**路径参数：** `tableName` - 逻辑模型表名

**响应示例：**
```json
{
  "code": "success",
  "data": {
    "tableName": "dim_user",
    "labelCh": "用户维度表",
    "status": "release",
    "openFlag": 1,
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

**说明：** 查询该模型的版本快照历史，仅记录触发版本变更的事件（发布、列结构变化等），描述性变更不生成版本。

**响应示例：**
```json
{
  "code": "success",
  "data": {
    "tableName": "dim_user",
    "list": [
      { "version": 3, "changeType": "COLUMN_ADDED", "changeSummary": "新增 email_sensitive 列", "releasedAt": "2024-04-20T10:00:00Z", "releasedBy": "user@example.com" },
      { "version": 2, "changeType": "RELEASE", "changeSummary": "模型发布", "releasedAt": "2024-04-10T08:00:00Z" },
      { "version": 1, "changeType": "CREATE", "changeSummary": "初始创建", "releasedAt": "2024-03-01T09:00:00Z" }
    ],
    "pagination": { "pageNum": 1, "pageSize": 20, "total": 3 }
  }
}
```

**changeType 枚举值：**

| 值 | 含义 |
|----|------|
| CREATE | 初始创建 |
| RELEASE | 状态发布（draft→release） |
| COLUMN_ADDED | 新增列 |
| COLUMN_MODIFIED | 列修改（类型/名称/映射表达式变化） |
| COLUMN_DELETED | 删除列 |
| PHYSICAL_MODEL_ADDED | 关联物理模型新增 |
| PHYSICAL_MODEL_DELETED | 关联物理模型删除 |
| PHYSICAL_MODEL_CHANGED | 物理模型映射变化 |
| OPEN_FLAG_CHANGED | 开放标识变化 |

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
      { "path": "logicalColumns[0].columnName", "from": "user_id", "to": "user_key", "changeType": "MODIFIED" },
      { "path": "logicalColumns", "to": { "columnName": "email_sensitive" }, "changeType": "ADDED" }
    ],
    "summary": { "columnsAdded": 1, "columnsModified": 1, "columnsDeleted": 0 }
  }
}
```

---

### 3.7 查询 DQ 核查结果列表

**GET** `/api/v1/data-quality/results`

每次 DQ 任务运行生成唯一的 `jobInsId`，即一个核查结果快照。

**查询参数：**

| 参数 | 必选 | 说明 |
|------|------|------|
| tenantId | 是 | 租户ID |
| catalogId | 否 | 目录ID |
| tableCode | 否 | 模型编码 |
| startDataTime | 否 | 数据开始日期（YYYY-MM-DD） |
| endDataTime | 否 | 数据结束日期（YYYY-MM-DD） |
| pageNum / pageSize | 否 | 分页参数 |

---

### 3.8 查询 DQ 核查结果详情

**GET** `/api/v1/data-quality/results/{jobInsId}`

返回包含四类核查明细的完整结果：

- `jobInstanceDetailTableList` - 表间任务稽核结果
- `jobInstanceDetailSingTableList` - 单表任务稽核结果
- `jobInstanceDetailFieldList` - 字段级稽核结果
- `jobInstanceDetailSqlList` - SQL 稽核结果

---

### 3.9 查询同一模型历史核查结果

**GET** `/api/v1/data-quality/results/{jobInsId}/history?limit=10`

返回同一 `tableCode` 的历史核查结果（不含当前），按时间倒序。

---

### 3.10 比对两次核查结果差异

**GET** `/api/v1/data-quality/results/{jobInsId}/diff?compareWith={previousJobInsId}`

**响应示例：**
```json
{
  "code": "success",
  "data": {
    "fromJobInsId": "task01_old",
    "toJobInsId": "task01_new",
    "fromScore": 95.0,
    "toScore": 98.5,
    "scoreDelta": 3.5,
    "ruleDeltas": [
      { "ruleCode": "RULE_ST_001", "ruleName": "邮箱格式校验", "fromBadRecord": 5000, "toBadRecord": 100, "delta": "IMPROVED" }
    ],
    "newFailedRules": [],
    "recoveredRules": ["RULE_ST_001"]
  }
}
```

**delta 枚举值：**

| 值 | 含义 |
|----|------|
| IMPROVED | 改善（错误记录数减少或从不通过变为通过） |
| DEGRADED | 恶化 |
| UNCHANGED | 无变化 |

---

### 3.11 查询 TMF SID 目录树

**GET** `/api/v1/tmf-sid-catalogs?familyCatalogTypeId=FC001&dataTypeId=1`

**dataTypeId：** 1=Domain, 2=ABE, 3=BE, 4=业务对象, 10=逻辑模型

返回递归嵌套的目录树结构。

---

### 3.12 查询目录下逻辑模型列表

**GET** `/api/v1/tmf-sid-catalogs/{catalogId}/models?familyCatalogId=FC001&themeId=T001&searchWord=user`

支持关键词搜索，返回高亮字段 `highlightFields`。

---

## 四、版本触发规则（逻辑模型）

仅以下变化会生成新版本并通知用户：

| 变化类型 | 是否 +1 版本 |
|---------|------------|
| `status` 从 `draft` → `release` | ✅ |
| 新增列 / 删除列 / 列重命名 | ✅ |
| 列数据类型变化 | ✅ |
| 列映射表达式变化 | ✅ |
| 关联物理模型新增 / 删除 | ✅ |
| `openFlag` 从 1→0（开放→私有） | ✅ |
| 仅修改描述/别名 | ❌（静默保存） |
| `lastModifyTime` 更新 | ❌（系统字段） |

---

## 五、版本触发规则（数据质量）

| 变化类型 | 是否通知用户 |
|---------|------------|
| 新生成核查结果（新的 jobInsId） | ✅ 通知 |
| 分数下降 | ✅ 强通知 |
| 分数不变或提升 | ⚠️ 可选通知 |
| `isCheckPass` 从 1→0（通过→不通过） | ✅ 强通知 |
| 失败规则数增加 | ✅ 通知 |
