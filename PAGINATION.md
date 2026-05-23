# 分页规范

## 一、通用分页响应结构

所有列表查询接口统一使用以下分页结构：

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

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| pageNum | integer | 当前页码，从 1 开始 | 1 |
| pageSize | integer | 每页条数 | 20 |
| total | integer | 符合条件的总记录数 | 156 |
| pages | integer | 总页数 | 8 |

---

## 二、分页参数定义（OpenAPI yaml）

```yaml
parameters:
  - name: pageNum
    in: query
    schema:
      type: integer
      default: 1
      minimum: 1
    description: 页码，从1开始

  - name: pageSize
    in: query
    schema:
      type: integer
      default: 20
      minimum: 1
      maximum: 500
    description: 每页条数，最大500
```

---

## 三、BFF 层分页处理

当老接口无分页能力时，BFF 封装层负责补充分页逻辑：

```
BFF 层分页处理流程：

1. 调老接口获取全量数据（或按 pageSize 上限拉取）
2. 按 pageNum + pageSize 计算偏移量
   offset = (pageNum - 1) * pageSize
3. 取当页数据
   list = data.slice(offset, offset + pageSize)
4. 补全 total / pages / count 字段
```

---

## 四、不使用 offset/limit 的原因

TMF 规范使用 offset/limit，但本项目使用 pageNum/pageSize，因为：

| 对比项 | offset/limit | pageNum/pageSize |
|--------|-------------|-----------------|
| 语义 | 偏移量/每页条数 | 页码/每页条数 |
| 计算复杂度 | 需前端计算 offset | BFF 封装层透明处理 |
| 适用场景 | 深度分页（跳页） | 普通翻页（首页/末页） |
| 老接口兼容性 | 不一致 | 一致 |

**当前项目 pageNum/pageSize 已足够满足 Dashboard 场景，不强制改为 offset/limit。**

---

## 五、分页场景处理

### 5.1 单页内全量数据

如果数据量小，不需要分页：

```json
{
  "data": {
    "list": [...],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 5,
      "pages": 1
    }
  }
}
```

### 5.2 空结果

```json
{
  "data": {
    "list": [],
    "pagination": {
      "pageNum": 1,
      "pageSize": 20,
      "total": 0,
      "pages": 0
    }
  }
}
```

### 5.3 跨页数据更新

当 `list` 在查询过程中发生变化时（新增/删除），`pagination.total` 可能不精确，BFF 层应在响应 `msg` 中注明：

```json
{
  "code": "success",
  "msg": "Query success (total may be approximate during concurrent updates)"
}
```