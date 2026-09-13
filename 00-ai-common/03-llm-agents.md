# LLM Agent 入门

> 前置知识：了解 [AI 基本概念](./01-ai-basics.md) 和 [提示词工程](./02-prompt-engineering.md)

AI Agent（智能体）是基于大型语言模型（LLM）的智能系统，能够自主感知环境、规划行动、执行任务并从反馈中学习。与简单的问答不同，Agent 具有「行动能力」，能够与外部世界交互，完成复杂任务。本文将介绍 LLM Agent 的核心概念、架构设计和实践方法。

## 1. 什么是 LLM Agent？

LLM Agent 是基于大语言模型的智能系统，它不仅能够理解和生成文本，还能自主决策、调用工具、执行多步骤任务。简单来说，Agent = LLM + 工具 + 记忆 + 规划能力。

### 1.1 Agent 与普通 LLM 的区别

| 特征 | 普通 LLM | LLM Agent |
|------|----------|-----------|
| 交互方式 | 被动响应，一次性回答 | 主动规划，持续交互 |
| 工具使用 | 无法使用外部工具 | 可以调用 API、搜索、计算器等 |
| 记忆能力 | 无状态（除上下文窗口） | 长期记忆，可存储和检索信息 |
| 任务能力 | 单一问答 | 多步骤复杂任务 |
| 自主性 | 低 | 高 |

### 1.2 Agent 的典型应用场景

- **软件开发**：自动编写代码、调试、测试
- **数据分析**：自动抓取数据、分析、可视化
- **智能助手**：日程管理、邮件处理、信息检索
- **自动化工作流**：自动执行多步骤业务流程
- **研究助理**：自动搜集信息、总结报告

## 2. Agent 的核心组件

一个典型的 LLM Agent 由以下核心组件构成：

### 2.1 规划（Planning）

Agent 需要将复杂任务分解为可执行的子任务，并规划执行顺序。

#### 思维链（Chain of Thought）

让 Agent 在回答前展示思考过程，提高推理质量。

```python
# 思维链示例
"用户要求：帮我规划一个从北京到上海的5天旅行

思考过程：
1. 了解用户偏好（预算、兴趣、出行方式）
2. 搜索上海热门景点和美食
3. 根据天数规划行程安排
4. 考虑交通和住宿
5. 生成详细行程单

最终输出：..."
```

#### 任务分解（Task Decomposition）

将复杂任务拆解为多个简单子任务。

```python
# 任务分解示例
"任务：分析某公司财务状况

拆解：
1. 获取公司年报数据
2. 提取关键财务指标（营收、利润、负债等）
3. 计算财务比率
4. 与行业平均水平对比
5. 生成分析报告

Agent 依次执行每个子任务"
```

### 2.2 记忆（Memory）

Agent 需要记住任务执行过程中的关键信息。

#### 记忆类型

| 类型 | 描述 | 实现方式 |
|------|------|----------|
| 短期记忆 | 当前对话上下文 | LLM 上下文窗口 |
| 长期记忆 | 跨会话持久存储 | 向量数据库、文件存储 |
| 工作记忆 | 任务执行中间状态 | 程序变量、状态机 |

#### 记忆检索

```
记忆检索流程：
1. 用户新输入
2. 将输入转换为向量
3. 在向量数据库中检索相似记忆
4. 将相关记忆加入上下文
5. 生成回复
6. 将重要信息存入记忆库
```

### 2.3 工具使用（Tool Use）

Agent 通过调用外部工具扩展能力边界。

#### 常见工具类型

- **搜索工具**：Google、Bing、Wikipedia
- **代码执行**：Python REPL、代码沙箱
- **API 调用**：REST API、数据库查询
- **文件操作**：读取、写入、编辑文件
- **浏览器控制**：自动化网页操作
- **计算工具**：计算器、数学求解器

#### 工具定义示例

```json
// 工具定义（JSON Schema 格式）
{
  "name": "search_web",
  "description": "搜索互联网获取最新信息",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词"
      },
      "max_results": {
        "type": "integer",
        "description": "返回结果数量",
        "default": 5
      }
    },
    "required": ["query"]
  }
}
```

### 2.4 反思（Reflection）

Agent 能够评估自己行动的结果，并从中学习改进。

```python
# 反思机制示例
"执行结果：代码运行失败，错误信息为 'division by zero'

反思：
- 错误原因：除数为0
- 根因：未对输入进行有效性检查
- 改进方向：添加输入验证逻辑
- 后续行动：修改代码添加除零检查

重新执行修改后的代码"
```

## 3. Agent 架构模式

### 3.1 ReAct 模式

ReAct（Reasoning + Acting）结合推理和行动，让 Agent 在思考中行动，在行动中思考。

```python
# ReAct 循环
while not task_complete:
    # 1. 思考（Reason）
    thought = llm.think(context)

    # 2. 行动（Act）
    action = llm.decide_action(thought)
    result = execute(action)

    # 3. 观察（Observe）
    context.add(result)

    # 4. 反思
    if result.is_failure:
        context.add("反思：需要调整策略")
```

### 3.2 Agent Loop 模式

典型的 Agent 执行循环：感知 → 规划 → 行动 → 反馈 → 反思。

```python
# Agent Loop 伪代码
class Agent:
    def run(self, task):
        # 1. 理解任务
        goal = self.parse_task(task)

        # 2. 规划步骤
        plan = self.plan(goal)

        # 3. 循环执行
        while not self.is_complete(plan):
            # 执行当前步骤
            step = plan.current_step()
            result = self.execute(step)

            # 4. 评估结果
            if self.evaluate(result) == "失败":
                # 5. 反思并调整
                plan = self.replan(plan, result)

            # 6. 更新记忆
            self.memory.add(result)

        # 7. 返回结果
        return self.memory.get_final_result()
```

### 3.3 多 Agent 协作

多个 Agent 协同工作，各司其职，完成复杂任务。

```
Agent 团队架构：

┌─────────────────────────────────────┐
│         协调 Agent                  │
│    (任务分发、结果汇总)              │
└───────────┬─────────────┬───────────┘
            │             │
    ┌───────▼──────┐ ┌────▼──────┐
    │  研究 Agent  │ │  编码 Agent │
    │ (信息搜集)   │ │ (代码实现) │
    └───────┬──────┘ └────┬──────┘
            │             │
    ┌───────▼──────┐ ┌────▼──────┐
    │  审查 Agent  │ │  测试 Agent │
    │ (代码审查)   │ │ (测试验证) │
    └─────────────┘ └────────────┘
```

## 4. 主流 Agent 框架

### 4.1 LangChain

最流行的 LLM 应用开发框架，提供完整的 Agent 构建工具。

```python
# LangChain Agent 示例
from langchain.agents import load_tools
from langchain.agents import AgentExecutor, create_openai_functions_agent
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder

# 1. 创建模型
llm = ChatOpenAI(model="gpt-4")

# 2. 加载工具
tools = load_tools(["serpapi", "llm-math"], llm=llm)

# 3. 定义提示词
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手，可以用工具来完成任务"),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("user", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

# 4. 创建 Agent
agent = create_openai_functions_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools)

# 5. 执行
result = executor.invoke({"input": "查找2024年AI最新发展并计算增长率"})
```

### 4.2 LlamaIndex

专注于知识检索和 RAG（检索增强生成）的框架。

```python
# LlamaIndex Agent 示例
from llama_index.agent import OpenAIAgent
from llama_index import VectorStoreIndex

# 1. 创建知识库索引
index = VectorStoreIndex.from_documents(documents)

# 2. 创建 Agent（带知识检索能力）
agent = OpenAIAgent.from_tools(
    tools=[query_engine_tool],
    verbose=True
)

# 3. 询问问题（Agent 会自动检索知识库）
response = agent.chat("关于我们公司的产品政策是什么？")
```

### 4.3 AutoGen

Microsoft 出品的多 Agent 协作框架。

```python
# AutoGen 多 Agent 示例
from autogen import ConversableAgent, GroupChat, GroupChatManager

# 1. 创建多个 Agent
coder = ConversableAgent(
    name="coder",
    system_message="你是一位Python专家，负责编写代码"
)

reviewer = ConversableAgent(
    name="reviewer",
    system_message="你是一位代码审查员，负责检查代码质量"
)

# 2. 创建群组聊天
group_chat = GroupChat(
    agents=[coder, reviewer],
    messages=[],
    max_round=10
)

# 3. 创建管理器
manager = GroupChatManager(groupchat=group_chat)

# 4. 启动对话
coder.initiate_chat(
    manager,
    message="请用Python实现一个快速排序算法，并让reviewer审查"
)
```

### 4.4 框架对比

| 框架 | 特点 | 适用场景 | 学习曲线 |
|------|------|----------|----------|
| LangChain | 组件丰富，生态完善 | 快速构建 LLM 应用 | 中等 |
| LlamaIndex | 专注知识检索，RAG 强 | 文档问答、知识库 | 低 |
| AutoGen | 多 Agent 协作 | 复杂多步骤任务 | 中等 |
| Claude Agent | 内置工具调用，安全可控 | 编程、任务自动化 | 低 |

## 5. 实战：构建一个简单的 Agent

### 5.1 需求定义

构建一个「研究助手」Agent，能够：

1. 接受用户的研究主题
2. 搜索相关资料
3. 总结关键信息
4. 生成报告

### 5.2 架构设计

```
研究助手 Agent 架构：

输入：研究主题
  ↓
  ┌─────────────┐
  │  规划器     │ → 分解任务：搜索→整理→生成
  └─────────────┘
  ↓
  ┌─────────────┐
  │  搜索工具   │ → 调用搜索引擎 API
  └─────────────┘
  ↓
  ┌─────────────┐
  │  总结器     │ → 提取关键信息
  └─────────────┘
  ↓
  ┌─────────────┐
  │  生成器     │ → 生成结构化报告
  └─────────────┘
  ↓
输出：研究报告
```

### 5.3 简化实现

```python
# 简化版研究助手 Agent（Python）
class ResearchAgent:
    def __init__(self, llm, search_tool, max_iterations=5):
        self.llm = llm
        self.search_tool = search_tool
        self.max_iterations = max_iterations
        self.memory = []

    def research(self, topic):
        # 1. 规划任务
        plan = self._create_plan(topic)
        self.memory.append({"role": "user", "content": topic})

        # 2. 循环执行
        for i in range(self.max_iterations):
            # 思考下一步
            thought = self._think(plan)

            # 执行行动
            if thought["action"] == "search":
                result = self.search_tool.run(thought["query"])
            elif thought["action"] == "summarize":
                result = self._summarize()
            elif thought["action"] == "generate":
                return self._generate_report()
            else:
                result = "未知行动"

            # 反思
            self._reflect(thought, result)
            self.memory.append({"thought": thought, "result": result})

        return "研究超时"

    def _create_plan(self, topic):
        "将任务分解为步骤"
        return {
            "topic": topic,
            "steps": ["search", "analyze", "generate"],
            "current": 0
        }

    def _think(self, plan):
        "决定下一步行动"
        prompt = f"根据当前状态决定下一步：{self.memory[-3:]}"
        response = self.llm.chat(prompt)
        return response.json()

    def _reflect(self, thought, result):
        "评估行动结果"
        if "error" in result.lower():
            self.memory.append({"反思": "需要调整搜索策略"})

# 使用示例
agent = ResearchAgent(llm=gpt4, search_tool=serpapi)
report = agent.research("AI Agent 的最新发展趋势")
print(report)
```

## 6. Agent 的挑战与未来

### 6.1 当前挑战

- **可靠性**：Agent 行为不可预测，可能产生错误
- **安全性**：工具调用可能带来安全风险
- **成本**：多轮交互消耗更多 Token
- **调试**：复杂 Agent 难以调试和追踪
- **评估**：缺乏统一的评估标准

### 6.2 未来趋势

- **多模态 Agent**：处理图像、音频、视频等多种输入
- **自主学习**：Agent 从交互中持续学习和改进
- **多 Agent 协作**：更复杂的 Agent 团队协作
- **垂直领域 Agent**：医疗、法律、金融等专业领域
- **Agent 编排平台**：低代码/无代码 Agent 构建工具

## 7. 总结

LLM Agent 是 AI 应用的重要方向，它将 LLM 的语言理解能力与工具调用、规划执行能力结合，能够完成复杂的实际任务。

### 核心组件

| 组件 | 功能 | 实现方式 |
|------|------|----------|
| 规划 | 任务分解、行动决策 | CoT、ToT、LLM 推理 |
| 记忆 | 存储和检索信息 | 向量数据库、上下文窗口 |
| 工具 | 扩展 Agent 能力 | API 调用、代码执行 |
| 反思 | 评估结果、改进策略 | 错误处理、重新规划 |

---

## 参考资料

- [LangChain Agents 文档](https://python.langchain.com/docs/modules/agents/)
- [LlamaIndex Agents](https://docs.llamaindex.ai/en/latest/module_guides/agents/)
- [AutoGen 文档](https://microsoft.github.io/autogen/)
- [ReAct 论文](https://arxiv.org/abs/2210.03629)
- [Agent 提示词技巧](https://www.promptingguide.ai/techniques/agents)

---

*&copy; 2024 AI DevOps 研究 | [返回目录](./index.md) | [返回首页](../index.html)*