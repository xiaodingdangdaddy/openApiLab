# 元数据管理查询接口 - API 接口文档

> 本文档基于 OpenAPI 3.1 规范编写，包含所有接口的请求参数、响应格式、字段说明。

---

## 一、接口总览

| 方法 | 路径 | 说明 | 已有接口映射 |
|------|------|------|------------|
| GET | `/logical-models` | 逻辑模型列表（分页） | 内部封装 |
| GET | `/logical-models/{tableName}` | 逻辑模型详情（含物理模型） | `queryLogicalInfoByName` |
| GET | `/logical-models/{tableName}/lineage` | 逻辑模型血缘关系 | `queryLogicalModelsLineage` |
| GET | `/data-quality/reports` | DQ 核查报告列表（分页） | 内部封装 |
| GET | `/data-quality/reports/{jobInsId}` | DQ 核查报告详情 | `queryJobInstanceDetail` |
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
| tenantId | 租户ID | 是 | 某些接口在 query 中传 |

### 2.3 统一响应格式

所有接口均返回统一 JSON 结构：

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

**失败示例：**
```json
{
  "code": "fail",
  "msg": "Logical model not found"
}
```

### 2.4 分页结构

列表查询接口返回分页信息：

```json
{
  "code": "success",
  "msg": "...",
  "data": {
    "list": [ ... ],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 100,
      "pages": 5
    }
  }
}
```

---

## 三、接口详情

### 3.1 查询逻辑模型列表

**GET** `/api/v1/logical-models`

查询逻辑模型列表，支持分页和筛选。

**请求参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| status | query | 否 | string | 发布状态：`draft` / `release` |
| billType | query | 否 | string | 单据类型 |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页条数，默认 20，最大 500 |

**请求示例：**
```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models?tenantId=example_tenant&status=release&pageNum=1&pageSize=20' \
  -H 'Authorization: Bearer <token>'
```

**响应示例（成功）：**
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
        "descCh": "用户主数据维度表",
        "descEn": "User master data dimension table",
        "billType": "buziType_dim",
        "tenantId": "example_tenant",
        "appName": "datacompass",
        "status": "release",
        "openFlag": 1,
        "subscribedFlag": 0
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

通过逻辑模型表名查询完整详情，包含物理模型映射、列定义、关联关系。

**路径参数：**

| 参数 | 必选 | 类型 | 说明 |
|------|------|------|------|
| tableName | 是 | string | 逻辑模型表名 |

**查询参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| isDefault | query | 否 | string | 是否记录访问：1-不记录，0-记录（默认） |

**请求示例：**
```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models/dim_user?tenantId=example_tenant' \
  -H 'Authorization: Bearer <token>'
```

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "tableName": "dim_user",
    "labelCh": "用户维度表",
    "labelEn": "User Dimension",
    "descCh": "用户主数据维度表",
    "descEn": "User master data dimension table",
    "billType": "buziType_dim",
    "tenantId": "example_tenant",
    "appName": "datacompass",
    "status": "release",
    "openFlag": 1,
    "subscribedFlag": 0,
    "relatedCatalogs": [
      {
        "catalogId": "C001",
        "catalogName": "公共维度层",
        "catalogNameEn": "Common Dimension Layer"
      }
    ],
    "relatedSubjectAreas": [
      {
        "subjectAreaId": 1001,
        "subjectAreaName": "客户主题域",
        "subjectAreaLabelCh": "客户主题域",
        "subjectAreaLabelEn": "Customer Subject Area"
      }
    ],
    "relatedPhysicalModels": [
      {
        "physicalModelName": "hive_dim_user",
        "moduleId": 201,
        "labelCh": "Hive用户维度表",
        "labelEn": "Hive User Table",
        "dataDirectory": "/user/hive/warehouse",
        "physicalColumns": [
          {
            "columnName": "user_id",
            "labelCh": "用户ID",
            "labelEn": "User ID",
            "dataType": "varchar",
            "dataLength": "64",
            "columnType": 1,
            "seqNo": 1,
            "openFlag": 1
          }
        ]
      }
    ],
    "logicalColumns": [
      {
        "columnName": "user_id",
        "labelCh": "用户ID",
        "labelEn": "User ID",
        "columnType": 1,
        "seqNo": 1,
        "openFlag": 1,
        "sensitiveType": "ID_CARD",
        "relatedPhyColumnMapping": [
          {
            "physicalModelName": "hive_dim_user",
            "mapPhyColumnExpression": "user_id"
          }
        ]
      }
    ],
    "logicTableRels": [],
    "indicatorsUsed": [],
    "measureUsed": []
  }
}
```

**响应字段说明（核心）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| tableName | string | 逻辑模型表名（主键） |
| labelCh / labelEn | string | 中/英文别名 |
| descCh / descEn | string | 中/英文描述 |
| billType | string | 单据类型，枚举值见上文 |
| status | string | `draft` 草稿 / `release` 已发布 |
| openFlag | integer | 0-私有，1-开放 |
| subscribedFlag | integer | 0-不订阅，1-订阅 |
| relatedPhysicalModels | array | 关联物理模型列表（完整嵌套） |
| logicalColumns | array | 逻辑列定义列表 |
| logicalColumns[].relatedPhyColumnMapping | array | 逻辑列到物理列的映射 |

---

### 3.3 查询逻辑模型血缘关系

**GET** `/api/v1/logical-models/{tableName}/lineage`

查询指定逻辑模型的前向血缘和后向影响链路。

**路径参数：**

| 参数 | 必选 | 类型 | 说明 |
|------|------|------|------|
| tableName | 是 | string | 逻辑模型表名 |

**查询参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |

**请求示例：**
```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models/dim_user/lineage?tenantId=example_tenant' \
  -H 'Authorization: Bearer <token>'
```

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "appName": "datacompass",
    "billType": "buziType_dim",
    "catalogId": ["C001", "C002"],
    "lastModifyTime": "2024-04-20 10:30:00",
    "logicalModelLabelCh": "用户维度表",
    "logicalModelLabelEn": "User Dimension",
    "logicalModelName": "dim_user",
    "forwardLineage": [
      {
        "appName": "datacompass",
        "billType": "buziType_xdr",
        "catalogId": ["C003"],
        "lastModifyTime": "2024-04-18 15:20:00",
        "logicalModelLabelCh": "用户行为明细表",
        "logicalModelLabelEn": "User Behavior Detail",
        "logicalModelName": "xdr_user_behavior",
        "forwardLineage": [],
        "backwardImpact": []
      }
    ],
    "backwardImpact": [
      {
        "appName": "datacompass",
        "billType": "buziType_sdr",
        "catalogId": ["C004"],
        "lastModifyTime": "2024-04-19 09:00:00",
        "logicalModelLabelCh": "用户汇聚表",
        "logicalModelLabelEn": "User Summary Table",
        "logicalModelName": "sdr_user_summary",
        "forwardLineage": [],
        "backwardImpact": []
      }
    ]
  }
}
```

**血缘结构说明：**

```
dim_user（当前模型）
├── forwardLineage[]   → 前向血缘（dim_user 依赖谁）
│   └── xdr_user_behavior
│       └── （递归向下）
└── backwardImpact[]   → 后向影响（谁依赖 dim_user）
    └── sdr_user_summary
        └── （递归向上）
```

---

### 3.4 查询数据质量核查报告列表

**GET** `/api/v1/data-quality/reports`

查询数据质量核查报告列表，支持按目录、模型、时间范围筛选。

**查询参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| tenantId | query | 是 | string | 租户ID |
| catalogId | query | 否 | string | 目录ID |
| tableCode | query | 否 | string | 模型编码 |
| startDataTime | query | 否 | string | 数据开始日期（YYYY-MM-DD） |
| endDataTime | query | 否 | string | 数据结束日期（YYYY-MM-DD） |
| pageNum | query | 否 | integer | 页码，默认1 |
| pageSize | query | 否 | integer | 每页条数，默认20，最大500 |

**请求示例：**
```bash
curl -k -X GET 'https://api.example.com/api/v1/data-quality/reports?tenantId=example_tenant&startDataTime=2024-04-01&endDataTime=2024-04-30&pageNum=1&pageSize=20' \
  -H 'Authorization: Bearer <token>'
```

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [
      {
        "jobInsId": "task01_9d43274da8ac444a",
        "jobInsName": "用户表数据质量核查",
        "catalogId": "C001",
        "catalogName": "公共维度层",
        "catalogPath": "/数据域/公共维度层",
        "tableCode": "dim_user",
        "tableName": "用户维度表",
        "dsType": 1,
        "taskName": "月度数据质量检查",
        "sumLineNum": 1000000,
        "startTime": 1713446400000,
        "endTime": 1713446460000,
        "status": 1,
        "score": 98.5,
        "createOperator": "admin"
      }
    ],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 50,
      "pages": 3
    }
  }
}
```

---

### 3.5 查询数据质量核查报告详情

**GET** `/api/v1/data-quality/reports/{jobInsId}`

根据任务实例ID查询完整核查结果详情，包含多维度（表间/单表/字段级/SQL级）结果。

**路径参数：**

| 参数 | 必选 | 类型 | 说明 |
|------|------|------|------|
| jobInsId | 是 | string | 稽核任务实例ID |

**请求示例：**
```bash
curl -k -X GET 'https://api.example.com/api/v1/data-quality/reports/task01_9d43274da8ac444a' \
  -H 'Authorization: Bearer <token>'
```

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "jobInsId": "task01_9d43274da8ac444a",
    "tableInsId": "TI001",
    "jobInsCode": "DQ_20240420_001",
    "jobInsName": "用户表数据质量核查",
    "taskId": "TASK001",
    "taskCode": "DQ_TASK_001",
    "catalogId": "C001",
    "catalogName": "公共维度层",
    "catalogCode": "COMMON_DIM",
    "catalogPath": "/数据域/公共维度层",
    "dsType": 1,
    "taskName": "月度数据质量检查",
    "tableCode": "dim_user",
    "tableName": "用户维度表",
    "createOperator": "admin",
    "description": "针对用户维度表的全面数据质量检查",
    "startDataTime": "2024-04-01",
    "endDataTime": "2024-04-30",
    "sumLineNum": 1000000,
    "startTime": 1713446400000,
    "endTime": 1713446460000,
    "status": 1,
    "score": 98.5,
    "scheduler": "调度系统A",
    "modelId": "MODEL001",
    "jsonData": {
      "dimensions": ["completeness", "accuracy"]
    },
    "jobInstanceDetailTableList": [
      {
        "ruleCode": "RULE_TABLE_001",
        "threshold": 100,
        "totalRecord": 100,
        "badRecord": 0,
        "isCheckPass": 1,
        "flagSite": "all_rows"
      }
    ],
    "jobInstanceDetailSingTableList": [
      {
        "ruleCode": "RULE_ST_001",
        "threshold": 0,
        "totalRecord": 1000000,
        "badRecord": 15000,
        "isCheckPass": 0,
        "fields": "user_id"
      }
    ],
    "jobInstanceDetailFieldList": [],
    "jobInstanceDetailSqlList": [],
    "jobInsResultDetailFieldBo": {
      "ruleCode": "RULE_FIELD_001",
      "threshold": 95,
      "totalRecord": 1000000,
      "badRecord": 15000,
      "isCheckPass": 0,
      "fields": "email"
    }
  }
}
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| score | double | 质量评分（0-100） |
| status | int | 任务状态 |
| sumLineNum | int | 核查总行数 |
| jobInstanceDetailTableList | array | 表间任务稽核结果 |
| jobInstanceDetailSingTableList | array | 单表任务稽核结果 |
| jobInstanceDetailFieldList | array | 字段级稽核结果 |
| jobInstanceDetailSqlList | array | SQL稽核结果 |
| isCheckPass | int | 0-未通过，1-通过 |

---

### 3.6 查询 TMF SID 目录树

**GET** `/api/v1/tmf-sid-catalogs`

查询 TMF SID 目录树，包含 Domain / ABE / BE / 业务对象 等各级目录。

**查询参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| familyCatalogTypeId | query | 是 | string | 目录族类型ID |
| dataTypeId | query | 是 | string | 资产类型ID（见下表） |
| themeId | query | 否 | string | 场景ID |

**dataTypeId 枚举值：**

| 值 | 含义 |
|----|------|
| 1 | Domain（领域） |
| 2 | ABE（Account Bilateral Entity） |
| 3 | BE（Business Entity） |
| 4 | 业务对象 |
| 10 | 逻辑模型 |

**请求示例：**
```bash
# 查询 Domain 层
curl -k -X GET 'https://api.example.com/api/v1/tmf-sid-catalogs?familyCatalogTypeId=FC001&dataTypeId=1' \
  -H 'Authorization: Bearer <token>'

# 查询 BE 层
curl -k -X GET 'https://api.example.com/api/v1/tmf-sid-catalogs?familyCatalogTypeId=FC001&dataTypeId=3' \
  -H 'Authorization: Bearer <token>'
```

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": [
    {
      "catalogId": "D001",
      "catalogNameCn": "客户域",
      "catalogNameEn": "Customer Domain",
      "catalogDesc": "客户相关的业务领域",
      "catalogDescEn": "Customer Business Domain",
      "createBy": "system",
      "parentCatalogId": null,
      "catalogLevel": "1",
      "createDate": "2024-01-01 00:00:00",
      "catalogSort": "1",
      "operID": "OP001",
      "themes": ["T001", "T002"],
      "lastUpdateDate": "2024-04-20 10:00:00",
      "modified": false,
      "children": [
        {
          "catalogId": "ABE001",
          "catalogNameCn": "客户账户ABE",
          "catalogNameEn": "Customer Account ABE",
          "catalogDesc": "客户账户相关ABE",
          "parentCatalogId": "D001",
          "catalogLevel": "2",
          "children": [
            {
              "catalogId": "BE001",
              "catalogNameCn": "客户账户BE",
              "catalogNameEn": "Customer Account BE",
              "catalogLevel": "3",
              "children": []
            }
          ]
        }
      ]
    }
  ]
}
```

**目录层级说明：**

```
Domain（领域）
└── ABE（Account Bilateral Entity）
    └── BE（Business Entity）
        └── 业务对象
            └── 逻辑模型
```

---

### 3.7 查询 TMF SID 目录下逻辑模型列表

**GET** `/api/v1/tmf-sid-catalogs/{catalogId}/models`

在指定目录下查询关联的逻辑模型列表，支持分页和关键词搜索。

**路径参数：**

| 参数 | 必选 | 类型 | 说明 |
|------|------|------|------|
| catalogId | 是 | string | 目录ID |

**查询参数：**

| 参数 | 位置 | 必选 | 类型 | 说明 |
|------|------|------|------|------|
| familyCatalogId | query | 是 | string | 目录族ID |
| themeId | query | 是 | string | 场景ID |
| searchWord | query | 否 | string | 搜索词（支持高亮） |
| onlyShowCurrent | query | 否 | boolean | 仅显示当前目录下模型，默认 false |
| pageNum | query | 否 | integer | 页码，默认 1 |
| pageSize | query | 否 | integer | 每页条数，默认 15，最大 100 |

**请求示例：**
```bash
curl -k -X GET 'https://api.example.com/api/v1/tmf-sid-catalogs/D001/models?familyCatalogId=FC001&themeId=T001&searchWord=user&pageNum=1&pageSize=15' \
  -H 'Authorization: Bearer <token>'
```

**响应示例：**
```json
{
  "code": "success",
  "msg": "Query success",
  "data": {
    "list": [
      {
        "dataAssetId": "AUTO_1",
        "dataDescCn": "用户主数据维度表",
        "dataDescEn": "User Master Data Dimension Table",
        "dataNameCn": "dim_user",
        "dataNameEn": "dim_user",
        "highlightFields": {
          "dataNameCn": ["<em>dim_user</em>"]
        },
        "logicalAttriList": [
          {
            "attributesId": "ATTR001",
            "attributesName": "用户ID",
            "attributesNameEn": "User ID",
            "attributesDescCn": "用户的唯一标识",
            "attributesDescEn": "Unique identifier for user"
          }
        ],
        "physicalAttriList": [],
        "physicalNameInfoList": [
          {
            "physicalEntityName": "hive_dim_user",
            "physicalEntityNameEn": "hive_dim_user"
          }
        ],
        "themeInfoList": [
          {
            "themeId": "T001"
          }
        ],
        "tenantId": "example_tenant",
        "createDateStr": "2024-04-25 00:30:34"
      }
    ],
    "pageNum": 1,
    "pageSize": 15,
    "total": 1,
    "pages": 1,
    "size": 1
  }
}
```

---

## 四、错误码说明

| code | 说明 | 可能原因 |
|------|------|---------|
| success | 查询成功 | - |
| fail | 查询失败 | 参数错误、系统异常 |

---

## 五、数据类型 billType 枚举说明

| 值 | 含义 |
|----|------|
| buziType_global | 全局 |
| buziType_xdr | XDR（详单数据） |
| buziType_sdr | SDR（统计/汇聚数据） |
| buziType_pre | PRE（预计算） |
| buziType_bkpi | BKPI（关键绩效指标） |
| buziType_dim | DIM（维度） |
| buziType_cfg | CFG（配置） |
| buziType_json | JSON（JSON数据） |
| buziType_event | EVENT（事件） |
| buziType_dcae_event | DCAE_EVENT（DCAE事件） |

---

## 六、数据源类型 dsType 说明

| 值 | 含义 |
|----|------|
| 1 | HIVE |
| 2 | Oracle |
| 3 | MySQL |
| ... | 其他数据源类型 |

---

## 七、敏感类型 sensitiveType 参考

| 值 | 含义 |
|----|------|
| ID_CARD | 身份证号 |
| PHONE | 手机号 |
| EMAIL | 邮箱 |
| BANK_CARD | 银行卡号 |
| PASSWORD | 密码 |
