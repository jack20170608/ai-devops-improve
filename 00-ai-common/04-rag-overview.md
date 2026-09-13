# RAG 技术概述

> 前置知识：了解 [AI 基本概念](./01-ai-basics.md) 和 [提示词工程](./02-prompt-engineering.md)，熟悉 LLM 的基本原理

检索增强生成（Retrieval-Augmented Generation，RAG）是一种将外部知识检索与大语言模型生成相结合的技术。它能够解决 LLM 知识时效性差、幻觉严重、缺乏专业领域知识等问题，是当前构建企业级 AI 应用的核心技术之一。

## 1. 为什么需要 RAG？

虽然大语言模型能力强大，但存在固有的局限性：

### 1.1 LLM 的局限性

| 问题 | 描述 | RAG 解决方案 |
|------|------|--------------|
| 知识时效性 | 训练数据有截止日期，无法回答最新信息 | 实时检索最新数据 |
| 幻觉问题 | 模型可能生成看似合理但错误的内容 | 基于真实文档生成，有据可查 |
| 领域知识 | 缺乏特定行业的专业知识 | 注入专业文档作为知识源 |
| 长上下文 | 处理长文本成本高，有上下文限制 | 只检索相关内容，节省 Token |
| 数据隐私 | 敏感数据不便用于训练 | 本地知识库，不训练模型 |
| 可解释性 | 难以解释回答的依据 | 引用来源，可追溯可验证 |

### 1.2 RAG 的核心价值

```
# RAG vs 纯 LLM 对比

# 纯 LLM 回答
用户：关于公司2024年Q3的财务表现？
LLM：抱歉，我的训练数据截止到2023年，无法提供最新信息。

# RAG 回答
用户：关于公司2024年Q3的财务表现？
RAG系统：
  1. 检索到2024年Q3财报PDF
  2. 提取关键财务数据
  3. 生成基于真实数据的回答
  4. 附上数据来源

回答：据2024年Q3财报显示，公司营收同比增长15%，
    净利润达到2.3亿元。详细数据见附件财务报告。
    📎 来源：/docs/2024-q3-report.pdf
```

## 2. RAG 工作原理

### 2.1 RAG 架构概览

```
# RAG 完整流程

用户问题 ──────────────────────────────────────────┐
    │                                                 │
    ▼                                                 │
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   理解问题   │ ──▶ │   检索知识   │ ──▶ │   生成回答   │
│  (Query     │     │  (Retrieve) │     │  (Generate) │
│   Under-    │     │             │     │             │
│   standing) │     └─────────────┘     └─────────────┘
└─────────────┘           │                   │
                          ▼                   ▼
                   ┌─────────────┐     ┌─────────────┐
                   │  知识库      │     │  上下文增强  │
                   │ (Knowledge  │     │ (Context    │
                   │   Base)     │     │  Augment)   │
                   └─────────────┘     └─────────────┘
                                              │
                                              ▼
                                       ┌─────────────┐
                                       │  带引用答案  │
                                       │ (Grounded   │
                                       │  Response)  │
                                       └─────────────┘
```

### 2.2 核心流程详解

#### 第一步：文档处理（Indexing）

```
原始文档 ──▶ 文本切分 ──▶ 向量化 ──▶ 存储索引
   │           │           │          │
   ▼           ▼           ▼          ▼
PDF/Word    段落/句子   Embedding  向量数据库
  /HTML      划分        模型       (Milvus/
                          │       Pinecone/
                          ▼       Chroma...)
                      向量表示
```

#### 第二步：检索（Retrieval）

```
用户问题 ──▶ 向量化 ──▶ 向量相似度搜索 ──▶ 获取Top-K相关文档
   │           │              │                   │
   ▼           ▼              ▼                   ▼
"公司Q3    embedding      余弦相似度          排序后的
 营收？"    模型           计算               相关文档
```

#### 第三步：增强（Augmentation）

```python
# 上下文增强

系统提示词 + 检索到的知识 + 用户问题 ──▶ 组合成完整 prompt

# 增强后的 prompt 示例
你是一个专业的财务分析师。请根据以下参考资料回答用户问题。

参考资料：
[1] 公司2024年Q3财报 - 营收数据
[2] 2024年Q3业绩说明会纪要

用户问题：2024年Q3公司的财务表现如何？

请基于上述参考资料回答，并注明来源。
```

#### 第四步：生成（Generation）

```
增强后的Prompt ──▶ LLM生成 ──▶ 带引用的回答
      │                      │
      ▼                      ▼
  完整上下文            结构化输出
                      (答案+来源)
```

## 3. RAG 核心技术组件

### 3.1 文本分割（Text Chunking）

将长文档分割成适合检索的小块，是 RAG 效果的关键环节。

#### 分割策略

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| 固定大小 | 按字符数或 Token 数固定分割 | 通用场景，简单高效 |
| 按段落 | 按自然段落分割 | 结构化文档 |
| 递归分割 | 按层级结构（章节→段落→句子）递归分割 | 复杂文档 |
| 语义分割 | 基于语义相似度智能分割 | 需要保持语义完整 |
| 特殊分割 | 按代码函数、Markdown 标题等分割 | 技术文档 |

#### 分割示例

```python
# 固定大小分割（Python）
from langchain.text_splitter import CharacterTextSplitter

splitter = CharacterTextSplitter(
    chunk_size=500,       # 块大小
    chunk_overlap=50,     # 块间重叠
    separator="\n"        # 分隔符
)

chunks = splitter.split_text(long_text)

# 递归分割（更智能）
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", " ", ""]
)
chunks = splitter.split_text(long_text)
```

### 3.2 向量化（Embedding）

将文本转换为向量表示，是语义检索的基础。

#### 常用 Embedding 模型

| 模型 | 特点 | 维度 | 适用场景 |
|------|------|------|----------|
| text-embedding-ada-002 | OpenAI 出品，效果好 | 1536 | 通用场景 |
| text-embedding-3-small | 性价比高，速度快 | 1536 | 大规模应用 |
| text-embedding-3-large | 效果最佳 | 3072 | 高精度需求 |
| text-multilingual-... | 多语言支持 | 768/1536 | 多语言场景 |
| BGE 系列 | 开源，中文效果好 | 1024 | 本地部署 |

#### Embedding 示例

```python
# 使用 OpenAI Embedding
from openai import OpenAI

client = OpenAI()

# 获取文本向量
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="RAG 是检索增强生成技术"
)

embedding = response.data[0].embedding
print(f"向量维度: {len(embedding)}")  # 1536
```

### 3.3 向量数据库（Vector Database）

存储和检索向量数据的 specialized 数据库。

#### 主流向量数据库对比

| 数据库 | 特点 | 适用场景 | 部署方式 |
|--------|------|----------|----------|
| Pinecone | 托管服务，易用性强 | 快速上线，无需运维 | 云服务 |
| Weaviate | 开源，功能丰富 | 需要本地部署 | 开源/云 |
| Milvus | 高性能，大规模 | 海量数据场景 | 开源/云 |
| Chroma | 轻量级，易集成 | 原型开发 | 开源/本地 |
| Qdrant | Rust 实现，性能高 | 生产级应用 | 开源/云 |
| Faiss | Facebook 出品，算法丰富 | 离线分析 | 开源/本地 |

#### 向量检索示例

```python
# 使用 Chroma 进行向量检索
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

# 创建向量存储
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings()
)

# 相似度检索
results = vectorstore.similarity_search(
    query="公司Q3营收如何？",
    k=3  # 返回Top-3相关文档
)

for i, doc in enumerate(results):
    print(f"【文档{i+1}】{doc.page_content[:100]}...")
```

### 3.4 检索策略

#### 基础检索

- **相似度检索**：基于向量相似度（余弦相似度、点积等）
- **关键词检索**：BM25、TF-IDF 等传统方法
- **混合检索**：结合向量检索和关键词检索

#### 高级检索策略

| 策略 | 描述 | 效果 |
|------|------|------|
| HyDE | 让 LLM 生成假设性回答，再用于检索 | 提升召回率 |
| Multi-Query | 用 LLM 生成多个检索查询 | 覆盖不同表达方式 |
| Parent Document | 检索小块但返回大块上下文 | 平衡相关性和完整性 |
| Contextual Compression | 压缩检索到的内容，保留关键信息 | 减少噪声，提高质量 |
| Ensemble | 融合多个检索器的结果 | 提升稳定性 |

## 4. RAG 完整实现示例

### 4.1 使用 LangChain 实现 RAG

```python
# 完整 RAG 实现（LangChain）
from langchain.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chat_models import ChatOpenAI
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate

# 1. 加载文档
loader = PyPDFLoader("docs/2024-q3-report.pdf")
documents = loader.load()

# 2. 文本分割
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50
)
chunks = splitter.split_documents(documents)

# 3. 向量化并存储
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings
)

# 4. 创建检索器
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}
)

# 5. 定义提示词
prompt = PromptTemplate(
    template="""你是一个专业的财务分析师。
请根据以下参考资料回答用户问题。

参考资料：
{context}

用户问题：{question}

请基于上述参考资料回答，并注明来源。""",
    input_variables=["context", "question"]
)

# 6. 创建 QA 链
llm = ChatOpenAI(model="gpt-4")
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=retriever,
    chain_type_kwargs={"prompt": prompt}
)

# 7. 问答
question = "2024年Q3公司的财务表现如何？"
answer = qa_chain.run(question)
print(answer)
```

### 4.2 使用 LlamaIndex 实现 RAG

```python
# 使用 LlamaIndex 实现 RAG
from llama_index import VectorStoreIndex
from llama_index.readers import PDFReader
from llama_index.llms import OpenAI

# 1. 加载文档
reader = PDFReader()
documents = reader.load_data("docs/2024-q3-report.pdf")

# 2. 创建索引
index = VectorStoreIndex.from_documents(documents)

# 3. 创建查询引擎
query_engine = index.as_query_engine(
    llm=OpenAI(model="gpt-4"),
    similarity_top_k=3
)

# 4. 问答
response = query_engine.query("2024年Q3公司的财务表现如何？")
print(response)
```

## 5. RAG 优化策略

### 5.1 检索优化

#### 数据质量优化

- **清洗数据**：去除噪音、重复、无关内容
- **优化分块**：根据内容类型选择合适的分块策略
- **添加元数据**：为文档添加标题、来源、时间等元数据
- **结构化**：使用 Markdown、标题层级等结构化文档

#### 检索效果优化

- **混合检索**：结合向量检索和关键词检索
- **重排序**：使用 Cross-Encoder 对结果重排序
- **查询扩展**：用 LLM 生成相关查询
- **自适应检索**：根据问题类型选择不同策略

### 5.2 生成优化

#### 提示词优化

```python
# 优化后的提示词
prompt = """你是一个专业的助手。请根据以下参考资料回答用户问题。

要求：
1. 只使用参考资料中的信息，不要添加额外知识
2. 如果参考资料中没有相关信息，请明确说明
3. 回答要引用具体的来源

参考资料：
{context}

用户问题：{question}

回答格式：
- 答案：[你的回答]
- 来源：[引用的文档]"""
```

#### 控制幻觉

- **来源标注**：在回答中标注信息来源
- **事实核查**：让 LLM 核查生成内容是否与参考一致
- **置信度**：对不确定的内容标记置信度
- **追问确认**：对关键事实进行追问确认

## 6. RAG 评估与监控

### 6.1 评估指标

| 指标 | 描述 | 评估方法 |
|------|------|----------|
| 召回率（Recall） | 相关文档被检索到的比例 | 人工标注/自动评估 |
| 精确率（Precision） | 检索到的文档中相关的比例 | 人工标注/自动评估 |
| 答案准确率 | 生成的答案是否正确 | 人工评估/LLM 评估 |
| 引用准确率 | 引用来源是否正确 | 人工评估 |
| 响应时间 | 系统响应速度 | 性能监控 |

### 6.2 评估工具

- **RAGAs**：专门评估 RAG 系统的指标框架
- **LangSmith**：LangChain 的评估和监控平台
- **DeepEval**：开源的 LLM 评估框架
- **Trulens**：RAG 评估的开源工具

## 7. RAG 的局限性与未来

### 7.1 当前局限性

- **检索质量依赖**：检索效果直接决定生成质量
- **复杂推理**：难以处理需要多步推理的问题
- **长上下文**：处理超长文档仍有挑战
- **延迟**：检索增加了响应时间
- **成本**：向量存储和检索有额外成本

### 7.2 未来发展趋势

- **原生 RAG**：模型原生支持检索能力
- **Graph RAG**：结合知识图谱，增强关系理解
- **Agentic RAG**：结合 Agent 能力，自主规划检索
- **多模态 RAG**：支持图像、视频等非文本内容
- **实时 RAG**：动态更新知识库，实时检索

## 8. 总结

RAG 是构建企业级 AI 应用的核心技术，它通过将外部知识检索与大语言模型生成相结合，有效解决了 LLM 的知识时效性、幻觉、领域知识缺乏等问题。

### 核心组件

| 组件 | 功能 | 关键技术 |
|------|------|----------|
| 文档处理 | 加载和分割文档 | PDFLoader, 文本分割器 |
| 向量化 | 将文本转为向量 | Embedding 模型 |
| 向量存储 | 存储和检索向量 | Pinecone, Chroma, Milvus |
| 检索 | 找到相关文档 | 相似度搜索, BM25, 混合检索 |
| 生成 | 生成最终回答 | LLM + 提示词 |

---

## 参考资料

- [LangChain RAG 文档](https://python.langchain.com/docs/modules/data_connection/)
- [LlamaIndex 文档](https://docs.llamaindex.ai/en/latest/index.html)
- [RAG 原始论文](https://arxiv.org/abs/2005.11401)
- [RAGAs 评估框架](https://www.ragas.io/)

---

*&copy; 2024 AI DevOps 研究 | [返回目录](./index.md) | [返回首页](../index.html)*