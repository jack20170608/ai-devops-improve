# AI Vibe Coding 项目目录结构最佳实践

> 前置知识：建议先阅读 [AI Prompt Engineering 入门](../00-AI-common/02-prompt-engineering.md)

在 AI 辅助编程时代，项目目录结构不仅仅是代码的组织方式，更是与 AI 协作的基础。一个良好的目录结构能让 AI 更快理解代码意图、准确定位问题、生成更准确的代码。

## 为什么目录结构对 AI 协作至关重要

与传统编程不同，AI vibe coding 依赖于 AI 对整个项目的理解能力。目录结构直接影响 AI 的上下文理解：

- **上下文理解**：清晰的目录结构帮助 AI 快速定位相关代码
- **意图传达**：目录命名本身就是一种隐式的文档
- **依赖推断**：合理的模块划分帮助 AI 理解代码依赖关系
- **代码生成**：AI 可以根据目录结构推断应该在哪里添加新代码

## 推荐的 Java 项目目录结构

以下是一个面向 AI 协作优化的 Java 项目目录结构：

```
project-root/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           ├── config/          # 配置层 - 所有配置类
│   │   │           ├── controller/      # 控制器层 - HTTP 请求处理
│   │   │           ├── service/         # 服务层 - 业务逻辑
│   │   │           ├── repository/      # 持久层 - 数据访问
│   │   │           ├── model/           # 模型层 - 数据模型
│   │   │           ├── dto/             # 数据传输对象
│   │   │           ├── vo/              # 视图对象
│   │   │           ├── entity/          # 实体类
│   │   │           ├── exception/       # 异常定义
│   │   │           ├── util/            # 工具类
│   │   │           └── constant/        # 常量定义
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── mapper/                  # MyBatis 映射文件
│   │       └── static/
│   └── test/
│       ├── java/
│       │   └── com/
│       │       └── example/
│       │           ├── controller/      # 控制器测试
│       │           ├── service/         # 服务层测试
│       │           ├── repository/      # 持久层测试
│       │           └── integration/     # 集成测试
│       └── resources/
├── docs/                                # 项目文档
├── scripts/                             # 脚本文件
├── README.md
└── pom.xml
```

## 目录设计核心原则

### 1. 分层清晰（Layered Architecture）

遵循标准的分层架构，让 AI 能够快速理解代码的职责归属：

- `controller/` - 处理 HTTP 请求，返回响应
- `service/` - 承载业务逻辑
- `repository/` - 负责数据访问
- `model/` - 数据模型定义

### 2. 单一职责（Single Responsibility）

每个包（package）应该有明确的职责：

| 包名 | 职责 | 包含内容 |
|------|------|----------|
| `config/` | 配置管理 | 配置类、Bean 定义 |
| `exception/` | 异常处理 | 自定义异常、异常处理类 |
| `dto/` | 数据传输 | 请求/响应对象 |
| `util/` | 工具函数 | 通用工具方法 |

### 3. 语义化命名（Semantic Naming）

目录和文件的命名应该能够传达其用途：

```bash
# 好的命名 - AI 能快速理解
UserController.java
UserService.java
UserRepository.java
OrderPaymentService.java

# 不好的命名 - AI 需要额外推断
UserCtrl.java
UserBo.java
UserDao.java
Ops.java
```

## AI 友好的目录结构特征

### 1. 扁平与深层的平衡

避免过深的目录嵌套（超过 4 层），但也避免将所有文件放在根目录：

```bash
# 不推荐 - 嵌套过深
src/main/java/com/example/module/submodule/feature/detail/handler/ActionHandler.java

# 推荐 - 扁平但有组织
src/main/java/com/example/module/feature/ActionHandler.java
```

### 2. 明确的边界

每个模块应该有清晰的边界，模块间通过接口通信：

```
com.example/
├── user/                   # 用户模块 - 独立边界
│   ├── UserController.java
│   ├── UserService.java
│   ├── UserRepository.java
│   └── User.java
├── order/                  # 订单模块 - 独立边界
│   ├── OrderController.java
│   ├── OrderService.java
│   └── Order.java
└── common/                 # 公共模块
    ├── config/
    ├── exception/
    └── util/
```

### 3. 测试目录与源码对应

测试代码应该与源码保持一一对应的关系：

| 源码位置 | 测试位置 |
|----------|----------|
| `service/UserService.java` | `test/service/UserServiceTest.java` |
| `controller/OrderController.java` | `test/controller/OrderControllerTest.java` |
| `repository/ProductRepository.java` | `test/repository/ProductRepositoryTest.java` |

## 特殊目录设计

### 1. 文档目录（docs/）

为 AI 辅助文档创建专门目录：

```
docs/
├── architecture.md         # 架构设计文档
├── api.md                  # API 接口文档
├── database.md             # 数据库设计
└── ai-prompt/              # AI 提示词模板
    ├── new-feature.md
    └── bug-fix.md
```

### 2. 脚本目录（scripts/）

存放运维和开发脚本：

```
scripts/
├── db/
│   ├── init.sql
│   └── migration.sh
├── deploy/
│   └── deploy.sh
└── dev/
    └── setup.sh
```

## 总结

| 原则 | 实践 |
|------|------|
| 分层清晰 | controller → service → repository 层级分明 |
| 单一职责 | 每个包有明确职责，避免职责混乱 |
| 语义化命名 | 命名要能传达用途，避免缩写 |
| 扁平结构 | 目录层级不超过 4 层 |
| 模块边界 | 按业务功能划分模块，清晰边界 |
| 测试对应 | 测试代码与源码一一对应 |

良好的目录结构是 AI vibe coding 的基础。它不仅帮助人类开发者理解代码，更能让 AI 快速把握项目全貌，生成更准确、更有针对性的代码。

---

## 参考资料

- [Spring Boot Official Documentation](https://spring.io/projects/spring-boot)
- [Java Style Guide - Google](https://google.github.io/styleguide/javaguide.html)
