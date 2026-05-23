# 元数据管理查询接口（Metadata Query API）

基于 OpenAPI 3.1 规范（v2.2）封装的元数据管理统一查询接口，对外提供逻辑模型、血缘关系、版本追踪、数据质量、TMF SID 数据目录的查询能力。

---

## 一、项目说明

本项目将企业内部零散的元数据查询接口统一封装为符合 OpenAPI 3.1 规范的 RESTful API，供 Datahub 等元数据管理平台调用，实现元数据的统一纳管。

**对接架构：**

```
已有系统（零散接口）
        ↓ 调用
   BFF / API Gateway 封装层
        ↓
   本 OpenAPI（统一出口）
        ↓
   Datahub 适配层（自行开发）
        ↓
   DataHub 元数据中心
```

---

## 二、接口清单

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/logical-models` | 逻辑模型列表（分页） |
| GET | `/logical-models/{tableName}` | 逻辑模型详情（含物理模型映射） |
| GET | `/logical-models/{tableName}/lineage` | 逻辑模型血缘关系（双向链路） |
| GET | `/logical-models/{tableName}/versions` | 逻辑模型版本历史 |
| GET | `/logical-models/{tableName}/versions/{version}` | 指定版本详情快照 |
| GET | `/logical-models/{tableName}/versions/diff` | 版本差异比对 |
| GET | `/data-quality/results` | DQ 核查结果列表（分页） |
| GET | `/data-quality/results/{jobInsId}` | DQ 核查结果详情 |
| GET | `/tmf-sid-catalogs` | TMF SID 目录树 |
| GET | `/tmf-sid-catalogs/{catalogId}/models` | 目录下逻辑模型列表（支持搜索） |

---

## 三、版本变更追踪规则

### 触发版本 +1 的变化

| 变化类型 | 说明 |
|---------|------|
| 逻辑列新增 | `logicalColumns` 中出现新的 columnName |
| 关联物理模型新增 | `relatedPhysicalModels` 中出现新的物理模型 |
| 关联物理模型删除 | `relatedPhysicalModels` 中某个物理模型消失 |
| 物理表列新增 | `physicalColumns` 中出现新的列 |

### 不触发版本的变化

- 逻辑列删除（系统不允许）
- 物理列删除（系统不允许）
- 列重命名（系统不允许）
- 列数据类型变化（系统不允许）
- 列映射表达式变化（系统不允许）
- 别名/描述修改、`lastModifyTime` 更新等描述性变化

以上变化系统本身不允许操作，因此不作为版本变更条件。

### 数据质量

不做版本变化追踪。每次 DQ 核查生成唯一 `jobInsId`，直接使用 `score` 字段作为评分指标。

---

## 四、认证方式

```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_access_token'
```

---

## 五、文件说明

| 文件 | 说明 |
|------|------|
| `openapi.yaml` | OpenAPI 3.1 规范文件（v2.2），可导入 Apifox / Postman / Swagger UI |
| `API接口文档.md` | 接口详细文档，含请求/响应示例、字段说明、版本变更规则 |
| `使用说明书.md` | 面向调用方的使用指南，含 Python 示例代码 |

---

## 六、快速开始

```bash
# 查询逻辑模型详情
curl -k -X GET 'https://api.example.com/api/v1/logical-models/dim_user?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_token'

# 查询 DQ 核查结果
curl -k -X GET 'https://api.example.com/api/v1/data-quality/results?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_token'

# 比对版本差异
curl -k -X GET 'https://api.example.com/api/v1/logical-models/dim_user/versions/diff?from=1&to=3' \
  -H 'Authorization: Bearer your_token'
```

---

## 七、错误码

| code | 说明 |
|------|------|
| `success` | 查询成功 |
| `fail` | 查询失败（见 msg 获取具体原因） |

---

## 九、BFF 封装设计说明

本节说明如何利用企业已有的老接口，封装为满足 Dashboard 多维度过滤场景的新 OpenAPI。核心思路是**在老接口前面加一层 BFF 封装层**，统一做参数转换、响应裁剪、接口串联，对外暴露标准化的 OpenAPI 接口。

### 9.1 整体架构

```
外部调用方（DataHub / Dashboard）
        ↓ 调用新 OpenAPI（统一出口）
┌──────────────────────────────────────────┐
│           新 OpenAPI BFF 封装层            │
│                                          │
│   参数标准化 ────►  老接口参数转换          │
│   多接口串联 ────►  按需组合多个老接口      │
│   响应裁剪   ────►  只返回约定的字段        │
└──────────────────────┬───────────────────┘
                       │
         ┌─────────────┼──────────────┐
         ▼             ▼              ▼
   ┌──────────┐ ┌──────────┐  ┌──────────┐
   │  老API 1 │ │ 老API 2  │  │ 老API 3  │
   │ DQ接口   │ │ 模型接口 │  │ 目录接口 │
   │（不改动）│ │（不改动）│  │（不改动）│
   └──────────┘ └──────────┘  └──────────┘
```

**核心原则：老接口不动、不改动；新 OpenAPI 站在老接口肩膀上对外服务。**

### 9.2 三种封装模式

| 模式 | 何时用 | 示例 |
|------|--------|------|
| **一一映射** | 新旧接口能力一致，只需转参数和响应结构 | `GET /logical-models/{tableName}` → `queryLogicalInfoByName` |
| **分页增强** | 老接口无分页或分页格式不标准，BFF 补全处理 | `GET /logical-models` 加 pageNum/pageSize 处理 |
| **串联聚合** | 单个老接口不够，需按业务逻辑串联多个老接口 | `catalogIdPath` 串联 CAT-1 + CAT-2 + DQ-2 |

### 9.3 场景一：DQ 核查结果按目录递归过滤

**需求：** Dashboard 用户按"订单域目录"过滤 DQ 结果

老 API `queryJobInstanceList` 只支持时间 + 状态过滤，不支持目录过滤。新 OpenAPI 通过 BFF 串联实现：

```
请求：
GET /api/v1/data-quality/results?catalogIdPath=cat_be_001&minScore=80

BFF 内部执行流程：

Step 1 — 查目录树（CAT-1）
  GET /dams/sys-mgmt/v1/dataCatalogTree?familyCatalogTypeId=FC001
  → 找到 cat_be_001 及其所有子节点 catalogId
    [cat_be_001, cat_be_001_001, cat_be_001_002]

Step 2 — 查这些目录下有哪些模型（CAT-2）
  POST /apiaccess/dams/search/v1/model/list
  body: { "catalogIds": [...], "familyCatalogId": "FC001", "pageSize": 1000 }
  → 得到模型 tableName 列表
    ["dim_order", "dim_order_detail", "fact_sale"]

Step 3 — 用模型列表过滤 DQ 结果（DQ-2）
  POST /taskJobMgmtRest/queryJobInstanceList
  body: { "tableCodeList": [...], "startDataTime": "...", "endDataTime": "..." }

Step 4 — BFF 补齐 catalogFullPath、owner 等字段，返回标准化结果
```

封装效果：用户只需传一个 `catalogIdPath=cat_be_001` 参数，BFF 自动串联 3 个老接口完成过滤。

### 9.4 场景二：逻辑模型列表按场景（themeId）过滤

**需求：** Dashboard 用户按"用户域"标签查看模型列表

```
请求：
GET /api/v1/logical-models?themeId=T001&status=release

BFF 内部执行流程：

Step 1 — 用老 model/list 接口查该场景下所有模型（CAT-2）
  POST /apiaccess/dams/search/v1/model/list
  body: { "themeId": "T001", "familyCatalogId": "FC001", "pageSize": 1000 }
  → 拿到 tableName 列表
    ["dim_user", "dim_user_ext", "rpt_user_active"]

Step 2 — 用 tableName 列表过滤 logical-models 结果
  对每个 tableName 调 GET /dams/asset-mgmt/v1/logicalDataModel/datacompass/queryLogicalInfoByName

Step 3 — BFF 聚合成列表返回
```

封装效果：用户只需传 `themeId=T001`，BFF 自动调老接口拿场景下所有模型并聚合返回。

### 9.5 场景三：逻辑模型血缘批量模式

**需求：** Dashboard 用户查看"订单域"下所有模型的上下游依赖关系

```
请求：
GET /api/v1/logical-models/_batch/lineage?catalogIdPath=cat_be_001&direction=both&depth=3

BFF 内部执行流程：

Step 1 — 同 catalogIdPath 逻辑，先拿目录下所有模型（CAT-1 + CAT-2）
  ["dim_order", "fact_sale", "dim_product"]

Step 2 — 批量查每个模型的 lineage（LM-2）
  对列表中每个模型各自调 /queryLogicalModelsLineage

Step 3 — BFF 聚合成数组返回
  {
    "code": "success",
    "data": [
      { "tableName": "dim_order",   "upstream": [...], "downstream": [...] },
      { "tableName": "fact_sale",   "upstream": [...], "downstream": [...] },
      { "tableName": "dim_product", "upstream": [...], "downstream": [...] }
    ]
  }
```

### 9.6 老 API 接口清单

封装前已有 6 个老接口，新 OpenAPI 的能力来自这些老接口：

| 序号 | 接口名称 | 老路径 | 用途 |
|------|---------|--------|------|
| DQ-1 | DQ核查结果详情 | `POST /taskJobMgmtRest/queryJobInstanceDetail` | 按 jobInsId 查单条 |
| DQ-2 | DQ核查结果列表 | `POST /taskJobMgmtRest/queryJobInstanceList` | 分页列表+时间过滤 |
| LM-1 | 逻辑模型详情 | `GET /dams/asset-mgmt/v1/logicalDataModel/datacompass/queryLogicalInfoByName` | 模型详情（含物理模型映射、owner、catalogId） |
| LM-2 | 逻辑模型血缘 | `GET /dams/asset-mgmt/v1/logicalDataModel/datacompass/queryLogicalModelsLineage` | 上下游血缘链路 |
| CAT-1 | TMF SID 目录树 | `GET /dams/sys-mgmt/v1/dataCatalogTree` | 查 Domain/ABE/BE 各级目录树 |
| CAT-2 | 目录下模型搜索 | `POST /apiaccess/dams/search/v1/model/list` | 在指定目录下搜索模型（含 themeId） |

### 9.7 工程化建议

| 建议 | 说明 |
|------|------|
| catalogIdPath 递归加缓存 | 目录层级深、模型多时，建议目录树缓存 5 分钟、模型列表缓存 1 分钟 |
| 批量接口控制上限 | tableNames 列表建议最多 50 个，lineage 批量建议最多 20 个 |
| 错误处理统一封装 | 所有老接口错误统一转换为 `{ "code": "fail", "msg": "具体原因" }` 格式 |
| BFF 只做转换，不改老接口逻辑 | 老接口的认证、限流、错误处理逻辑原样复用，BFF 只做参数/响应的标准化 |

---

## 八、接口响应时间参考

| 接口 | 预计响应时间 |
|------|------------|
| `/logical-models` | 500ms ~ 2s |
| `/logical-models/{tableName}` | 200ms ~ 1s |
| `/logical-models/{tableName}/lineage` | 300ms ~ 1.5s |
| `/logical-models/{tableName}/versions` | 200ms ~ 1s |
| `/logical-models/{tableName}/versions/diff` | 300ms ~ 1s |
| `/data-quality/results` | 300ms ~ 1s |
| `/data-quality/results/{jobInsId}` | 200ms ~ 1s |
| `/tmf-sid-catalogs` | 200ms ~ 2s |
| `/tmf-sid-catalogs/{catalogId}/models` | 500ms ~ 3s |
