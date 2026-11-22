# 18 API 契约与事件规范

> 本章把 14 章的端点示例补成可执行的契约:OpenAPI/Protobuf 组织方式、核心 Schema、完整端点目录、分页/幂等/一致性头、事件信封与 Webhook 规范、契约测试与弃用流程。

## 1. 契约优先工作流

```
需求 → 契约变更提案(OpenAPI/Proto)
  → 评审(兼容性/安全/性能)
  → 生成服务端 stub + 客户端 SDK
  → 契约测试(mock/回放)
  → 实现 → 集成测试 → 发布
```

原则:

- 契约是**唯一事实源**;文档、SDK、mock 都从契约生成。
- 破坏性变更必须走提案 + 弃用期 + 迁移指南。
- 契约仓库与实现仓库分离,变更通过 PR 评审。
- 每个字段有描述、示例、是否必填、是否弃用。

### 1.1 仓库结构

```
contracts/
  openapi/
    ontology.v1.yaml
    actions.v1.yaml
    schemas.v1.yaml
    admin.v1.yaml
  proto/
    ontology/v1/*.proto
  events/
    envelope.json
    object-events.v1.json
  examples/
    query-objects.json
    apply-action.json
```

## 2. OpenAPI 组织

### 2.1 顶层结构

```yaml
openapi: 3.1.0
info:
  title: Ontology Platform API
  version: 1.0.0
servers:
  - url: https://api.example.com/v1
security:
  - bearerAuth: []
tags:
  - name: Objects
  - name: Actions
  - name: Schemas
paths: {}
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas: {}
  parameters:
    PageSize:
      name: pageSize
      in: query
      schema: {type: integer, minimum: 1, maximum: 1000, default: 100}
    PageToken:
      name: pageToken
      in: query
      schema: {type: string}
    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: true
      schema: {type: string, maxLength: 128}
```

### 2.2 通用响应头

| 头 | 说明 |
|---|---|
| `X-Request-Id` | 请求追踪 ID |
| `X-Ontology-Version` | 元数据版本 |
| `X-Index-Watermark` | 索引水位 |
| `X-Policy-Version` | 策略版本 |
| `X-Stale` | 数据是否可能过期 |
| `ETag` / `If-Match` | 乐观并发 |
| `Retry-After` | 限流/不可用 |
| `Deprecation` / `Sunset` | 弃用信息 |

## 3. 核心 Schema

### 3.1 Object

```yaml
Object:
  type: object
  required: [objectType, primaryKey, version, properties]
  properties:
    objectType: {type: string}
    primaryKey: {type: string}
    version: {type: integer, format: int64}
    properties:
      type: object
      additionalProperties: true
    links:
      type: object
      additionalProperties:
        $ref: '#/components/schemas/LinkPage'
    deleted: {type: boolean, default: false}
```

### 3.2 LinkPage

```yaml
LinkPage:
  type: object
  properties:
    linkType: {type: string}
    objects:
      type: array
      items: {$ref: '#/components/schemas/ObjectRef'}
    nextPageToken: {type: string, nullable: true}
ObjectRef:
  type: object
  required: [objectType, primaryKey]
  properties:
    objectType: {type: string}
    primaryKey: {type: string}
    displayName: {type: string}
```

### 3.3 ObjectSet

```yaml
ObjectSet:
  oneOf:
    - $ref: '#/components/schemas/ObjectTypeSet'
    - $ref: '#/components/schemas/FilterSet'
    - $ref: '#/components/schemas/LinkSet'
    - $ref: '#/components/schemas/UnionSet'
    - $ref: '#/components/schemas/IntersectSet'
    - $ref: '#/components/schemas/SubtractSet'

FilterSet:
  type: object
  required: [op, input, predicate]
  properties:
    op: {const: filter}
    input: {$ref: '#/components/schemas/ObjectSet'}
    predicate: {$ref: '#/components/schemas/Predicate'}

Predicate:
  oneOf:
    - type: object
      required: [and]
      properties: {and: {type: array, items: {$ref: '#/components/schemas/Predicate'}}}
    - type: object
      required: [or]
      properties: {or: {type: array, items: {$ref: '#/components/schemas/Predicate'}}}
    - type: object
      required: [eq]
      properties: {eq: {type: array, items: {}}}
    - type: object
      required: [gt]
      properties: {gt: {type: array, items: {}}}
```

### 3.4 Action

```yaml
ActionRequest:
  type: object
  required: [parameters]
  properties:
    parameters:
      type: object
      additionalProperties: true
    branch: {type: string}
    scenario: {type: string}
ActionResponse:
  type: object
  required: [actionRunId, status]
  properties:
    actionRunId: {type: string}
    status: {type: string, enum: [SUCCESS, FAILED, PARTIAL]}
    editSeq: {type: integer, format: int64}
    edits:
      type: array
      items:
        type: object
        properties:
          objectType: {type: string}
          primaryKey: {type: string}
          version: {type: integer, format: int64}
    sideEffects:
      type: array
      items:
        type: object
        properties:
          type: {type: string}
          status: {type: string, enum: [QUEUED, SENT, FAILED]}
```

### 3.5 Error

```yaml
Error:
  type: object
  required: [error]
  properties:
    error:
      type: object
      required: [code, message, requestId, retryable]
      properties:
        code: {type: string}
        message: {type: string}
        details:
          type: array
          items:
            type: object
            additionalProperties: true
        requestId: {type: string}
        retryable: {type: boolean}
```

## 4. 完整端点目录

### 4.1 元数据与模式

| 方法 | 路径 | 说明 | 作用域 |
|---|---|---|---|
| GET | `/ontologies` | 列出 Ontology | `ontology:read` |
| GET | `/ontologies/{o}` | Ontology 详情 | `ontology:read` |
| GET | `/ontologies/{o}/object-types` | 列出对象类型 | `schema:read` |
| GET | `/ontologies/{o}/object-types/{t}` | 对象类型详情 | `schema:read` |
| POST | `/ontologies/{o}/object-types` | 创建(草稿) | `schema:write` |
| PATCH | `/ontologies/{o}/object-types/{t}` | 更新(If-Match) | `schema:write` |
| POST | `/ontologies/{o}/object-types/{t}/publish` | 发布 | `schema:publish` |
| POST | `/ontologies/{o}/object-types/{t}/deprecate` | 弃用 | `schema:publish` |
| GET | `/ontologies/{o}/object-types/{t}/diff` | 版本差异 | `schema:read` |
| GET | `/ontologies/{o}/link-types` | 链接类型 | `schema:read` |
| POST | `/ontologies/{o}/link-types` | 创建 | `schema:write` |
| GET | `/ontologies/{o}/interfaces` | 接口 | `schema:read` |
| GET | `/ontologies/{o}/value-types` | 值类型 | `schema:read` |
| GET | `/ontologies/{o}/dependencies` | 依赖/血缘 | `schema:read` |

### 4.2 对象与查询

| 方法 | 路径 | 说明 | 作用域 |
|---|---|---|---|
| GET | `/ontologies/{o}/objects/{t}` | 列表/过滤 | `objects:{t}:read` |
| GET | `/ontologies/{o}/objects/{t}/{pk}` | 单对象 | `objects:{t}:read` |
| POST | `/ontologies/{o}/objects/{t}/search` | 搜索 | `objects:{t}:read` |
| POST | `/ontologies/{o}/objects/{t}/aggregate` | 聚合 | `objects:{t}:read` |
| GET | `/ontologies/{o}/objects/{t}/{pk}/links/{link}` | 链接对象 | `objects:{t}:read` |
| POST | `/ontologies/{o}/objects/{t}/batch-get` | 批量取 | `objects:{t}:read` |
| POST | `/ontologies/{o}/object-sets` | 创建对象集 | `object-sets:write` |
| GET | `/ontologies/{o}/object-sets/{id}` | 获取定义 | `object-sets:read` |
| POST | `/ontologies/{o}/object-sets/{id}/objects` | 加载对象 | `object-sets:read` |
| POST | `/ontologies/{o}/object-sets/{id}/aggregate` | 聚合 | `object-sets:read` |
| DELETE | `/ontologies/{o}/object-sets/{id}` | 删除 | `object-sets:write` |

### 4.3 动作与函数

| 方法 | 路径 | 说明 | 作用域 |
|---|---|---|---|
| GET | `/ontologies/{o}/actions` | 列出动作 | `actions:read` |
| GET | `/ontologies/{o}/actions/{a}` | 动作定义 | `actions:read` |
| POST | `/ontologies/{o}/actions/{a}/apply` | 提交 | `actions:{a}:execute` |
| POST | `/ontologies/{o}/actions/{a}/apply-batch` | 批量 | `actions:{a}:execute` |
| GET | `/ontologies/{o}/action-runs/{id}` | 运行详情 | `actions:read` |
| POST | `/ontologies/{o}/queries/{q}/execute` | 查询函数 | `functions:{q}:execute` |
| GET | `/ontologies/{o}/functions` | 函数列表 | `functions:read` |

### 4.4 版本、分支、情景

| 方法 | 路径 | 说明 | 作用域 |
|---|---|---|---|
| GET | `/ontologies/{o}/branches` | 分支列表 | `branch:read` |
| POST | `/ontologies/{o}/branches` | 创建分支 | `branch:write` |
| POST | `/ontologies/{o}/branches/{b}/rebase` | 变基 | `branch:write` |
| GET | `/ontologies/{o}/branches/{b}/diff` | 差异 | `branch:read` |
| POST | `/ontologies/{o}/proposals` | 创建提案 | `proposal:write` |
| POST | `/ontologies/{o}/proposals/{id}/approve` | 审批 | `proposal:approve` |
| POST | `/ontologies/{o}/proposals/{id}/merge` | 合并 | `proposal:merge` |
| POST | `/ontologies/{o}/scenarios` | 创建情景 | `scenario:write` |
| GET | `/ontologies/{o}/scenarios/{id}/diff` | 差异 | `scenario:read` |

### 4.5 安全、交付、运维

| 方法 | 路径 | 说明 | 作用域 |
|---|---|---|---|
| GET | `/ontologies/{o}/policies` | 策略列表 | `policy:read` |
| POST | `/ontologies/{o}/policies` | 创建策略 | `policy:write` |
| POST | `/ontologies/{o}/policies/{id}/publish` | 发布 | `policy:publish` |
| POST | `/ontologies/{o}/policies/explain` | 解释决策 | `policy:read` |
| GET | `/audit/events` | 审计查询 | `audit:read` |
| GET | `/products` | 产品列表 | `product:read` |
| POST | `/products/{id}/install` | 安装 | `product:install` |
| POST | `/products/{id}/upgrade` | 升级 | `product:install` |
| GET | `/health` | 健康检查 | 无 |
| GET | `/metrics` | 指标 | `ops:read` |

## 5. 分页、幂等与一致性规范

### 5.1 分页

```http
GET /objects/Flight?pageSize=100&pageToken=...
```

响应:

```json
{
  "data": [],
  "nextPageToken": "...",
  "snapshot": {"watermark": 981234, "policyVersion": 42}
}
```

规则:

- `pageToken` 不透明、签名;包含查询指纹、快照、策略版本。
- 最大 `pageSize` 由服务端限制;超出返回 400。
- 游标过期返回 `QUERY004`;客户端需重新查询。
- 排序必须稳定;无排序时按主键排序。

### 5.2 幂等

- 写操作必须带 `Idempotency-Key`(建议),服务端默认保存 24 小时(平台级 TTL 配置项)。
- 相同 key + 相同请求 → 返回首次结果。
- 相同 key + 不同请求 → `ACTION010`。
- 幂等记录与动作运行同事务。

**TTL 过期语义**(与 11 §7.3 一致):

- 清理条件为"TTL 到期 **且** 该动作的所有 outbox 副作用已投递完成",否则顺延——保证消费方看到过成功响应的 key 不会在副作用未完成时被重放。
- TTL 过期后同 key 视为新请求重新执行;响应中 `actionRunId` 会不同,客户端据此识别重放并触发对账。
- 长于 TTL 的业务流程不应依赖幂等 key,应使用业务单号外部对账。

### 5.3 一致性

| 请求头 | 作用 |
|---|---|
| `X-Consistency: strong` | 读己之写/最新(可能更慢) |
| `X-Consistency: eventual` | 允许索引延迟(默认) |
| `If-Match: <version>` | 乐观并发更新 |
| `X-Snapshot: <token>` | 跨请求固定快照 |

`strong` 的语义按查询类型分级(见 10 §9.1):

| 查询类型 | `eventual`(默认) | `strong` |
|---|---|---|
| 点查 | 有界 overlay 读己之写 | 同左(通常已满足) |
| 列表/搜索 | 快照 + 30s overlay 窗口 | overlay 超窗时降级直读权威对象库 |
| 聚合/遍历 | 索引水位(最终一致) | **放弃索引,主存重算**;延迟与吞吐受主存限制,应设并发上限并限流 |

`strong` 聚合是逃生通道而非默认路径;高频调用会打穿主存,网关对 `strong` 聚合单独限流。

## 6. gRPC 契约

内部/高性能场景使用 Protobuf:

```proto
syntax = "proto3";
package ontology.v1;

service ObjectService {
  rpc GetObject(GetObjectRequest) returns (Object);
  rpc ListObjects(ListObjectsRequest) returns (ListObjectsResponse);
  rpc SearchAround(SearchAroundRequest) returns (SearchAroundResponse);
}

message Object {
  string object_type = 1;
  string primary_key = 2;
  int64 version = 3;
  google.protobuf.Struct properties = 4;
}

message ListObjectsRequest {
  string ontology = 1;
  string object_type = 2;
  string filter = 3;
  int32 page_size = 4;
  string page_token = 5;
}
```

约定:

- 字段号永不复用;删除字段用 `reserved`。
- 可选字段用 `optional`/wrapper 区分默认值。
- 错误用 `google.rpc.Status` + details。
- 流式接口用于大结果/事件订阅。

## 7. 错误码注册表

| 域 | 前缀 | 示例 |
|---|---|---|
| 元模型 | META | META001–META008 |
| 查询 | QUERY | QUERY001–QUERY007 |
| 动作 | ACTION | ACTION001–ACTION010 |
| 认证/权限 | AUTH/PERM | AUTH001–003, PERM001–003 |
| 租户/限流 | TENANT/RATE | TENANT001, RATE001 |
| 平台 | INT | INT001–002 |

规则:

- 错误码稳定;文案可本地化。
- `details` 结构按域定义,便于客户端处理。
- 新增错误码需更新注册表与 SDK 枚举。
- 不在错误信息中泄露对象/属性的敏感值。

## 8. 事件规范

### 8.1 事件信封

```json
{
  "specVersion": "1.0",
  "id": "evt_01J...",
  "type": "ontology.object.updated",
  "source": "ontology-platform",
  "subject": "Flight/CA1234-2026-11-22",
  "time": "2026-11-22T07:12:00Z",
  "tenantId": "t_acme",
  "ontologyId": "o_airline",
  "dataSchema": "https://schemas.example.com/ontology/object-updated/1.0",
  "data": {
    "objectType": "Flight",
    "primaryKey": "CA1234-2026-11-22",
    "version": 1843,
    "changed": ["status", "scheduledDeparture"],
    "actionType": "DelayFlight",
    "actionRunId": "run_7f3a"
  },
  "traceId": "req_9c2"
}
```

### 8.2 事件类型

| 类型 | 触发 |
|---|---|
| `ontology.object.created` | 对象创建 |
| `ontology.object.updated` | 属性修改 |
| `ontology.object.deleted` | 对象删除 |
| `ontology.link.created` | 链接创建 |
| `ontology.link.deleted` | 链接删除 |
| `ontology.schema.published` | Schema 发布 |
| `ontology.action.completed` | 动作成功 |
| `ontology.action.failed` | 动作失败 |
| `ontology.policy.changed` | 策略变更 |

### 8.3 兼容性

- 事件 Schema 版本化(`dataSchema`)。
- 只增字段,不删/不改语义;破坏性变更发新版本。
- 消费方忽略未知字段。
- 保留期与回放:按主题配置;支持从 offset 重放。

### 8.4 投递语义

- 至少一次;分区键 `tenantId + objectType + primaryKey` 保证同对象有序。
- 消费方幂等(按事件 id/对象版本)。
- 失败重试 + DLQ;持续失败告警。
- 事件不替代查询;消费方需能按需回查当前状态。

## 9. Webhook 规范

### 9.1 请求

```http
POST https://customer.example.com/hooks/ontology
Content-Type: application/json
X-Webhook-Id: wh_123
X-Webhook-Timestamp: 1763789520
X-Webhook-Signature: v1=base64(hmac_sha256(secret, timestamp + "." + body))
```

### 9.2 安全与可靠性

- HMAC 签名 + 时间戳(±5 分钟)防重放。
- 仅允许 HTTPS;限制重定向;防 SSRF。
- 超时 10 s;指数退避重试(如 1m/5m/30m/2h/6h)。
- 连续失败进入 DLQ;提供手动重放。
- 至少一次;接收方按 `X-Webhook-Id` 幂等。

## 10. 契约测试与弃用

### 10.1 契约测试

| 测试 | 内容 |
|---|---|
| Schema 校验 | 响应符合 OpenAPI/Proto |
| 兼容性 | 旧客户端可解析新响应 |
| 示例回放 | 请求/响应示例可执行 |
| 错误码 | 错误响应符合注册表 |
| 事件 | 消费方契约测试 |

### 10.2 弃用流程

1. 标记 `deprecated: true`,加 `Deprecation`/`Sunset` 头。
2. 发布迁移指南与替代端点。
3. 监控旧版本调用量。
4. 弃用期 ≥ 6 个月(按客户合同)。
5. 下线前通知 + 最终审计。

## 11. 示例:查询对象(完整)

```http
GET /v1/ontologies/o_airline/objects/Flight?pageSize=2&pageToken=
Authorization: Bearer <token>
X-Consistency: strong
```

```json
{
  "data": [
    {
      "objectType": "Flight",
      "primaryKey": "CA1234-2026-11-22",
      "version": 1843,
      "properties": {
        "flightNumber": "CA1234",
        "status": "DELAYED",
        "scheduledDeparture": "2026-11-22T09:00:00Z",
        "aircraftId": "B-1234"
      }
    }
  ],
  "nextPageToken": "eyJ...",
  "snapshot": {"watermark": 981234, "policyVersion": 42},
  "stale": false
}
```

## 12. 开放问题

| 问题 | 说明 |
|---|---|
| OpenAPI 3.1 vs 3.0 | 工具链兼容性 |
| 批量接口上限 | 请求体大小/事务大小 |
| 事件 Schema 注册中心 | 自建 vs 标准(Confluent/AsyncAPI) |
| 错误码粒度 | 过细难维护,过粗难处理 |
| GraphQL 叠加 | 是否需要以及如何限制复杂度 |

> 上一章:[17-图分析搜索与时空数据.md](17-图分析搜索与时空数据.md) | 下一章:[19-计量计费与多区域双活.md](19-计量计费与多区域双活.md) | 返回目录:[README.md](README.md)
