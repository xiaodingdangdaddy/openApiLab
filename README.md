# 元数据管理查询接口（Metadata Query API）

基于 OpenAPI 3.1 规范封装的元数据管理统一查询接口，对外提供逻辑模型、血缘关系、数据质量、TMF SID 数据目录的查询能力。

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
| GET | `/logical-models/{tableName}` | 逻辑模型详情（含物理模型映射、列定义） |
| GET | `/logical-models/{tableName}/lineage` | 逻辑模型血缘关系（双向链路） |
| GET | `/data-quality/reports` | 数据质量核查报告列表（分页） |
| GET | `/data-quality/reports/{jobInsId}` | 数据质量核查报告详情（多维度） |
| GET | `/tmf-sid-catalogs` | TMF SID 目录树（Domain/ABE/BE/业务对象） |
| GET | `/tmf-sid-catalogs/{catalogId}/models` | 目录下逻辑模型列表（支持搜索） |

---

## 三、认证方式

所有接口通过 `Authorization: Bearer <token>` Header 认证。

```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_access_token'
```

---

## 四、文件说明

| 文件 | 说明 |
|------|------|
| `openapi.yaml` | OpenAPI 3.1 规范文件，可导入 Apifox / Postman / Swagger UI |
| `API接口文档.md` | 接口详细文档，含请求/响应示例、字段说明、枚举值 |
| `使用说明书.md` | 面向调用方的使用指南，含 7 个业务场景的 Python 示例代码 |

---

## 五、快速开始

### 5.1 导入接口文档

1. **Apifox / Postman**：导入 `openapi.yaml`，自动生成接口集合
2. **Swagger UI**：访问 https://editor.swagger.io/ ，粘贴 `openapi.yaml` 内容

### 5.2 查询示例

**查询逻辑模型详情：**

```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models/dim_user?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_token'
```

**查询数据质量报告：**

```bash
curl -k -X GET 'https://api.example.com/api/v1/data-quality/reports?tenantId=your_tenant&startDataTime=2024-04-01&endDataTime=2024-04-30' \
  -H 'Authorization: Bearer your_token'
```

### 5.3 Python 调用示例

```python
import requests

def get_logical_model_detail(table_name, tenant_id, token):
    headers = {"Authorization": f"Bearer {token}"}
    url = f"https://api.example.com/api/v1/logical-models/{table_name}"
    resp = requests.get(url, headers=headers, params={"tenantId": tenant_id})
    data = resp.json()
    if data["code"] != "success":
        raise ValueError(f"Failed: {data['msg']}")
    return data["data"]

detail = get_logical_model_detail("dim_user", "your_tenant", "your_token")
print(f"Model: {detail['tableName']}, Columns: {len(detail['logicalColumns'])}")
```

更多示例请参考 `使用说明书.md`。

---

## 六、数据模型说明

### 6.1 逻辑模型（LogicalModel）

逻辑模型是核心业务抽象，关联物理模型、列定义、主题域、数据标准等元数据。一个逻辑模型可对应多个物理模型。

### 6.2 血缘关系（Lineage）

血缘关系分前向（forwardLineage，我依赖谁）和后向（backwardImpact，谁依赖我），递归嵌套结构。

### 6.3 数据质量（DataQuality）

DQ 核查报告包含表间、单表、字段级、SQL级四类核查结果，以及整体质量评分（score）。

### 6.4 TMF SID 目录

TMF SID 目录按层级组织：Domain → ABE → BE → 业务对象 → 逻辑模型。目录族类型（familyCatalogTypeId）确定目录归属。

---

## 七、错误码

| code | 说明 |
|------|------|
| `success` | 查询成功 |
| `fail` | 查询失败（见 msg 获取具体原因） |

---

## 八、注意事项

1. **分页限制**：列表接口每次最多返回 500 条（logical-models）和 500 条（data-quality/reports）
2. **字符编码**：接口路径和参数值需做 URL 编码
3. **Token 管理**：建议将 Access Token 存储在配置中心，避免硬编码
4. **错误重试**：接口调用失败时建议做 3 次重试，间隔递增（1s, 5s, 30s）

---

## 九、接口响应时间参考

| 接口 | 预计响应时间 |
|------|------------|
| `/logical-models` | 500ms ~ 2s |
| `/logical-models/{tableName}` | 200ms ~ 1s |
| `/logical-models/{tableName}/lineage` | 300ms ~ 1.5s |
| `/data-quality/reports` | 300ms ~ 1s |
| `/data-quality/reports/{jobInsId}` | 200ms ~ 1s |
| `/tmf-sid-catalogs` | 200ms ~ 2s |
| `/tmf-sid-catalogs/{catalogId}/models` | 500ms ~ 3s |
