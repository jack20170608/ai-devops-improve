# AI 辅助开发项目结构规范

> 目标：让开发者与 GitHub Copilot 都能快速理解项目，并在明确的范围、权限和验证规则内完成变更。

## 1. “Vibe Coding”在企业中的正确定位

Vibe Coding 通常指开发者用自然语言快速表达意图，由 AI 生成、修改和验证代码。它适合原型、小型变更和探索，但企业项目不能以“看起来能运行”作为完成标准。

企业级 AI 辅助开发必须从：

```text
Prompt → 生成代码 → 人凭感觉接受
```

升级为：

```text
Issue 与验收标准
  → Agent 读取仓库事实
  → 研究和计划
  → 小范围实现
  → 自动测试与安全扫描
  → 独立人工评审
  → 受控合并和发布
```

### 1.1 强制原则

- **MUST**：AI 生成代码与人工代码使用相同质量和安全门禁。
- **MUST**：每项变更有明确目标、范围、非目标和验收标准。
- **MUST**：安全、权限、合并和部署由确定性策略控制，不能只靠 Prompt。
- **MUST**：提交者对接受的 AI 输出负责。
- **SHOULD**：大型任务拆成研究、计划、实现、验证四个阶段。
- **SHOULD**：从单仓库、小范围、可自动验证、容易回滚的任务开始。
- **MUST NOT**：让 Agent 在没有人工审批的情况下合并、部署生产、修改权限或删除生产数据。

## 2. 推荐的 Java 项目目录

```text
project/
├── README.md
├── CONTRIBUTING.md
├── SECURITY.md
├── LICENSE
├── CHANGELOG.md
├── AGENTS.md                         # 可选：跨 Agent 的通用规则
├── pom.xml                           # 或 settings.gradle/build.gradle
├── mvnw
├── mvnw.cmd
├── .mvn/
│   └── wrapper/
├── .gitignore
├── .gitattributes
├── .editorconfig
├── .env.example                      # 只能有占位符，不得有真实凭据
│
├── docs/
│   ├── architecture/
│   │   ├── README.md                 # 架构导航
│   │   ├── context.md                # 系统上下文
│   │   ├── containers.md             # 应用和数据存储
│   │   ├── components.md             # 关键模块
│   │   ├── deployment.md
│   │   └── data-flows.md
│   ├── domain/
│   │   ├── glossary.md               # 领域词汇
│   │   ├── bounded-contexts.md
│   │   ├── invariants.md             # 不可破坏的业务规则
│   │   └── examples.md
│   ├── adr/
│   │   ├── README.md
│   │   ├── template.md
│   │   └── 0001-select-database.md
│   ├── api/
│   │   └── openapi.yaml
│   └── runbooks/
│       ├── local-development.md
│       ├── deployment.md
│       └── incident-response.md
│
├── src/
│   ├── main/
│   │   ├── java/com/example/product/
│   │   │   ├── orders/
│   │   │   │   ├── domain/
│   │   │   │   ├── application/
│   │   │   │   ├── api/
│   │   │   │   └── persistence/
│   │   │   └── shared/               # 仅放真正共享且稳定的能力
│   │   └── resources/
│   │       ├── application.yml       # 无真实凭据
│   │       └── db/migration/
│   ├── test/
│   │   ├── java/com/example/product/
│   │   └── resources/
│   └── generated/                    # 仅在必须提交生成结果时使用
│
├── .github/
│   ├── CODEOWNERS
│   ├── copilot-instructions.md
│   ├── dependabot.yml
│   ├── pull_request_template.md
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug.yml
│   │   ├── feature.yml
│   │   ├── agent-task.yml
│   │   └── config.yml
│   ├── instructions/
│   │   ├── java.instructions.md
│   │   ├── tests.instructions.md
│   │   ├── workflows.instructions.md
│   │   └── migrations.instructions.md
│   ├── prompts/                      # 按客户端支持情况选择
│   │   ├── implement-issue.prompt.md
│   │   ├── write-tests.prompt.md
│   │   └── security-review.prompt.md
│   ├── agents/
│   │   ├── test-specialist.agent.md
│   │   └── architecture-reviewer.agent.md
│   ├── skills/
│   │   ├── database-migration/
│   │   │   ├── SKILL.md
│   │   │   └── validate.sh
│   │   └── release-check/
│   │       └── SKILL.md
│   └── workflows/
│       ├── ci.yml
│       ├── codeql.yml
│       ├── dependency-review.yml
│       └── copilot-setup-steps.yml
│
└── .vscode/
    └── mcp.json                      # 可选；不得包含真实凭据
```

这是通用模板，不要求所有项目一次性创建所有文件。**MUST NOT** 创建没有内容或无人维护的空目录来制造“完整架构”的假象。

## 3. 根目录入口

### 3.1 README.md

`README.md` **MUST** 回答：

- 项目解决什么问题；
- 当前状态和适用范围；
- 使用的 JDK、框架和构建工具；
- 最短可运行路径；
- 唯一正确的构建、测试和验证命令；
- 主要模块和文档入口；
- 如何报告问题和安全漏洞。

**MUST NOT** 写不存在的目录、依赖和命令。Copilot 会把这些错误信息当作项目事实。

### 3.2 CONTRIBUTING.md

`CONTRIBUTING.md` **MUST** 定义：

- 开发环境和 Wrapper 用法；
- 分支、提交和 PR 规则；
- 必须运行的检查；
- 数据库/API 兼容性要求；
- 生成代码和迁移文件规则；
- 哪些变更需要架构或安全评审。

### 3.3 SECURITY.md

`SECURITY.md` **MUST** 包含私密漏洞报告渠道、支持版本、响应预期和禁止公开披露的敏感信息类型。

## 4. 架构与领域文档

- **SHOULD**：用 `docs/architecture/README.md` 作为架构导航，不把所有内容堆进一张巨大图。
- **SHOULD**：记录系统上下文、模块依赖、部署和关键数据流。
- **SHOULD**：在 `docs/domain/glossary.md` 中统一业务术语。
- **MUST**：关键业务不变量应能链接到实现和测试。
- **SHOULD**：重要决策使用 ADR，记录背景、选择、替代方案和后果。
- **MUST NOT**：在多个文档中复制同一规则；应链接到权威来源。

这些内容能显著减少 Agent 在多个文件中猜测模块职责和业务含义。

## 5. Copilot Instructions

### 5.1 仓库级 Instructions

`.github/copilot-instructions.md` **SHOULD** 简短、自包含，并适用于大多数任务：

```markdown
# Repository overview
- Java 25 LTS, Spring Boot, Maven multi-module project.
- Architecture entry: docs/architecture/README.md.
- Domain terminology: docs/domain/glossary.md.
- Domain code must not depend on persistence or HTTP frameworks.

# Commands
- Use ./mvnw, never a globally installed Maven.
- Full validation: ./mvnw verify.
- Format: ./mvnw spotless:apply.
- Regenerate API sources: ./mvnw generate-sources.

# Change rules
- Do not hand-edit target/ or generated sources.
- Add or update tests for behavior changes.
- Never add credentials, customer data, or production values.
- Do not weaken tests or security checks to make CI pass.
- Report commands actually run and checks not run.
```

要求：

- **MUST**：所有命令已经在干净环境验证。
- **MUST NOT**：包含 token、密码或内部敏感数据。
- **MUST NOT**：与构建配置、CI 或 CONTRIBUTING 冲突。
- **SHOULD**：控制在约两页以内。
- **SHOULD**：详细低频流程放到普通文档或 Skill。
- **MUST**：由 CODEOWNER 评审变更。

### 5.2 路径级 Instructions

仅特定路径适用的规则应放在 `.github/instructions/*.instructions.md`：

```markdown
---
applyTo: "src/main/java/**/domain/**/*.java"
---

- Domain classes must not import persistence, HTTP, or framework packages.
- Preserve aggregate invariants in constructors and behavior methods.
- Add tests for every changed invariant.
```

```markdown
---
applyTo: ".github/workflows/**/*.yml"
---

- Declare explicit minimum permissions.
- Pin third-party actions to reviewed full commit SHAs.
- Never interpolate untrusted event fields directly into shell scripts.
```

**MUST NOT** 在仓库级 Instructions、`AGENTS.md` 和路径级 Instructions 中复制互相可能漂移的长规则。

## 6. Prompt、Skill、Agent 与 MCP

| 制品 | 适用场景 | 关键治理 |
|------|----------|----------|
| Prompt file | 人主动调用的单次任务模板 | 不得替代强制安全规则 |
| Skill | 复杂、可重复、按需加载的 SOP | 审查脚本、来源和工具权限 |
| Custom agent | 工具和职责明确的专门角色 | 显式最小工具集和停止条件 |
| MCP | 连接 GitHub、监控、文档等外部系统 | Server 审批、工具 allowlist、最小凭据 |

### 6.1 Skill

- **MUST**：第三方 Skill 安装前审查 `SKILL.md`、脚本和引用文件。
- **MUST**：外部 Skill 固定到已评审的 tag，优先固定 commit SHA。
- **MUST NOT**：未经完整审查就预批准 `shell` 或 `bash`。
- **SHOULD**：内部维护经过审批的 Skill 清单。

### 6.2 Custom agent

- **MUST**：显式配置工具；省略工具列表可能获得过宽能力。
- **SHOULD**：只读评审 Agent 不得拥有编辑或 shell 权限。
- **SHOULD**：测试 Agent 明确是否允许修改生产代码。
- **MUST**：定义输出、完成和停止条件，而不是只定义“人格”。

### 6.3 MCP

- **MUST**：仅使用组织批准并完成数据流评估的 MCP Server。
- **MUST NOT**：在 MCP JSON 中提交 PAT、API key、密码或 OAuth token。
- **MUST**：只开放任务需要的 server 和 toolset，默认只读。
- **MUST**：部署、删除、权限变更和生产写操作保留外部审批。
- **SHOULD**：使用 OAuth、短期凭据和最小 scope。
- **MUST**：MCP 配置变更由安全 CODEOWNER 评审。

## 7. Issue 与 PR 是 Agent 的工作契约

### 7.1 Agent 任务 Issue

```markdown
## Goal
修复 checkout 在优惠券过期时返回 500 的问题。

## Evidence
- Sentry issue: ...
- Failing request: ...

## Scope
- Allowed: checkout module
- Forbidden: workflow, dependency, public API changes

## Acceptance criteria
- Regression test fails before and passes after the fix
- ./mvnw verify passes
- Existing API response schema remains compatible

## Stop conditions
- Cannot reproduce
- Requires cross-repository change
- Requires production write access
```

### 7.2 PR 模板

PR **SHOULD** 要求：

- 变更目标和关联 Issue；
- 修改范围；
- 实际运行的命令和结果；
- 未运行检查及原因；
- API、数据库、配置和安全影响；
- 风险与回滚方法；
- 生成代码和文档是否同步。

AI 使用声明可以记录，但不能转移提交者责任。

## 8. 测试、CI 与所有权

- **MUST**：提交 Wrapper，并提供单一完整验证入口，如 `./mvnw verify`。
- **MUST**：CI 从干净环境执行格式、编译、单元测试、集成测试和静态分析。
- **MUST**：默认分支和发布分支要求 PR、required checks 和独立人工审批。
- **MUST**：工作流、认证授权、密钥、发布和 Copilot 配置路径由 CODEOWNERS 保护。
- **MUST**：禁止自我批准，限制 bypass，禁止 force push 和分支删除。
- **SHOULD**：Issue、PR 和验证日志形成需求到发布的证据链。

## 9. 生成代码

- **MUST**：明确生成代码的单一源，例如 OpenAPI 或 schema。
- **MUST**：记录生成命令和工具版本。
- **MUST**：生成输出与手写代码分目录。
- **MUST NOT**：手工修改标记为 generated 的文件。
- **MUST**：CI 重新生成并检查差异，防止源与输出漂移。
- **SHOULD**：只有消费者无法在构建时生成或审查需求要求时，才提交生成输出。

## 10. 反模式

- 创建大量空目录和模板，但没有真实项目内容；
- README 中保留不存在的命令；
- 把所有代码放进 `controller/service/repository/util` 全局大包；
- 把整份编码规范复制到 Copilot Instructions；
- 用 Prompt 要求 Agent “遵循最佳实践”，却没有测试和 CI；
- 给 Agent、Skill 或 MCP 默认开放全部工具；
- 在 `.env.example`、测试或文档中使用真实凭据；
- 为通过 CI 让 Agent 删除测试或降低安全规则；
- 让 Copilot Review 成为唯一审批者；
- 让生成代码和手写代码混在同一目录。

## 11. 检查清单

新开发者或 Agent 应能在不猜测的情况下回答：

- [ ] 项目解决什么问题？
- [ ] 领域词和模块职责在哪里定义？
- [ ] 为什么采用当前架构？
- [ ] 使用哪个 JDK 和构建工具版本？
- [ ] 唯一正确的完整验证命令是什么？
- [ ] 哪些目录可修改，哪些是生成代码？
- [ ] 行为变更需要哪些测试和文档？
- [ ] 哪些路径需要安全或架构负责人审批？
- [ ] 凭据应该存在哪里，绝不能存在哪里？
- [ ] Agent、Skill 和 MCP 可以使用哪些工具？
- [ ] 什么证据才表示任务完成？

## 12. 主要依据

- [GitHub：Repository custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions)
- [GitHub：Custom instructions support](https://docs.github.com/en/copilot/reference/custom-instructions-support)
- [GitHub：About agent skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
- [GitHub：Create custom agents](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/create-custom-agents)
- [GitHub：About MCP](https://docs.github.com/en/copilot/concepts/context/mcp)
- [GitHub：Issue and PR templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/about-issue-and-pull-request-templates)
- [GitHub：About CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [Maven standard directory layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)

