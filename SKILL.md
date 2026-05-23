---
name: openapi-generator
description: |
  Design and generate OpenAPI 3.1 specifications from existing legacy APIs.
  Use when: (1) user provides scattered existing APIs and wants a unified OpenAPI standard,
  (2) user asks to design OpenAPI from scratch, (3) user mentions BFF encapsulation or
  packaging old APIs into OpenAPI, (4) integrating with Datahub or similar metadata platforms.
  Triggers (match any): "帮我设计OpenAPI", "生成OpenAPI接口", "根据已有接口生成openapi",
  "封装老接口为OpenAPI", "generate OpenAPI", "design OpenAPI from legacy APIs",
  "openapi yaml", "帮我生成openapi".
---

# openapi-generator

Design and generate OpenAPI 3.1 specifications from existing legacy APIs, with optional GitHub push.

## 适用场景

- 企业内部零散老接口封装为统一 OpenAPI 标准
- BFF 聚合层接口设计
- DataHub / 元数据管理平台对接接口设计
- 新项目 REST API 规范设计

---

## 工作流程

### Step 1：收集老接口信息

对每个老接口，收集以下信息：

| 信息项 | 说明 |
|--------|------|
| 接口名称 | 业务名称，如"DQ核查结果查询" |
| 请求类型 | GET / POST / PUT / DELETE |
| 老路径 | 原始 URL 路径 |
| 请求参数 | 名称、必选/可选、类型、说明 |
| 响应结构 | 字段名、类型、说明、示例值 |
| 认证方式 | Bearer Token / 自定义 Header 等 |

**引导式提问：**
```
请提供每个接口的：
1. 接口名称和用途
2. 请求类型和完整路径
3. 请求参数清单（含必选/可选）
4. 响应数据示例（完整 JSON）
5. 认证方式
```

### Step 2：确定封装模式

对每个新接口，判断属于哪种封装模式：

| 模式 | 适用场景 | 示例 |
|------|---------|------|
| **一一映射** | 新旧接口能力一致，只需参数/响应标准化 | `GET /logical-models/{tableName}` → 老 LM-1 接口 |
| **分页增强** | 老接口无分页，BFF 补充分页处理 | `GET /logical-models` 加 pageNum/pageSize |
| **串联聚合** | 单个老接口不够，需串联多个老接口 | `catalogIdPath` 过滤 DQ 结果 |

### Step 3：设计路径结构

**路径命名原则：**

| 场景 | 路径 | 说明 |
|------|------|------|
| 列表查询 | `GET /resources` | 复数名词 |
| 详情查询 | `GET /resources/{id}` | 路径参数 |
| 子资源 | `GET /resources/{id}/sub` | 嵌套表达从属 |
| 版本历史 | `GET /resources/{id}/versions` | 子资源 |
| 版本差异 | `GET /resources/{id}/versions/diff?from=&to=` | diff 查询 |
| 批量模式 | `GET /resources/_batch/sub?filter=xxx` | underscore 前缀表示特殊模式 |

**禁止：**
- 动词路径（`/getResource` ❌ → `GET /resources` ✅）
- 路径参数用 query string 代替（`?id=xxx` ❌ → `/{id}` ✅）

### Step 4：设计 Schema

**响应包装结构（必须）：**

```json
{
  "code": "success",
  "msg": "Query success",
  "data": { ... }
}
```

**分页结构（必须）：**

```json
{
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

**Schema 设计原则：**
- 直接复用老接口响应字段结构，不做过度抽象
- 嵌套层级与老接口保持一致
- 枚举值直接映射原接口
- 不确定字段类型时，使用 `type: string` + `description` 说明

### Step 5：生成文件

生成以下文件到用户工作目录（如 `datacubeOpenApi/`）：

| 文件 | 说明 |
|------|------|
| `openapi.yaml` | 完整 OpenAPI 3.1 规范（主文件） |
| `API接口文档.md` | 接口详细文档，含参数表 + 响应示例 |
| `使用说明书.md` | 面向调用方，含 Python 示例代码 |
| `README.md` | 项目总览，快速开始 |

**版本号规范：** 首次生成 `v2.0`，后续变更 `v2.1` / `v2.2` ...

### Step 6：推送到 GitHub（可选）

**触发条件：** 用户明确说"推送到 GitHub" / "上传到 GitHub" / "sync to repo"

**操作方式：**

使用 `openapi-github-sync` 技能：

```bash
python3 ~/.agents/skills/openapi-github-sync/scripts/push_to_github.py \
  <openapi.yaml> <API接口文档.md> <使用说明书.md> <README.md> \
  -m "<commit message>"
```

**推送前确认事项：**
- 文件已保存到本地工作目录
- GitHub token 在 `openapi-github-sync/scripts/config.json` 中已配置
- 仓库名为 `xiaodingdangdaddy/openApiLab`（默认）

---

## OpenAPI 核心规范速查

### 路径结构模板

```yaml
openapi: 3.1.0
info:
  title: <项目名称>
  version: <版本>
servers:
  - url: https://api.example.com/api/v1
    description: 正式环境
  - url: https://api-test.example.com/api/v1
    description: 测试环境
paths:
  /<resource>:
    get:
      operationId: <operationId>
      summary: <简要说明>
      description: <详细说明>
      parameters:
        - name: <paramName>
          in: query
          required: true/false
          schema:
            type: string
          description: <说明>
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/<SchemaName>'
components:
  schemas:
    <SchemaName>:
      type: object
      properties:
        field:
          type: string
          description: <说明>
```

### 响应 Schema 模板

```yaml
# 分页列表响应
<Resource>ListResponse:
  type: object
  properties:
    code: { type: string, example: success }
    msg: { type: string, example: Query success }
    data:
      type: object
      properties:
        list:
          type: array
          items: { $ref: '#/components/schemas/<Resource>' }
        pagination: { $ref: '#/components/schemas/PaginationInfo' }

# 详情响应
<Resource>DetailResponse:
  type: object
  properties:
    code: { type: string, example: success }
    msg: { type: string, example: Query success }
    data: { $ref: '#/components/schemas/<Resource>' }

# 分页结构
PaginationInfo:
  type: object
  properties:
    pageNum: { type: integer, example: 1 }
    pageSize: { type: integer, example: 20 }
    total: { type: integer, example: 156 }
    pages: { type: integer, example: 8 }
```

### 枚举值处理

直接映射老接口枚举值，不添加未出现在老接口中的枚举：

```yaml
status:
  type: string
  enum: [success, failed, warning]
  description: 作业状态
```

---

## 错误处理规范

| 场景 | code | msg |
|------|------|-----|
| 成功 | `success` | `Query success` |
| 参数缺失 | `bad_request` | 具体缺失参数名 |
| 认证失败 | `unauthorized` | `Invalid token` |
| 资源不存在 | `not_found` | 具体资源标识 |
| 服务端错误 | `internal_error` | `Internal server error` |

---

## 参考文档

| 文件 | 说明 |
|------|------|
| `references/TEMPLATE.yaml` | OpenAPI 3.1 模板，可直接复制填充 |
| `references/PATH_CONVENTIONS.md` | RESTful 路径 / 参数 / 响应命名规范 |
| `references/SCHEMA_GUIDE.md` | Schema 设计规范，含 ApiResponse / Pagination 结构 |
| `references/PAGINATION.md` | 分页规范详细说明 |

---

## 推送后向用户确认

推送成功后告知：
- 仓库地址
- 文件清单和大小
- 版本号
- 本次新增/变更的接口说明