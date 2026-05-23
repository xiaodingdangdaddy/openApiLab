# Schema 设计规范

## 一、Schema 设计原则

1. **直接映射老接口响应字段**，不做过度的抽象和归一
2. **嵌套层级与老接口保持一致**，避免引入不存在的层级
3. **字段类型优先使用 string**，复杂结构再用 object / array
4. **枚举值严格按老接口**，不添加老接口中没有的枚举
5. **必选字段由老接口决定**，OpenAPI 不做强制约束（除非明确）

---

## 二、Schema 结构模板

### 2.1 列表项 Schema

```yaml
<Resource>:
  type: object
  description: <资源>描述
  properties:
    id:
      type: string
      description: <ID说明>
    name:
      type: string
      description: <名称说明>
    # 直接列出字段，不做过多嵌套
    fieldA:
      type: string
      description: 字段A说明
    fieldB:
      type: integer
      format: int64
      description: 字段B说明
```

### 2.2 分页列表响应 Schema

```yaml
<Resource>ListResponse:
  type: object
  properties:
    code:
      type: string
      example: success
    msg:
      type: string
      example: Query success
    data:
      type: object
      properties:
        list:
          type: array
          items: { $ref: '#/components/schemas/<Resource>' }
        pagination: { $ref: '#/components/schemas/PaginationInfo' }
```

### 2.3 详情响应 Schema

```yaml
<Resource>DetailResponse:
  type: object
  properties:
    code:
      type: string
      example: success
    msg:
      type: string
      example: Query success
    data: { $ref: '#/components/schemas/<Resource>' }
```

### 2.4 通用结构复用

```yaml
components:
  schemas:
    ApiResponse:
      type: object
      properties:
        code: { type: string, example: success }
        msg: { type: string, example: Query success }

    PaginationInfo:
      type: object
      properties:
        pageNum: { type: integer, example: 1 }
        pageSize: { type: integer, example: 20 }
        total: { type: integer, example: 156 }
        pages: { type: integer, example: 8 }

    ErrorResponse:
      type: object
      properties:
        code: { type: string, example: fail }
        msg: { type: string, example: Resource not found }
```

---

## 三、字段类型选择

| 老接口类型 | OpenAPI 类型 | 注意 |
|-----------|------------|------|
| String / VARCHAR | `type: string` | — |
| INT / BIGINT | `type: integer` | 可加 `format: int64` |
| FLOAT / DOUBLE | `type: number` | 可加 `format: double` |
| BOOLEAN | `type: boolean` | — |
| DATE / DATETIME | `type: string` + `format: date-time` | — |
| JSON / OBJECT | `type: object` | 描述结构或说"见原接口" |
| LIST / ARRAY | `type: array` + `items` | — |

---

## 四、字段描述规范

每个字段必须有 `description`：

```yaml
properties:
  jobInsId:
    type: string
    description: 稽核任务实例ID（每次核查运行生成唯一ID，即一个结果快照）
  score:
    type: number
    format: double
    description: 质量评分（0-100）
  startTime:
    type: integer
    format: int64
    description: 开始时间（Unix毫秒）
```

---

## 五、嵌套对象处理

如果老接口返回深层嵌套，BFF 层会做字段截断，设计 Schema 时：

**保留必要嵌套，不展开过度抽象的深层结构：**

```yaml
# ✅ 保留嵌套（老接口结构）
relatedPhysicalModels:
  type: array
  items:
    type: object
    properties:
      physicalModelName: { type: string }
      physicalColumns: { type: array, items: { type: object } }

# ❌ 不要展开成扁平结构
physicalModelName: { type: string }
physicalColumnName: { type: string }
# —— 这会丢失结构语义
```

---

## 六、枚举值处理

枚举值严格按老接口，不添加新枚举：

```yaml
# ✅ 按老接口枚举
status:
  type: string
  enum: [success, failed, warning]

# ❌ 不要添加老接口中没有的枚举
status:
  type: string
  enum: [success, failed, warning, running, pending]
  # —— running 和 pending 如果老接口没有，不加
```

---

## 七、多态类型标注（可选，当前不需要 TMF 对齐）

如需兼容 TMF 规范，可增加：

```yaml
LogicalModel:
  type: object
  description: 逻辑模型
  properties:
    @type: { type: string, example: LogicalModel }
    @baseType: { type: string, example: LogicalModelBase }
    @schemaLocation:
      type: string
      format: uri
      example: https://api.example.com/schema/logical-model
    id: { type: string }
    name: { type: string }
```

**当前不需要 TMF 对齐，此项为可选项，不强制要求。**