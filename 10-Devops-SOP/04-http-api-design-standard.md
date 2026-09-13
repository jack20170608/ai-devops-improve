# HTTP API 设计规范

> 适用范围：企业内部、合作方和公共 HTTP/JSON API  
> 原则：遵守 HTTP 语义，使用一致契约，让客户端不依赖实现细节。

## 1. 标准与组织选择

本规范区分：

- **协议标准**：RFC、W3C Recommendation 和 OpenAPI 定义的语义；
- **组织规范**：为了企业内部一致性作出的选择，例如 URI 命名、分页参数和版本策略。

组织规范可以设为内部 MUST，但不得错误宣称为 HTTP 协议要求。

## 2. 资源与 URI

### 2.1 基本规则

- **MUST**：URI 稳定标识资源，而不是 Java 类、数据库表或部署节点。
- **SHOULD**：集合使用复数名词。
- **SHOULD**：使用小写和 kebab-case。
- **MUST**：资源 ID 当作不透明字符串；客户端不得解析内部结构。
- **MUST NOT**：路径中包含 access token、密码、个人敏感信息或可变显示名称。
- **MUST**：路径段和查询参数分别按 RFC 3986 正确编码。
- **MUST NOT**：通过通用字符串替换模拟 URI 编码。
- **SHOULD**：嵌套层级保持浅；只有明确从属关系才使用子资源。

推荐：

```text
/v1/orders
/v1/orders/{orderId}
/v1/orders/{orderId}/items
```

不推荐：

```text
/v1/orderController/findOrderByDatabaseId
/v1/getOrders
/v1/users/user@example.com/orders
```

### 2.2 动作建模

优先将行为建模为资源状态变化。无法自然表达为 CRUD 的业务动作，可建模为子资源：

```http
POST /v1/orders/{orderId}/cancellations
POST /v1/payments/{paymentId}/refunds
```

## 3. HTTP 方法语义

RFC 9110 定义：

| 方法 | Safe | Idempotent | 典型用途 |
|------|------|------------|----------|
| GET | 是 | 是 | 获取资源 |
| HEAD | 是 | 是 | 获取响应元数据 |
| OPTIONS | 是 | 是 | 查询通信选项 |
| POST | 否 | 默认否 | 创建或提交命令 |
| PUT | 否 | 是 | 已知 URI 的完整替换或创建 |
| PATCH | 否 | 默认否 | 部分修改 |
| DELETE | 否 | 是 | 删除资源 |

规则：

- **MUST NOT**：`GET`、`HEAD` 产生扣款、创建、删除等业务副作用。
- **MUST**：`PUT` 表达已知 URI 的完整替换或创建，不用于任意命令。
- **SHOULD**：部分修改使用 `PATCH`，并明确 patch 媒体类型和语义。
- **SHOULD**：创建成功返回 `201 Created` 和 `Location`。
- **SHOULD**：异步受理返回 `202 Accepted` 和状态查询链接。
- **SHOULD**：无响应体的成功操作返回 `204 No Content`。
- **MUST NOT**：`204` 返回消息体。

幂等表示多次相同请求产生的预期服务器效果与一次相同，不代表每次日志、`Date` 或 trace ID 完全一样。

## 4. 状态码

| 场景 | 状态码 | 要求 |
|------|-------:|------|
| 成功读取 | 200 | 有表示的普通成功 |
| 创建成功 | 201 | **SHOULD** 包含 `Location` |
| 异步接受 | 202 | **SHOULD** 提供状态查询 |
| 无响应体成功 | 204 | **MUST NOT** 返回 body |
| 条件 GET 未变化 | 304 | 与 ETag/Last-Modified 一起使用 |
| 语法或协议参数错误 | 400 | JSON 解析、参数形态错误 |
| 认证凭据缺失/无效 | 401 | 通常包含 `WWW-Authenticate` |
| 已认证但无权限 | 403 | 不等同于“未登录” |
| 不存在或隐藏存在性 | 404 | 避免对象枚举 |
| 方法不允许 | 405 | **MUST** 返回 `Allow` |
| 当前状态冲突 | 409 | 状态转换、幂等键冲突 |
| 不支持媒体类型 | 415 | 请求 `Content-Type` 不支持 |
| 内容无法处理 | 422 | 字段或业务约束不成立 |
| 前置条件失败 | 412 | ETag 不匹配 |
| 要求条件请求 | 428 | 强制防丢失更新 |
| 速率限制 | 429 | **SHOULD** 返回 `Retry-After` |
| 内部错误 | 500 | 不泄露实现细节 |
| 临时过载/维护 | 503 | **SHOULD** 返回 `Retry-After` |
| 网关超时 | 504 | 下游未及时响应 |

企业统一采用：

- `400`：请求无法按协议解析，或通用参数形态错误；
- `409`：请求与资源当前状态冲突；
- `422`：请求结构正确，但字段或业务约束无效。

**MUST NOT** 所有失败返回 `200` 并在 JSON 中放 `"success": false`。

## 5. 请求与响应表示

- **MUST**：请求和响应声明正确的 `Content-Type`。
- **SHOULD**：JSON 使用 `application/json` 和 UTF-8。
- **MUST**：未知 JSON 字段的兼容策略明确。
- **SHOULD**：客户端对响应新增字段保持前向兼容。
- **MUST**：字段含义、单位、范围、格式和 nullability 写入 OpenAPI。
- **MUST NOT**：用空字符串、`0` 和 `null` 混合表示“缺失”。
- **SHOULD**：JSON 字段统一使用 lower camelCase。
- **MUST**：金额同时表达数值和币种，或采用企业统一 Money schema。
- **MUST NOT**：使用浮点数表达需要精确计算的金额。

```json
{
  "amount": {
    "value": "129.90",
    "currency": "CNY"
  }
}
```

## 6. 日期与时间

- **MUST**：JSON 中的时间点使用 RFC 3339 且包含 offset。
- **SHOULD**：统一输出 UTC `Z`。
- **MUST NOT**：用无 offset 的 `"2026-09-13 16:48:10"` 表示时间点。
- **MUST**：明确小数秒精度。
- **SHOULD**：Java 使用 `Instant` 或 `OffsetDateTime`。
- **SHOULD**：只有确实表示本地墙上时间时才使用 `LocalDateTime`。
- **MUST**：HTTP `Date`、日期形式 `Retry-After` 和 `Last-Modified` 使用 RFC 9110 `HTTP-date`，不要混用 RFC 3339。

```json
{
  "createdAt": "2026-09-13T08:48:10.784Z",
  "businessDate": "2026-09-13"
}
```

## 7. 错误模型：RFC 9457 Problem Details

所有非成功 API 错误 **MUST** 使用一致的 `application/problem+json`，除非已有稳定且经过治理的领域错误媒体类型。

标准字段：

| 字段 | 含义 |
|------|------|
| `type` | 问题类型的稳定机器标识 |
| `status` | HTTP 状态码的副本 |
| `title` | 问题类型的人类可读摘要 |
| `detail` | 本次问题的人类可读说明 |
| `instance` | 本次问题实例标识 |

规则：

- **MUST**：`type` 使用组织控制的稳定绝对 URI。
- **MUST**：状态行与 JSON `status` 一致；状态行是权威值。
- **MUST NOT**：客户端解析 `title` 或 `detail` 决策。
- **MUST NOT**：错误中暴露堆栈、SQL、类名、主机、文件路径和下游响应。
- **SHOULD**：字段验证错误使用结构化 `errors` 数组。
- **SHOULD**：trace ID 使用独立扩展字段。
- **MUST**：客户端忽略未知扩展字段。

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Request validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "https://api.example.com/problems/instances/01K51",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    {
      "pointer": "/customer/email",
      "code": "invalid_format",
      "message": "Must be a valid email address."
    }
  ]
}
```

## 8. OpenAPI 3.1

- **MUST**：所有外部和跨团队 HTTP API 维护 OpenAPI 3.1 文档。
- **MUST**：描述路径、方法、参数编码、媒体类型、成功和错误响应、认证、分页、限流、ETag 和 trace Header。
- **MUST**：包含统一 RFC 9457 schema。
- **MUST**：CI 校验语法、实现一致性和破坏性变更。
- **SHOULD**：契约先于客户端生成。
- **SHOULD**：复用 `components`，但避免单一巨大企业 schema。
- **MUST NOT**：在 OpenAPI 3.1 中使用 3.0 的 `nullable: true`。

OpenAPI 3.1 可空类型：

```yaml
type: [string, "null"]
```

OpenAPI 是契约的事实来源。Controller 注解、README 示例和生成代码不得各自形成冲突的第二套契约。

## 9. 幂等性与重试

- **MUST**：支付、订单创建、退款等不可安全重复的 `POST` 支持组织定义的 `Idempotency-Key`。
- **MUST**：key 在调用方和操作范围内唯一。
- **MUST**：原子保存 key、请求指纹和最终结果。
- **MUST**：相同 key、相同请求返回原业务结果。
- **MUST**：相同 key、不同请求返回冲突错误。
- **MUST**：定义 key 有效期。
- **MUST NOT**：只在单实例内存保存 key。
- **MUST NOT**：把 key 当认证凭据。
- **SHOULD**：客户端只在结果未知时用相同 key 重试。
- **SHOULD**：重试采用指数退避和随机抖动。
- **MUST**：明确哪些错误可重试；验证失败、权限失败和业务拒绝通常不可重试。

```http
POST /v1/payments HTTP/1.1
Idempotency-Key: 01K51PMVJ7N6X9Z2J4M8SQ5M3K
Content-Type: application/json
```

`Idempotency-Key` 在本基线仍是 IETF Internet-Draft，不得描述成 RFC 9110 已定义的 Header。

## 10. 缓存

- **MUST**：每个 `GET`/`HEAD` 端点明确缓存策略。
- **MUST**：包含凭据、token 或高度敏感个人信息的响应使用适当策略，通常是：

```http
Cache-Control: no-store
```

- **SHOULD**：稳定公开资源使用显式 `max-age` 和验证器。
- **MUST**：内容协商时正确设置 `Vary`。
- **MUST NOT**：把 `no-cache` 误认为“禁止存储”；它表示复用前必须验证。
- **MUST**：评估带 `Authorization` 响应的共享缓存风险。

## 11. 乐观并发控制

可被多个参与者修改的重要资源 **MUST** 支持乐观并发：

```http
GET /v1/orders/ord_789 HTTP/1.1
```

```http
HTTP/1.1 200 OK
ETag: "ord_789-v17"
```

```http
PUT /v1/orders/ord_789 HTTP/1.1
If-Match: "ord_789-v17"
```

版本已变化时：

```http
HTTP/1.1 412 Precondition Failed
Content-Type: application/problem+json
```

规则：

- **MUST**：重要更新使用 `If-Match` 或等效条件请求。
- **SHOULD**：ETag 是不透明值。
- **MUST NOT**：客户端解析 ETag 中的数据库版本。
- **MUST NOT**：仅依赖客户端提交的 `updatedAt` 防止丢失更新。
- **SHOULD**：更新成功返回新 ETag。
- **MAY**：缺少条件 Header 时返回 `428 Precondition Required`。

## 12. 分页、过滤与排序

这些属于组织选择，并没有统一的通用 IETF JSON API 格式。本规范选择：

```http
GET /v1/orders?limit=50&cursor=opaque-token&sort=-createdAt,id
```

- **MUST**：服务端限制最大页大小。
- **SHOULD**：大规模变化数据使用 cursor/keyset pagination。
- **MUST**：cursor 不透明、受完整性保护，并绑定过滤和排序条件。
- **MUST**：排序包含唯一、稳定的最终 tie-breaker。
- **MUST**：定义默认排序，不依赖数据库自然顺序。
- **MUST**：过滤字段和操作符使用 allowlist。
- **MUST**：限制过滤数量、长度和计算复杂度。
- **MUST NOT**：直接暴露 SQL、JPQL 或任意表达式。
- **SHOULD**：非法过滤或排序返回 `400`/`422` Problem Details。
- **SHOULD**：通过 RFC 8288 `Link` 提供 `next`、`prev`。

```http
Link: </v1/orders?limit=50&cursor=next-token&sort=-createdAt,id>; rel="next"
```

```json
{
  "items": [],
  "page": {
    "nextCursor": "next-token",
    "hasMore": true
  }
}
```

## 13. API 版本与兼容性

本规范对公共 API 推荐路径主版本：

```text
/v1/orders
/v2/orders
```

路径版本是组织选择，不是 HTTP 要求。

- **MUST**：先定义 breaking change。
- **MUST**：优先非破坏性演进。
- **MUST**：客户端忽略未知响应字段。
- **MUST NOT**：无新版本删除或重命名字段。
- **MUST NOT**：改变字段类型、单位、时区或 nullability。
- **MUST NOT**：改变状态码和默认排序语义。
- **MUST NOT**：把可选请求字段直接改成必填。
- **SHOULD**：只在真实不兼容变化时升级主版本。
- **MUST**：发布弃用政策、迁移指南和停止服务日期。
- **MUST**：CI 对 OpenAPI 执行 breaking-change 检查。

通常可兼容：

- 新增可选请求字段；
- 新增响应字段；
- 新增端点；
- 新增客户端必须忽略的扩展。

是否兼容仍需结合客户端行为和契约测试确认。

## 14. Rate Limiting

- **MUST**：超过限制返回 `429 Too Many Requests` 和 Problem Details。
- **SHOULD**：返回 `Retry-After`。
- **MUST**：说明限制维度，如 tenant、用户、凭据、资源和时间窗。
- **MUST NOT**：多租户已认证 API 仅按来源 IP 限制。
- **MUST**：客户端重试建议包含随机抖动。
- **MUST NOT**：把剩余额度当作服务保证。
- **SHOULD**：在 OpenAPI 中记录限流行为。

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/problem+json
```

通用 RateLimit Header 在本基线仍需按 IETF 草案管理，采用前必须固定所依据的草案版本。

## 15. Trace 与请求关联

- **MUST**：支持 W3C `traceparent`，并传播有效值。
- **MUST**：无有效值时生成新 trace。
- **MUST**：校验外部 trace metadata。
- **MUST NOT**：把 trace ID 当身份、权限或防重放凭据。
- **SHOULD**：保留 `X-Request-ID` 时明确它只用于请求关联。
- **MUST NOT**：在 `tracestate` 中放用户 ID、邮箱或个人信息。
- **SHOULD**：Problem Details 包含 `traceId` 扩展。

## 16. 安全要求

- **MUST**：TLS 保护传输。
- **MUST**：认证和授权在服务端执行。
- **MUST**：所有输入按信任边界校验。
- **MUST**：使用请求和响应大小限制。
- **MUST**：设置连接、读取、写入和总体超时。
- **MUST**：敏感响应明确 `Cache-Control`。
- **MUST NOT**：在 URI 中传递凭据。
- **MUST NOT**：错误泄露内部实现。
- **MUST**：CORS、CSRF 和 Cookie 策略与客户端类型一致。
- **MUST**：Webhook 验证来源、签名、时间窗和重放。
- **MUST**：文件上传限制大小、类型、名称和存储位置，并进行恶意内容检查。

## 17. 完整请求示例

```http
POST /v1/orders HTTP/1.1
Host: api.example.com
Authorization: <redacted>
Content-Type: application/json
Accept: application/json
Idempotency-Key: 01K51PMVJ7N6X9Z2J4M8SQ5M3K
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

{
  "customerId": "cus_123",
  "requestedAt": "2026-09-13T08:48:10.784Z",
  "items": [
    {
      "productId": "prd_456",
      "quantity": 2
    }
  ]
}
```

```http
HTTP/1.1 201 Created
Location: /v1/orders/ord_789
ETag: "ord_789-v1"
Content-Type: application/json
Cache-Control: no-store
```

## 18. Copilot 实现 API 时的约束

路径级 Instructions 建议：

```markdown
---
applyTo: "src/main/java/**/*Controller.java,docs/api/**/*.yaml"
---

- OpenAPI 3.1 is the contract source of truth.
- Preserve HTTP method semantics from RFC 9110.
- Use RFC 9457 Problem Details for errors.
- Do not expose stack traces, SQL, class names, or downstream bodies.
- Add authorization, validation, idempotency, timeout, and concurrency tests.
- Run the OpenAPI compatibility check before completing the task.
- Never add credentials or personal data to examples.
```

Copilot 生成 API 代码时 **MUST** 核对：

- 方法和状态码语义；
- 认证与对象级授权；
- 输入长度、范围和 allowlist；
- 幂等和重试；
- ETag 并发；
- 超时与下游失败；
- 错误模型；
- 缓存和敏感信息；
- OpenAPI 与实现一致；
- 正向、负向和兼容性测试。

## 19. Review 检查清单

- [ ] URI 表达资源而非实现
- [ ] HTTP 方法没有违反 safe/idempotent 语义
- [ ] 状态码准确且错误不是统一 200
- [ ] 认证和对象级授权完整
- [ ] 输入验证包含长度、范围和业务语义
- [ ] 错误采用 RFC 9457 且无内部泄漏
- [ ] 时间使用 RFC 3339 和明确 offset
- [ ] OpenAPI 3.1 与实现一致
- [ ] 高风险 POST 支持幂等键
- [ ] 重要更新使用 ETag/If-Match
- [ ] 分页有上限、稳定排序和不透明 cursor
- [ ] 限流、重试和超时明确
- [ ] Breaking change 已阻断或进入新版本
- [ ] Trace 可传播但不含个人信息

## 20. 主要依据

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 3986: URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)
- [RFC 3339: Date and Time](https://www.rfc-editor.org/rfc/rfc3339.html)
- [RFC 9457: Problem Details](https://www.rfc-editor.org/rfc/rfc9457.html)
- [RFC 6585: Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)
- [RFC 8288: Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)
- [OpenAPI 3.1.1](https://spec.openapis.org/oas/v3.1.1.html)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [IETF Idempotency-Key draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [IETF RateLimit fields draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)

