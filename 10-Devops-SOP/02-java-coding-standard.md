# Java 编码规范

> 适用范围：企业 Java 应用、服务、库和批处理项目  
> 默认基线：受支持的 LTS JDK、Maven 或 Gradle Wrapper、Google Java Style  
> 注意：具体 JDK 和框架版本必须由项目构建配置决定，本规范不替代项目事实。

## 1. Java 版本与工具链

- **MUST**：生产使用受支持的 LTS JDK，并固定发行版、补丁版本和构建镜像。
- **MUST**：编译、测试、静态分析和 CI 使用相同 Java toolchain。
- **MUST**：提交 Maven Wrapper 或 Gradle Wrapper，固定构建工具版本。
- **MUST**：校验 Wrapper 下载内容的 SHA-256。
- **MUST**：声明源码编码为 UTF-8。
- **SHOULD**：区分构建 JDK、语言级别和产物最低运行时版本。
- **SHOULD**：生产只使用 GA 功能；Preview 功能必须显式审批。
- **MUST NOT**：依赖开发机当前 `JAVA_HOME` 或全局 Maven/Gradle 才能构建。

截至本基线，新项目可评估 Java 25 LTS；已有 Java 21 LTS 项目应根据供应商支持周期制定迁移计划，而不是仅为追逐版本立即升级。

Maven 示例：

```xml
<properties>
  <maven.compiler.release>25</maven.compiler.release>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>
```

## 2. 格式与命名

默认采用 [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)：

- **MUST**：UTF-8，使用空格而不是 Tab。
- **MUST**：每个普通源文件只有一个顶级类型。
- **MUST**：所有 `if`、`else`、`for`、`while`、`do` 使用大括号。
- **MUST**：禁止通配符导入。
- **MUST**：块缩进 2 个空格，通常限制到 100 个 Unicode code point。
- **MUST**：重载方法连续放置。
- **SHOULD**：使用 `google-java-format` 或统一格式化器自动执行。
- **SHOULD**：类名使用名词，方法名使用动词，布尔方法表达谓词，如 `isActive()`。
- **SHOULD NOT**：使用 `Util`、`Helper`、`Manager`、`Common` 等无法表达职责的宽泛名称。
- **MUST NOT**：在代码评审中反复争论可由格式化器决定的排版。

组织可以选择其他格式，但 **MUST** 形成单一、自动化、仓库级配置。

## 3. 包与模块结构

### 3.1 按业务能力组织

推荐：

```text
com.example.product.orders
├── api
│   ├── OrderController
│   ├── CreateOrderRequest
│   └── OrderResponse
├── application
│   ├── CreateOrder
│   └── FindOrder
├── domain
│   ├── Order
│   ├── OrderId
│   └── OrderRepository
├── persistence
│   └── JdbcOrderRepository
└── internal
```

不推荐全系统共享的大包：

```text
com.example.product
├── controller/
├── service/
├── repository/
├── dto/
└── util/
```

规则：

- **MUST**：依赖方向明确且无环。
- **SHOULD**：顶层包按业务能力或 bounded context 划分。
- **MUST NOT**：领域代码依赖 HTTP、ORM、Web 框架或具体消息中间件。
- **MUST NOT**：Controller 直接访问数据库。
- **SHOULD**：跨模块协作使用窄接口、命令/查询 DTO 或领域事件。
- **MUST**：使用最小可见性；只有真正稳定的类型才成为公共接口。
- **SHOULD**：非扩展点类型使用 `final`，或明确限制继承。
- **SHOULD**：Java 模块只 `exports` 稳定公共包。

## 4. 类型与空值

- **MUST**：公共接口和跨模块接口明确 nullness。
- **SHOULD**：采用 JSpecify，在主要范围使用 `@NullMarked`。
- **MUST**：外部输入进入系统时立即校验。
- **SHOULD**：返回空集合而不是 `null` 集合。
- **SHOULD**：查询可能无结果时使用 `Optional<T>`。
- **SHOULD NOT**：把 `Optional` 用作实体字段、集合元素或普通方法参数。
- **MUST NOT**：默认把未标注的第三方接口视为非空。
- **MUST NOT**：使用 `null` 同时表示“未提供”“不存在”“失败”等多个含义。

```java
@NullMarked
public interface OrderRepository {
  Optional<Order> find(OrderId id);

  void save(Order order);

  @Nullable OrderLock tryAcquire(OrderId id);
}
```

## 5. 不可变性与状态

- **SHOULD**：值对象、命令、事件和跨线程消息默认不可变。
- **SHOULD**：优先使用 `record` 或 `final` 类。
- **MUST**：构造时验证不变量。
- **MUST**：对外部可变数组和集合做防御性复制。
- **MUST**：返回不可变集合或副本。
- **MUST NOT**：暴露公共可变静态状态。
- **MUST NOT**：仅因为引用是 `final` 就宣称对象不可变。
- **SHOULD**：使用 `java.time` 类型，不使用 `Date`、`Calendar` 表达新业务模型。

```java
public record OrderSnapshot(
    OrderId id,
    List<LineItem> items,
    Instant updatedAt) {

  public OrderSnapshot {
    Objects.requireNonNull(id);
    Objects.requireNonNull(items);
    Objects.requireNonNull(updatedAt);
    items = List.copyOf(items);
  }
}
```

## 6. 方法与接口设计

- **MUST**：方法只承担一个可清楚命名的责任。
- **SHOULD**：参数表达业务概念；避免多个易混淆的 `String`、`boolean` 参数。
- **SHOULD**：使用值对象替代原始字符串形式的 ID、金额和状态。
- **MUST**：金额使用 `BigDecimal` 或领域金额类型，并明确币种和舍入规则。
- **MUST NOT**：使用 `double` 表示需要精确计算的货币值。
- **SHOULD**：返回结果而不是通过隐藏的全局状态产生结果。
- **SHOULD**：接受依赖，而不是在业务方法中创建数据库、HTTP Client 或系统时钟。
- **MUST NOT**：公开可变内部集合。
- **SHOULD**：复杂布尔条件提取成表达业务含义的方法。

不推荐：

```java
void process(String id, String value, boolean force, boolean sync);
```

推荐：

```java
ProcessResult processOrder(
    OrderId orderId,
    Money amount,
    ProcessingPolicy policy);
```

## 7. 异常与错误处理

- **MUST**：只捕获能够处理的最具体异常。
- **MUST NOT**：空 `catch`、仅记录后继续、或返回伪成功。
- **MUST NOT**：常规业务代码捕获 `Throwable`、`Error` 或无差别捕获 `Exception`。
- **MUST**：包装异常时保留原始 cause。
- **MUST**：使用 try-with-resources 管理 `AutoCloseable`。
- **MUST**：中断要么传播，要么恢复中断标志。
- **SHOULD**：领域异常携带稳定的机器可判定信息，不要求解析 message。
- **MUST**：HTTP 边界将内部异常映射为统一的 RFC 9457 错误。
- **MUST NOT**：向客户端暴露堆栈、SQL、文件路径、类名、主机信息或下游凭据。
- **SHOULD**：同一异常只在拥有处理或观测责任的位置记录一次。

```java
try {
  repository.save(order);
} catch (SQLException cause) {
  throw new OrderPersistenceException(
      "Unable to store order " + order.id(), cause);
}
```

```java
catch (InterruptedException cause) {
  Thread.currentThread().interrupt();
  throw new OperationInterruptedException(cause);
}
```

## 8. 并发与异步

- **MUST**：不存在未保护的数据竞争。
- **SHOULD**：按“不共享 → 不可变 → 线程封闭 → 高层并发工具 → 显式锁”的顺序降低复杂度。
- **MUST**：阻塞 I/O、锁、远程调用和 Future 等待都有超时、取消或明确理由。
- **MUST**：线程池定义所有者、容量、队列上限、拒绝策略和关闭过程。
- **MUST NOT**：使用 `Thread.stop()`。
- **MUST NOT**：认为 `volatile` 能让 `x++` 等复合操作原子化。
- **SHOULD**：优先使用 `java.util.concurrent` 高层抽象。
- **SHOULD**：虚拟线程用于高并发阻塞 I/O，不得被误作 CPU 并行度倍增器。
- **MUST**：即使使用虚拟线程，也限制数据库连接和下游并发。
- **MUST**：并发测试使用 latch、barrier、future 和确定性超时，不依赖 `sleep()` 猜测调度。

## 9. 日志与可观测性

- **MUST**：使用组织标准日志门面，不使用 `System.out`、`System.err` 或散落的 `printStackTrace()`。
- **MUST**：使用结构化、参数化日志。
- **MUST**：至少支持 `trace_id`、`span_id`、`request_id`、`event`、`outcome`、`error_type` 等稳定字段。
- **MUST NOT**：记录密码、token、session ID、私钥、完整支付数据或不必要的个人信息。
- **MUST**：限制外部输入日志字段长度，并防止 CR/LF 日志注入。
- **SHOULD**：正常业务拒绝不全部记录为 `ERROR`。
- **SHOULD**：安全审计日志和普通应用日志使用不同访问、保留和完整性策略。
- **MUST NOT**：把完整请求/响应对象直接序列化到日志。

```java
log.atInfo()
    .addKeyValue("event", "order_created")
    .addKeyValue("order_id", orderId)
    .addKeyValue("outcome", "success")
    .log("Order created");
```

## 10. 测试规范

- **MUST**：测试独立、可重复，并能在 CI 无人工干预运行。
- **MUST**：业务规则有单元测试。
- **MUST**：数据库、消息和 HTTP 集成有集成测试。
- **MUST**：外部 API 有 OpenAPI 契约测试。
- **MUST**：认证、授权和输入边界包含负向测试。
- **MUST**：超时、重试、幂等和并发冲突有对应测试。
- **SHOULD**：测试行为和不变量，不断言私有实现细节。
- **SHOULD**：缺陷修复先加入修复前失败的回归测试。
- **MUST**：时间、随机数、UUID 和外部 I/O 通过可替换接口控制。
- **MUST**：长期 `@Disabled` 必须关联 Issue、负责人和到期时间。
- **SHOULD**：新测试使用 JUnit Jupiter。

命名示例：

```java
@Test
void rejectsUpdateWhenEtagIsStale() {}

@Test
void preventsTenantFromReadingAnotherTenantsOrder() {}
```

## 11. 依赖与构建

- **MUST**：依赖和插件使用确定版本。
- **MUST NOT**：使用动态版本或 Maven version range。
- **SHOULD**：使用 BOM、dependency management 或 version catalog 集中管理版本。
- **MUST**：直接、传递和构建插件依赖都接受漏洞扫描。
- **MUST**：生产构建可从干净环境完成。
- **MUST NOT**：依赖开发者本地缓存中未发布的 artifact。
- **MUST**：区分 compile、runtime 和 test 依赖。
- **MUST NOT**：测试库进入生产运行时。
- **SHOULD**：构建可复现，固定影响产物的时间戳和输入。
- **SHOULD**：遵循 Maven 标准布局：

```text
src/main/java
src/main/resources
src/test/java
src/test/resources
target/
```

## 12. Copilot 生成 Java 代码的规则

路径级 Instructions 建议包含：

```markdown
---
applyTo: "src/**/*.java"
---

- Follow the repository formatter; do not hand-format generated output.
- Preserve package dependency direction.
- Do not add a dependency until existing platform APIs were checked.
- Do not catch broad exceptions or return success-shaped fallbacks.
- Do not log secrets, tokens, request bodies, or personal data.
- Add negative tests for validation and authorization changes.
- For bug fixes, reproduce with a failing test before changing behavior.
- Run ./mvnw verify and report checks that were not run.
```

Copilot 输出 **MUST** 重点审查：

- 是否编造不存在的类、方法或依赖版本；
- 是否绕过认证、授权和租户隔离；
- 是否引入宽泛异常捕获或静默失败；
- 是否用字符串拼接 SQL、命令或 URL；
- 是否记录敏感数据；
- 是否缺少超时、取消和资源关闭；
- 是否修改公开接口却没有兼容性计划；
- 是否只生成成功路径测试。

## 13. 自动化门禁

项目 **SHOULD** 在 `./mvnw verify` 或等效入口中包含：

- 编译；
- 单元测试；
- 集成测试；
- 格式检查；
- 静态分析；
- 依赖漏洞检查；
- OpenAPI 契约检查；
- 架构依赖检查；
- 覆盖率或变异测试策略；
- 禁止 API 和敏感日志规则。

工具由组织选型，本规范不强制具体供应商；关键是本地与 CI 使用同一配置。

## 14. Review 检查清单

- [ ] Java/JDK/Wrapper 版本与项目配置一致
- [ ] 格式化器通过，无手工格式争论
- [ ] 包依赖方向未被破坏
- [ ] 公共接口和 nullness 清晰
- [ ] 值对象和集合没有泄露可变状态
- [ ] 异常未被吞掉且保留 cause
- [ ] 阻塞与远程调用有超时
- [ ] 日志不含敏感信息
- [ ] 依赖必要、固定版本且经过扫描
- [ ] 正向、负向、边界和回归测试充分
- [ ] AI 生成内容经过独立人工确认

## 15. 主要依据

- [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [google-java-format](https://github.com/google/google-java-format)
- [Oracle Secure Coding Guidelines for Java](https://www.oracle.com/java/technologies/javase/seccodeguide.html)
- [JSpecify User Guide](https://jspecify.dev/docs/user-guide/)
- [Java concurrency package](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html)
- [JUnit User Guide](https://docs.junit.org/current/user-guide/)
- [Maven standard directory layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)
- [Maven reproducible builds](https://maven.apache.org/guides/mini/guide-reproducible-builds.html)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

