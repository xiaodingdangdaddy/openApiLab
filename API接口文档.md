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
| pageSize | query | 否 | integer | 每页最大 500，默认 20 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [
      {
        "tableName": "dim_user",
        "labelCh": "用户维度表",
        "labelEn": "User Dimension",
        "status": "release",
        "owner": "admin@example.com",
        "createTime": "2024-01-15T08:00:00Z"
      }
    ],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 150,
      "pages": 8
    }
  }
}
```

---

### 3.2 查询逻辑模型详情

**GET** `/api/v1/logical-models/{tableName}`

通过 `tableName` 查询包含物理模型映射、列定义等完整详情。

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tableName | path | 是 | string | 逻辑模型名称（即 tableCode） |

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "tableName": "dim_user",
    "labelCh": "用户维度表",
    "labelEn": "User Dimension",
    "status": "release",
    "owner": "admin@example.com",
    "description": "用户主数据维度表",
    "createTime": "2024-01-15T08:00:00Z",
    "lastModifyTime": "2024-04-20T10:00:00Z",
    "relatedPhysicalModels": [
      {
        "physicalModelName": "hive_dim_user",
        "database": "dw_ods",
        "physicalColumns": [
          {
            "columnName": "user_id",
            "dataType": "bigint",
            "dataLength": 20,
            "description": "用户ID"
          },
          {
            "columnName": "email",
            "dataType": "varchar",
            "dataLength": 255,
            "description": "用户邮箱"
          }
        ]
      }
    ],
    "logicalColumns": [
      {
        "columnName": "user_id",
        "labelCh": "用户ID",
        "dataType": "bigint",
        "description": "用户唯一标识",
        "relatedPhyColumnMapping": [
          {
            "physicalModelName": "hive_dim_user",
            "phyColumnName": "user_id"
          }
        ]
      },
      {
        "columnName": "email",
        "labelCh": "用户邮箱",
        "dataType": "varchar",
        "description": "用户邮箱地址",
        "relatedPhyColumnMapping": [
          {
            "physicalModelName": "hive_dim_user",
            "phyColumnName": "email"
          }
        ]
      }
    ],
    "catalogId": "cat_be_001",
    "catalogFullPath": "Domain/ABE/BE/订单域"
  }
}
```

---

### 3.3 查询逻辑模型血缘

**GET** `/api/v1/logical-models/{tableName}/lineage`

返回递归嵌套的前向血缘 / 后向影响链路。

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tableName | path | 是 | string | 逻辑模型名称 |

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| direction | query | 否 | string | 血缘方向：`upstream`（前向）/ `downstream`（后向），默认返回双向 |
| depth | query | 否 | integer | 递归深度，默认 3，最大 10 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "tableName": "dim_user",
    "upstream": [
      {
        "tableName": "ods_user",
        "labelCh": "用户原始表",
        "direction": "upstream",
        "lineageLevel": 1,
        "children": [
          {
            "tableName": "ods_user_backup",
            "labelCh": "用户历史备份表",
            "direction": "upstream",
            "lineageLevel": 2
          }
        ]
      }
    ],
    "downstream": [
      {
        "tableName": "rpt_user_stats",
        "labelCh": "用户统计报表",
        "direction": "downstream",
        "lineageLevel": 1,
        "children": []
      }
    ]
  }
}
```

---

### 3.4 查询逻辑模型版本历史

**GET** `/api/v1/logical-models/{tableName}/versions`

查询版本快照历史，按版本号倒序。

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tableName | path | 是 | string | 逻辑模型名称 |

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页最大 50，默认 20 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "tableName": "dim_user",
    "list": [
      {
        "version": 3,
        "changeType": "COLUMN_ADDED",
        "changeSummary": "新增 email_sensitive 列",
        "releasedAt": "2024-04-20T10:00:00Z",
        "releasedBy": "user@example.com",
        "changeDetail": {
          "columnName": "email_sensitive",
          "dataType": "varchar",
          "dataLength": 255
        }
      },
      {
        "version": 2,
        "changeType": "PHYSICAL_MODEL_ADDED",
        "changeSummary": "新增 Hive 物理表映射 dw_ods.hive_dim_user",
        "releasedAt": "2024-04-10T08:00:00Z",
        "releasedBy": "admin@example.com",
        "changeDetail": {
          "physicalModelName": "hive_dim_user",
          "database": "dw_ods"
        }
      },
      {
        "version": 1,
        "changeType": "CREATE",
        "changeSummary": "初始创建",
        "releasedAt": "2024-03-01T09:00:00Z",
        "releasedBy": "admin@example.com"
      }
    ],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 3,
      "pages": 1
    }
  }
}
```

**changeType 枚举值：**

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

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tableName | path | 是 | string | 逻辑模型名称 |
| version | path | 是 | integer | 版本号 |

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "tableName": "dim_user",
    "version": 2,
    "snapshot": {
      "labelCh": "用户维度表",
      "status": "release",
      "owner": "admin@example.com",
      "relatedPhysicalModels": [
        {
          "physicalModelName": "hive_dim_user",
          "database": "dw_ods",
          "physicalColumns": [...]
        }
      ],
      "logicalColumns": [...]
    },
    "changeType": "PHYSICAL_MODEL_ADDED",
    "changeSummary": "新增 Hive 物理表映射 dw_ods.hive_dim_user",
    "releasedAt": "2024-04-10T08:00:00Z",
    "releasedBy": "admin@example.com"
  }
}
```

---

### 3.6 比对逻辑模型版本差异

**GET** `/api/v1/logical-models/{tableName}/versions/diff`

比对两个版本的模型快照差异。

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tableName | path | 是 | string | 逻辑模型名称 |

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| from | query | 是 | integer | 源版本号（较早版本） |
| to | query | 是 | integer | 目标版本号（较新版本） |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "tableName": "dim_user",
    "fromVersion": 1,
    "toVersion": 3,
    "changedFields": [
      {
        "path": "logicalColumns",
        "changeType": "ADDED",
        "from": null,
        "to": {
          "columnName": "email_sensitive",
          "dataType": "varchar",
          "dataLength": 255
        }
      },
      {
        "path": "relatedPhysicalModels",
        "changeType": "ADDED",
        "from": null,
        "to": {
          "physicalModelName": "hive_dim_user",
          "database": "dw_ods"
        }
      }
    ],
    "summary": {
      "columnsAdded": 1,
      "physicalModelsAdded": 1,
      "physicalModelsDeleted": 0
    }
  }
}
```

---

### 3.7 查询 DQ 核查结果列表

**GET** `/api/v1/data-quality/results`

分页查询数据质量核查结果列表，支持多维度过滤。

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| jobInsId | query | 否 | string | 任务实例ID（精确匹配） |
| catalogIdPath | query | 否 | string | 目录ID（递归，该目录及所有子目录下模型） |
| tableCode | query | 否 | string | 模型编码（精确匹配） |
| startDataTime | query | 否 | string | 数据开始日期（YYYY-MM-DD） |
| endDataTime | query | 否 | string | 数据结束日期（YYYY-MM-DD） |
| status | query | 否 | string | 作业状态：`success` / `failed` / `warning` |
| minScore | query | 否 | number | 最低质量评分（0-100） |
| maxScore | query | 否 | number | 最高质量评分（0-100） |
| ruleType | query | 否 | string | 规则类型（数据质量六性）：`Timeliness` / `accuracy` / `consistency` / `completeness` / `validity` / `uniqueness` |
| owner | query | 否 | string | 模型责任人（精确匹配） |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页最大 500，默认 20 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [
      {
        "jobInsId": "job_ins_001",
        "taskName": "ODS层数据质量核查",
        "tableCode": "ods_user",
        "tableName": "用户原始表",
        "catalogId": "cat_be_001",
        "catalogName": "订单域",
        "catalogPath": "Domain/ABE/BE/订单域",
        "status": "success",
        "score": 95.5,
        "startTime": 1747200000000,
        "endTime": 1747200100000,
        "startDataTime": "2026-05-22",
        "endDataTime": "2026-05-22",
        "sumLineNum": 1523000,
        "createOperator": "system",
        "ruleType": "completeness",
        "owner": "admin@example.com"
      },
      {
        "jobInsId": "job_ins_002",
        "taskName": "DWD层数据质量核查",
        "tableCode": "dwd_user",
        "tableName": "用户维度表",
        "catalogId": "cat_be_002",
        "catalogName": "用户域",
        "catalogPath": "Domain/ABE/BE/用户域",
        "status": "warning",
        "score": 78.2,
        "startTime": 1747200000000,
        "endTime": 1747200050000,
        "startDataTime": "2026-05-22",
        "endDataTime": "2026-05-22",
        "sumLineNum": 1523000,
        "createOperator": "system",
        "ruleType": "accuracy",
        "owner": "admin@example.com"
      }
    ],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 2,
      "pages": 1
    }
  }
}
```

---

### 3.8 查询 DQ 核查结果详情

**GET** `/api/v1/data-quality/results/{jobInsId}`

返回包含四类核查明细的完整结果：表间、单表、字段级、SQL级。

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| jobInsId | path | 是 | string | 任务实例ID |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "jobInsId": "job_ins_001",
    "tableInsId": "table_ins_001",
    "jobInsCode": "DQ_20260522_001",
    "jobInsName": "ODS层数据质量核查",
    "taskId": "task_001",
    "taskCode": "dq_ods_user",
    "catalogId": "cat_be_001",
    "catalogName": "订单域",
    "catalogPath": "Domain/ABE/BE/订单域",
    "dsType": 1,
    "taskName": "ODS层数据质量核查",
    "tableCode": "ods_user",
    "tableName": "用户原始表",
    "status": "success",
    "score": 95.5,
    "startTime": 1747200000000,
    "endTime": 1747200100000,
    "startDataTime": "2026-05-22",
    "endDataTime": "2026-05-22",
    "sumLineNum": 1523000,
    "createOperator": "system",
    "scheduler": "airflow",
    "ruleType": "completeness",
    "owner": "admin@example.com",
    "jobInstanceDetailTableList": [
      {
        "ruleCode": "RULE_TABLE_ROW_COUNT",
        "ruleName": "表行数核查",
        "threshold": ">=1000000",
        "actualValue": 1523000,
        "checkResult": "pass",
        "errorCount": 0
      }
    ],
    "jobInstanceDetailSingTableList": [
      {
        "ruleCode": "RULE_NULL_CHECK",
        "ruleName": "空值核查",
        "columnName": "user_id",
        "threshold": "=0",
        "actualValue": 0,
        "checkResult": "pass",
        "errorCount": 0
      }
    ],
    "jobInstanceDetailFieldList": [
      {
        "ruleCode": "RULE_DATA_TYPE",
        "ruleName": "字段类型核查",
        "columnName": "create_time",
        "threshold": "=datetime",
        "actualValue": "varchar",
        "checkResult": "warning",
        "errorCount": 0
      }
    ],
    "jobInstanceDetailSqlList": [
      {
        "ruleCode": "RULE_SQL_CUSTOM",
        "ruleName": "自定义SQL核查",
        "checkSql": "SELECT COUNT(*) FROM ods_user WHERE status = -1",
        "threshold": "=0",
        "actualValue": 15,
        "checkResult": "failed",
        "errorCount": 15
      }
    ]
  }
}
```

---

### 3.9 查询 TMF SID 目录树

**GET** `/api/v1/tmf-sid-catalogs`

查询 Domain / ABE / BE 等各级目录树结构。

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| familyCatalogTypeId | query | 否 | string | 目录族类型ID，默认 `FC001` |
| dataTypeId | query | 否 | integer | 数据类型ID：`1`=Domain / `2`=ABE / `3`=BE / `4`=业务对象 / `10`=逻辑模型 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": [
    {
      "catalogId": "cat_domain_001",
      "catalogName": "Domain",
      "catalogCode": "DOMAIN",
      "catalogLevel": 1,
      "children": [
        {
          "catalogId": "cat_abe_001",
          "catalogName": "ABE",
          "catalogCode": "ABE",
          "catalogLevel": 2,
          "children": [
            {
              "catalogId": "cat_be_001",
              "catalogName": "订单域",
              "catalogCode": "BE_ORDER",
              "catalogLevel": 3,
              "children": [
                {
                  "catalogId": "cat_be_001_001",
                  "catalogName": "订单明细",
                  "catalogCode": "BE_ORDER_DETAIL",
                  "catalogLevel": 4,
                  "children": []
                }
              ]
            },
            {
              "catalogId": "cat_be_002",
              "catalogName": "用户域",
              "catalogCode": "BE_USER",
              "catalogLevel": 3,
              "children": []
            }
          ]
        }
      ]
    }
  ]
}
```

---

### 3.10 查询目录下逻辑模型列表

**GET** `/api/v1/tmf-sid-catalogs/{catalogId}/models`

在指定目录下搜索逻辑模型，支持关键词搜索。

**路径参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| catalogId | path | 是 | string | 目录ID |

**Query 参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| familyCatalogId | query | 否 | string | 目录族ID，默认 `FC001` |
| themeId | query | 否 | string | 主题ID |
| searchWord | query | 否 | string | 关键词搜索（匹配模型名称/编码） |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页最大 100，默认 20 |

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [
      {
        "tableName": "dim_user",
        "labelCh": "用户维度表",
        "labelEn": "User Dimension",
        "status": "release",
        "owner": "admin@example.com",
        "catalogId": "cat_be_001",
        "catalogPath": "Domain/ABE/BE/用户域",
        "highlightFields": ["user_id", "email", "create_time"]
      },
      {
        "tableName": "dim_order",
        "labelCh": "订单维度表",
        "labelEn": "Order Dimension",
        "status": "release",
        "owner": "admin@example.com",
        "catalogId": "cat_be_001",
        "catalogPath": "Domain/ABE/BE/订单域",
        "highlightFields": ["order_id", "user_id"]
      }
    ],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 2,
      "pages": 1
    }
  }
}
```

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

**数据质量六性（ruleType 枚举值）：**

| 值 | 含义 |
|----|------|
| Timeliness | 及时性 |
| accuracy | 准确性 |
| consistency | 一致性 |
| completeness | 完整性 |
| validity | 有效性 |
| uniqueness | 唯一性 |

---

## 六、版本变更记录

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| v2.0 | 2026-05-21 | 初始版本，10 个查询接口 |
| v2.1 | 2026-05-23 | DQ 列表接口增加 jobInsId/status/minScore/maxScore/ruleType/owner 等过滤维度；修正 ruleType 枚举为六性；补全所有接口的请求/响应详情 |