# 元数据管理查询接口（Metadata Query API）

基于 OpenAPI 3.1 规范（v2.0）封装的元数据管理统一查询接口，对外提供逻辑模型、血缘关系、数据质量、TMF SID 数据目录的查询能力，以及版本变更追踪与比对能力。

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
| GET | `/data-quality/results/{jobInsId}` | DQ 核查结果详情（多维度） |
| GET | `/data-quality/results/{jobInsId}/history` | 同一模型历史核查结果 |
| GET | `/data-quality/results/{jobInsId}/diff` | 核查结果差异比对 |
| GET | `/tmf-sid-catalogs` | TMF SID 目录树 |
| GET | `/tmf-sid-catalogs/{catalogId}/models` | 目录下逻辑模型列表（支持搜索） |

---

## 三、核心能力：版本变更追踪

### 3.1 逻辑模型版本变更

**触发版本 +1 的变化：**
- `draft` → `release` 发布事件
- 新增列 / 删除列 / 列重命名 / 类型变化
- 列映射表达式变化
- 关联物理模型新增 / 删除
- 访问权限从开放变为私有

**不触发版本的变化（静默保存）：**
- 描述/别名修改、`lastModifyTime` 更新等

### 3.2 数据质量变更通知

每次 DQ 任务运行生成新的 `jobInsId`，即一个核查结果快照。
- 分数下降 → 强通知
- `isCheckPass` 从通过→不通过 → 强通知
- 新失败规则出现 → 通知
- 分数不变/提升 → 可选通知

---

## 四、认证方式

所有接口通过 `Authorization: Bearer <token>` Header 认证。

```bash
curl -k -X GET 'https://api.example.com/api/v1/logical-models?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_access_token'
```

---

## 五、文件说明

| 文件 | 说明 |
|------|------|
| `openapi.yaml` | OpenAPI 3.1 规范文件（v2.0），可导入 Apifox / Postman / Swagger UI |
| `API接口文档.md` | 接口详细文档，含所有接口请求/响应示例、字段说明、变更规则 |
| `使用说明书.md` | 面向调用方的使用指南，含 Python 示例代码 |

---

## 六、快速开始

```bash
# 查询逻辑模型详情
curl -k -X GET 'https://api.example.com/api/v1/logical-models/dim_user?tenantId=your_tenant' \
  -H 'Authorization: Bearer your_token'

# 查询 DQ 核查结果
curl -k -X GET 'https://api.example.com/api/v1/data-quality/results?tenantId=your_tenant&startDataTime=2024-04-01' \
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
| `/data-quality/results/{jobInsId}/diff` | 300ms ~ 1s |
| `/tmf-sid-catalogs` | 200ms ~ 2s |
| `/tmf-sid-catalogs/{catalogId}/models` | 500ms ~ 3s |
