# AI Vibe Coding 的代码评审和测试最佳实践

> 前置知识：建议先阅读 [AI Vibe Coding 的 Java 代码设计规范](02-java-coding-standards.md)

在 AI vibe coding 模式下，代码评审和测试有了新的意义：它们不仅是质量保障，更是人类开发者监督和纠正 AI 的关键机制。AI 生成的代码虽然快，但需要人类的审查来确保其正确性、安全性和可维护性。

## AI Vibe Coding 中的代码评审变革

### 评审角色的转变

传统代码评审是开发者之间的相互审查，而 AI vibe coding 中的评审是**人类开发者对 AI 输出的审查**。这带来了新的挑战：

- **AI 不会质疑自己**：AI 生成代码时不会怀疑自己的方案
- **上下文理解有限**：AI 可能不完全理解业务背景
- **可能产生"幻觉"**：AI 可能生成看似合理但实际错误的代码
- **安全风险**：AI 可能生成包含安全漏洞的代码

## AI 辅助代码评审最佳实践

### 1. 快速审查清单（AI 生成代码必查）

| 检查项 | 描述 | 重要性 |
|--------|------|--------|
| 业务逻辑正确性 | 代码是否正确实现了业务需求 | 🔴 关键 |
| 边界条件处理 | 空值、异常值、超出范围等边界情况 | 🔴 关键 |
| 安全漏洞 | SQL 注入、XSS、权限校验等 | 🔴 关键 |
| 资源泄漏 | 数据库连接、文件句柄、线程等资源 | 🟠 高 |
| 性能问题 | N+1 查询、全表扫描、无限循环等 | 🟠 高 |
| 代码可读性 | 命名、注释、结构是否清晰 | 🟡 中 |

### 2. 评审流程

```
1. AI 生成代码
   ↓
2. 开发者快速审查（清单检查）
   ↓
3. 运行单元测试（AI 生成或补充）
   ↓
4. 集成测试验证
   ↓
5. 人工深度审查（关键路径）
   ↓
6. 合并代码
```

### 3. 关键评审点详解

#### 业务逻辑验证

```java
// AI 可能忽略业务规则
// 评审要点：这段代码是否正确实现了业务需求？

# 场景：用户注册时，需要验证邮箱格式

// AI 生成的代码（可能有问题）
public void registerUser(UserRequest request) {
    // AI 可能只做了简单的非空检查
    if (request.getEmail() == null) {
        throw new Exception("Email is required");
    }
    userRepository.save(request);
}

// 评审后应该补充
public void registerUser(UserRequest request) {
    // 1. 验证邮箱格式
    if (!isValidEmail(request.getEmail())) {
        throw new ValidationException("Invalid email format");
    }
    // 2. 验证邮箱唯一性
    if (userRepository.existsByEmail(request.getEmail())) {
        throw new BusinessException("Email already exists");
    }
    // 3. 验证密码强度
    if (!isStrongPassword(request.getPassword())) {
        throw new ValidationException("Password too weak");
    }
    userRepository.save(request);
}
```

#### 安全漏洞检查

```java
// AI 常见安全漏洞

# 1. SQL 注入（最常见）
// ❌ 危险 - AI 可能生成这样的代码
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// ✅ 安全 - 使用参数化查询
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);

# 2. 权限校验缺失
// ❌ 危险 - 缺少权限校验
@PostMapping("/admin/users")
public void deleteUser(Long id) { ... }

// ✅ 安全 - 添加权限校验
@PostMapping("/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

# 3. 敏感数据泄露
// ❌ 危险 - 错误信息泄露敏感信息
catch (Exception e) {
    return "Error: " + e.getMessage();  // 可能泄露数据库结构
}

// ✅ 安全 - 通用错误信息
catch (Exception e) {
    log.error("Failed to process request", e);
    return "An error occurred. Please contact support.";
}
```

## AI 辅助测试最佳实践

### 1. 测试金字塔在 AI 时代的应用

```
        /\        少量 E2E 测试（关键用户路径）
       /  \
      /----\      更多 集成测试（API、模块交互）
     /      \
    /--------\   大量 单元测试（每个方法、每个类）
   /          \
  --------------
```

### 2. AI 生成测试的策略

#### 边界条件测试

```java
// AI 应该生成的测试用例

@Test
void createUser_withNullEmail_throwsException() {
    UserRequest request = UserRequest.builder()
        .name("John")
        .email(null)
        .build();

    assertThrows(ValidationException.class,
        () -> userService.createUser(request));
}

@Test
void createUser_withInvalidEmail_throwsException() {
    UserRequest request = UserRequest.builder()
        .name("John")
        .email("invalid-email")
        .build();

    assertThrows(ValidationException.class,
        () -> userService.createUser(request));
}

@Test
void createUser_withEmptyName_throwsException() {
    UserRequest request = UserRequest.builder()
        .name("")
        .email("john@example.com")
        .build();

    assertThrows(ValidationException.class,
        () -> userService.createUser(request));
}
```

#### 业务规则测试

```java
// 业务规则测试

@Test
void order_withDiscount_shouldApplyCorrectly() {
    // 场景：订单满 100 元打 9 折
    Order order = Order.builder()
        .amount(new BigDecimal("150.00"))
        .build();

    BigDecimal finalAmount = discountService.calculateFinalAmount(order);

    assertEquals(new BigDecimal("135.00"), finalAmount);
}

@Test
void order_belowThreshold_shouldNotApplyDiscount() {
    // 场景：订单不满 100 元不打折
    Order order = Order.builder()
        .amount(new BigDecimal("80.00"))
        .build();

    BigDecimal finalAmount = discountService.calculateFinalAmount(order);

    assertEquals(new BigDecimal("80.00"), finalAmount);
}
```

### 3. 测试覆盖策略

| 测试类型 | 覆盖目标 | AI 生成优先级 |
|----------|----------|---------------|
| 正常路径 | 核心业务逻辑 | 高 |
| 边界条件 | 空值、极限值、临界值 | 高 |
| 异常路径 | 异常处理、错误提示 | 中 |
| 并发测试 | 多线程场景 | 低（需人工补充） |
| 性能测试 | 响应时间、资源使用 | 低（需人工补充） |

### 4. 使用 AI 生成测试的提示词

```markdown
# AI 生成测试的提示词示例

"为 UserService.createUser() 方法生成单元测试，
需要覆盖以下场景：
1. 正常创建用户
2. 邮箱为空时抛出 ValidationException
3. 邮箱格式无效时抛出 ValidationException
4. 邮箱已存在时抛出 BusinessException
5. 密码强度不足时抛出 ValidationException
6. 成功创建后发送欢迎邮件"
```

## AI 辅助测试的工具支持

### 1. 测试框架选择

| 框架 | 用途 | AI 友好度 |
|------|------|----------|
| JUnit 5 | 单元测试 | ⭐⭐⭐⭐⭐ |
| Mockito | Mock 对象 | ⭐⭐⭐⭐⭐ |
| Testcontainers | 数据库测试 | ⭐⭐⭐⭐ |
| MockMvc | Controller 测试 | ⭐⭐⭐⭐ |
| RestAssured | API 测试 | ⭐⭐⭐⭐ |

### 2. 测试代码质量标准

```java
// 好的测试 - AI 和人类都容易理解

@Test
void shouldThrowExceptionWhenEmailAlreadyExists() {
    // Arrange
    String existingEmail = "john@example.com";
    when(userRepository.existsByEmail(existingEmail)).thenReturn(true);

    UserRequest request = UserRequest.builder()
        .name("John")
        .email(existingEmail)
        .build();

    // Act & Assert
    BusinessException exception = assertThrows(
        BusinessException.class,
        () -> userService.createUser(request)
    );

    assertEquals("Email already exists", exception.getMessage());
}
```

## 持续改进：AI 与人类的协作闭环

```
1. AI 生成代码
   ↓
2. 开发者审查 + 补充测试
   ↓
3. 运行测试，发现问题
   ↓
4. 反馈给 AI，修正代码
   ↓
5. 重复直到通过
   ↓
6. 合并代码，积累知识
```

### 反馈闭环的关键点

1. **具体反馈**：告诉 AI 哪里错了，而不只是说"不对"
2. **提供上下文**：解释业务背景和约束条件
3. **示范正确做法**：给出正确实现的示例
4. **积累知识库**：将常见问题和建议记录下来

## 总结

| 环节 | 最佳实践 | 关键点 |
|------|----------|--------|
| 代码评审 | 使用审查清单 | 业务逻辑、安全、边界条件 |
| 单元测试 | AI 生成 + 人类补充 | 边界条件、异常路径 |
| 集成测试 | 覆盖关键路径 | 模块交互、数据一致性 |
| 持续改进 | 建立反馈闭环 | AI 学习人类反馈 |

AI vibe coding 并不意味着放弃代码质量，相反，它要求人类开发者更加主动地参与代码审查和测试。通过建立有效的评审和测试机制，我们既能享受 AI 带来的效率提升，又能确保代码质量。

---

## 参考资料

- [JUnit 5 Documentation](https://junit.org/junit5/)
- [Mockito Documentation](https://site.mockito.org/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Spring Boot Testing Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)