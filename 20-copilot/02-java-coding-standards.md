# AI Vibe Coding 的 Java 代码设计规范

> 前置知识：建议先阅读 [AI Vibe Coding 项目目录结构最佳实践](01-project-structure.md)

在 AI 辅助编程场景下，代码设计规范不仅影响代码质量，更直接影响 AI 理解代码、生成代码的效率。良好的代码设计规范让 AI 能够准确把握代码意图，生成更准确、更安全的代码。

## 核心原则：AI 友好的代码设计

### 1. 单一职责原则（SRP）

每个类和方法应该只有一个职责。这对 AI 协作尤为重要：

```java
// 不推荐 - 职责过多，AI 难以理解
public class UserManager {
    public User createUser(Map<String, Object> data) { ... }
    public void sendEmail(User user) { ... }
    public void generateReport() { ... }
    public void cacheUser(User user) { ... }
}

// 推荐 - 单一职责，AI 容易理解
public class UserService { public User createUser(...) { ... } }
public class EmailService { public void sendEmail(...) { ... } }
public class ReportGenerator { public void generateReport() { ... } }
public class UserCache { public void cacheUser(...) { ... } }
```

### 2. 明确的命名规范

命名是 AI 理解代码的第一入口。遵循以下规范：

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase，名词 | `UserService`, `OrderController` |
| 方法名 | camelCase，动词或动词短语 | `createUser`, `findById` |
| 变量名 | camelCase，名词 | `userList`, `totalCount` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| 布尔值 | is/has/can/should 前缀 | `isActive`, `hasPermission` |

### 3. 方法长度控制

方法应该简短（建议不超过 30 行），让 AI 能够完整理解整个方法逻辑：

```java
// 不推荐 - 方法过长
public User createUserAndSendEmail(UserRequest request) {
    // 验证参数
    if (request == null) { throw new IllegalArgumentException(); }
    if (request.getName() == null) { throw new IllegalArgumentException(); }
    // 验证邮箱格式
    if (!request.getEmail().contains("@")) { throw new IllegalArgumentException(); }
    // 创建用户
    User user = new User();
    user.setName(request.getName());
    user.setEmail(request.getEmail());
    user.setCreatedAt(new Date());
    // 保存到数据库
    userRepository.save(user);
    // 发送欢迎邮件
    emailService.sendWelcomeEmail(user);
    // 记录日志
    log.info("User created: " + user.getId());
    // 更新缓存
    cacheService.put("user:" + user.getId(), user);
    return user;
}

// 推荐 - 方法职责单一
public User createUser(UserRequest request) {
    validateRequest(request);
    User user = buildUser(request);
    userRepository.save(user);
    onUserCreated(user);
    return user;
}

private void validateRequest(UserRequest request) { ... }
private User buildUser(UserRequest request) { ... }
private void onUserCreated(User user) { ... }
```

## 面向 AI 的代码规范

### 1. 显式类型声明

避免使用 `var` 让 AI 能够快速推断变量类型：

```java
// 不推荐 - AI 需要额外推断类型
var user = userService.findById(id);
var result = orderList.stream().map(...).collect(...);

// 推荐 - 类型明确
User user = userService.findById(id);
List<OrderDTO> result = orderList.stream()
    .map(OrderDTO::from)
    .collect(Collectors.toList());
```

### 2. 异常处理规范

明确声明受检异常，让 AI 清楚知道需要处理的异常：

```java
// 不推荐 - 隐藏异常信息
public User findById(Long id) {
    try {
        return userRepository.findById(id).orElse(null);
    } catch (Exception e) {
        return null;  // 静默吞掉异常
    }
}

// 推荐 - 明确异常处理
public User findById(Long id) throws UserNotFoundException {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

### 3. 链式调用与可读性

在长链式调用中添加适当换行，提高可读性：

```java
// 不推荐 - 难以阅读
List<UserDTO> users = userRepository.findByStatus("ACTIVE").stream().filter(u -> u.getAge() > 18).map(u -> new UserDTO(u.getName(), u.getEmail())).collect(Collectors.toList());

// 推荐 - 清晰可读
List<UserDTO> users = userRepository.findByStatus("ACTIVE")
    .stream()
    .filter(u -> u.getAge() > 18)
    .map(UserDTO::from)
    .collect(Collectors.toList());
```

## 分层架构中的代码规范

### Controller 层规范

```java
// Controller 应该简洁，只负责请求分发
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    // 构造函数注入（推荐）
    public UserController(UserService userService) {
        this.userService = userService;
    }

    // 单一职责：只处理 HTTP 相关逻辑
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        User user = userService.createUser(request);
        return ResponseEntity.ok(UserResponse.from(user));
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(UserResponse.from(user));
    }
}
```

### Service 层规范

```java
// Service 层承载业务逻辑
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final CacheService cacheService;

    public UserService(
            UserRepository userRepository,
            EmailService emailService,
            CacheService cacheService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.cacheService = cacheService;
    }

    // 业务方法应该清晰表达业务意图
    public User createUser(CreateUserRequest request) {
        validateUniqueEmail(request.getEmail());
        User user = buildUserEntity(request);
        User saved = userRepository.save(user);
        emailService.sendWelcomeEmail(saved);
        return saved;
    }

    private void validateUniqueEmail(String email) {
        if (userRepository.existsByEmail(email)) {
            throw new BusinessException("Email already exists");
        }
    }

    private User buildUserEntity(CreateUserRequest request) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        user.setStatus(UserStatus.ACTIVE);
        return user;
    }
}
```

### Repository 层规范

```java
// Repository 层专注于数据访问
public interface UserRepository extends JpaRepository<User, Long> {

    // 清晰的方法命名，让 AI 理解查询意图
    Optional<User> findByEmail(String email);

    List<User> findByStatus(UserStatus status);

    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    // 复杂查询使用 @Query
    @Query("SELECT u FROM User u WHERE u.status = :status AND u.age >= :minAge")
    List<User> findActiveUsersByMinAge(
        @Param("status") UserStatus status,
        @Param("minAge") Integer minAge);
}
```

## AI 辅助下的代码规范检查

### 1. 使用 Builder 模式

Builder 模式让 AI 能够更清晰地构造对象：

```java
// 使用 Lombok @Builder
@Builder
public class UserDTO {
    private final Long id;
    private final String name;
    private final String email;
    private final LocalDateTime createdAt;
}

// AI 容易理解如何构造
UserDTO dto = UserDTO.builder()
    .id(1L)
    .name("John")
    .email("john@example.com")
    .createdAt(LocalDateTime.now())
    .build();
```

### 2. 使用 Lombok 减少样板代码

减少样板代码让 AI 聚焦于业务逻辑：

```java
// 推荐 - 使用 Lombok 简化
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private String email;
}

// 不推荐 - 大量样板代码分散 AI 注意力
public class User {
    private Long id;
    private String name;
    private String email;

    // 30+ 行 getter/setter/equals/hashCode/toString
}
```

### 3. 使用 record（Java 16+）

对于不可变数据对象，使用 record 简化代码：

```java
// 推荐 - 使用 record 定义 DTO
public record UserResponse(
    Long id,
    String name,
    String email
) {}

// 传统方式需要更多代码
@Data
@Builder
public class UserResponse {
    private Long id;
    private String name;
    private String email;
}
```

## 总结

| 规范类别 | 最佳实践 | AI 收益 |
|----------|----------|---------|
| 单一职责 | 类和方法的职责应该单一 | 快速理解代码意图 |
| 命名规范 | 使用语义化的命名 | 准确推断类型和用途 |
| 方法长度 | 方法不超过 30 行 | 完整理解方法逻辑 |
| 类型声明 | 使用显式类型而非 var | 直接获取类型信息 |
| 异常处理 | 明确声明和抛出异常 | 了解错误处理路径 |
| 代码简化 | 使用 Lombok、record | 聚焦业务逻辑 |

良好的代码设计规范是 AI vibe coding 的核心。遵循这些规范不仅能提高代码质量，更能让 AI 成为你真正的编程助手，显著提升开发效率。

---

## 参考资料

- [Spring Boot Official Documentation](https://spring.io/projects/spring-boot)
- [Java 16+ Features](https://www.oracle.com/java/technologies/javase/16-downloads.html)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)