# LLM 微调入门

> 前置知识：了解 [AI 基本概念](./01-ai-basics.md)、[提示词工程](./02-prompt-engineering.md) 和 [LLM Agent](./03-llm-agents.md)

模型微调（Fine-tuning）是指在一个预训练模型的基础上，使用特定领域的数据进行进一步训练，使模型能够更好地完成特定任务。相比于提示词工程，微调能够更深入地定制模型行为，是构建专业化 AI 应用的重要技术。

## 1. 为什么需要微调？

虽然通用的预训练模型能力强大，但在特定场景下往往表现不足。微调可以解决以下问题：

### 1.1 提示词工程的局限性

| 问题 | 提示词工程 | 微调 |
|------|------------|------|
| 知识局限性 | 模型知识受限于训练数据 | 注入新知识 |
| 风格一致性 | 难以保持一致的输出风格 | 学习特定风格 |
| Token 浪费 | 每次请求需要重复提示 | 提示精简，节省成本 |
| 复杂指令 | 复杂指令容易失效 | 内化复杂任务模式 |
| 延迟敏感 | 长提示增加响应时间 | 响应更快 |

### 1.2 微调的典型应用场景

- **领域专属模型**：医疗、法律、金融等专业领域
- **特定风格**：品牌文风、代码规范、对话风格
- **任务优化**：情感分析、实体识别、代码生成等特定任务
- **私有化部署**：数据不出域的敏感场景
- **成本优化**：使用小模型替代大模型

> **建议**：优先尝试提示词工程，只有当提示词工程无法满足需求时再考虑微调。微调成本高、周期长，应作为最后手段。

## 2. 微调的基本概念

### 2.1 预训练 vs 微调

```
# 预训练阶段
大规模通用数据 ──▶ 基础模型（通用能力）

# 微调阶段
领域特定数据 ──▶ 基础模型 ──▶ 微调模型（专业能力）

# 对比
┌──────────────┬─────────────────┬─────────────────┐
│    阶段      │    训练数据     │    目标能力     │
├──────────────┼─────────────────┼─────────────────┤
│ 预训练       │ 海量通用文本     │ 通用语言理解    │
│ 微调         │ 任务/领域数据    │ 特定任务能力    │
└──────────────┴─────────────────┴─────────────────┘
```

### 2.2 微调的训练方式

#### 全参数微调（Full Fine-tuning）

更新模型的所有参数。

- **优点**：效果最好，可塑性最强
- **缺点**：需要大量 GPU 显存和计算资源
- **适用**：有充足资源，数据量大的场景

#### 参数高效微调（PEFT）

只更新部分参数，大大降低资源需求。

| 方法 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| LoRA | 低秩适配，添加可训练的低秩矩阵 | 显存需求低，训练快 | 效果略低于全参数 |
| QLoRA | 量化 + LoRA，进一步压缩 | 可在消费级 GPU 上训练 | 训练时间较长 |
| Prefix Tuning | 在输入前添加可训练前缀 | 参数量极少 | 效果不稳定 |
| Adapter | 在 Transformer 层插入适配器 | 可插拔，便于切换 | 需要修改模型结构 |

### 2.3 训练数据格式

#### Instruction Tuning（指令微调）

```json
// 指令微调数据格式
{
  "instruction": "将以下英文翻译成中文",
  "input": "Hello, world!",
  "output": "你好，世界！"
}

// 更复杂的格式
{
  "system": "你是一个专业的技术文档工程师",
  "instruction": "为以下代码生成文档",
  "input": "def add(a, b): return a + b",
  "output": "## add 函数\n\n...文档内容..."
}
```

#### ChatML 格式

```json
// ChatML 格式（多轮对话）
[
  {"role": "system", "content": "你是一个有帮助的助手"},
  {"role": "user", "content": "什么是机器学习？"},
  {"role": "assistant", "content": "机器学习是..."},
  {"role": "user", "content": "它和深度学习有什么区别？"}
]

// 训练时使用 messages 格式
{
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ]
}
```

## 3. 主流微调框架

### 3.1 Hugging Face Transformers

最通用的深度学习框架，支持大多数预训练模型。

```python
# 使用 Transformers 微调（简化版）
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from trl import SFTTrainer
from datasets import Dataset

# 1. 加载模型和分词器
model_name = "meta-llama/Llama-2-7b-hf"
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    device_map="auto",
    load_in_8bit=True  # 8位量化，减少显存
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# 2. 准备数据
train_data = Dataset.from_list(train_samples)

# 3. 配置训练参数
training_args = TrainingArguments(
    output_dir="./output",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    learning_rate=2e-4,
    save_strategy="epoch",
    logging_steps=10,
)

# 4. 创建训练器
trainer = SFTTrainer(
    model=model,
    train_dataset=train_data,
    tokenizer=tokenizer,
    args=training_args,
    max_seq_length=512,
)

# 5. 开始训练
trainer.train()
```

### 3.2 LoRA 微调（PEFT）

参数高效微调的代表，只需训练少量参数。

```python
# 使用 PEFT 实现 LoRA 微调
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig, get_peft_model, TaskType

# 1. 加载基础模型
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    load_in_8bit=True,
    device_map="auto"
)

# 2. 配置 LoRA
lora_config = LoraConfig(
    r=8,                       # LoRA 秩
    lora_alpha=16,             # LoRA 缩放因子
    target_modules=["q_proj", "v_proj"],  # 目标层
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM
)

# 3. 应用 LoRA
model = get_peft_model(model, lora_config)

# 4. 打印可训练参数
model.print_trainable_parameters()
# 输出: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.06%
```

### 3.3 Unsloth

专注于加速 LoRA 微调的工具，训练速度提升 2-5 倍。

```python
# 使用 Unsloth 加速微调
from unsloth import FastLanguageModel
import torch

# 1. 加载模型（Unsloth 优化版）
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "unsloth/llama-2-7b-bnb-4bit",
    max_seq_length = 2048,
    dtype = torch.float16,
    load_in_4bit = True,
)

# 2. 添加 LoRA
model = FastLanguageModel.get_peft_model(
    model,
    r = 16,
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_alpha = 16,
    lora_dropout = 0,
    bias = "none",
    use_gradient_checkpointing = "unsloth",
)

# 3. 使用 SFTTrainer 训练
# ... 后续训练代码相同
```

### 3.4 框架对比

| 框架 | 特点 | 适用场景 | 学习曲线 |
|------|------|----------|----------|
| Transformers | 通用性强，生态丰富 | 各种微调场景 | 中等 |
| PEFT | 参数高效，显存需求低 | 资源受限场景 | 低 |
| Unsloth | 训练速度快，显存优化好 | 快速实验、小团队 | 低 |
| DeepSpeed | 分布式训练、ZeRO 优化 | 大规模训练 | 高 |
| Axolotl | 一键微调，配置简单 | 快速上手 | 低 |

## 4. 微调实战流程

### 4.1 数据准备

#### 数据质量要点

- **数据量**：通常 1000-10000 条高质量样本即可见效
- **多样性**：覆盖各种场景和边缘情况
- **准确性**：标签准确，格式规范
- **一致性**：风格统一，避免矛盾

#### 数据清洗示例

```python
# 数据清洗脚本示例
import json
import re

def clean_data(sample):
    # 1. 去除特殊字符
    sample["output"] = re.sub(r'\s+', ' ', sample["output"])

    # 2. 去除过短/过长的样本
    if len(sample["output"]) < 10 or len(sample["output"]) > 5000:
        return None

    # 3. 去除重复
    if sample["output"] in seen_outputs:
        return None

    return sample

# 批量处理
cleaned_data = [s for s in raw_data if (s := clean_data(s))]
```

### 4.2 训练配置

#### 关键超参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| Learning Rate | 1e-4 ~ 5e-5 | 学习率过大会导致灾难性遗忘 |
| Epochs | 2~5 | 过拟合前停止 |
| Batch Size | 4~16 | 根据显存调整 |
| Warmup Steps | 100~500 | 学习率预热 |
| Max Length | 512~2048 | 根据任务需求 |

#### 避免灾难性遗忘

```python
# 灾难性遗忘应对策略

# 策略1：降低学习率
learning_rate = 1e-5  # 比预训练低 10-100 倍

# 策略2：混合预训练数据
train_data = interleave(
    domain_data,      # 领域数据 80%
    general_data,     # 通用数据 20%
    probabilities = [0.8, 0.2]
)

# 策略3：使用 LoRA
lora_config = LoraConfig(r=8)  # 只训练少量参数

# 策略4：渐进式解冻
# 逐渐解锁模型层，从顶层向底层
```

### 4.3 训练执行

```python
# 完整的微调训练脚本
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from trl import SFTTrainer
from datasets import Dataset
import torch

# 1. 配置
model_name = "meta-llama/Llama-2-7b-chat-hf"
output_dir = "./llama-finetune"

# 2. 加载模型（4bit 量化）
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,
    load_in_4bit=True,
    device_map="auto",
)

# 3. 加载分词器
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

# 4. 加载数据
dataset = Dataset.from_json("train_data.json")

# 5. 训练参数
training_args = TrainingArguments(
    output_dir=output_dir,
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,    # 梯度累积
    learning_rate=2e-4,
    warmup_steps=100,
    save_strategy="epoch",
    save_total_limit=2,
    logging_steps=10,
    fp16=True,                       # 混合精度
    dataloader_num_workers=4,
    report_to="none",
)

# 6. 创建训练器
trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    args=training_args,
    max_seq_length=1024,
    packing=True,                    # 短序列打包
)

# 7. 开始训练
trainer.train()

# 8. 保存模型
trainer.save_model(f"{output_dir}/final")
tokenizer.save_pretrained(f"{output_dir}/final")
```

### 4.4 评估与部署

#### 评估指标

| 指标 | 描述 | 评估方法 |
|------|------|----------|
| Loss | 训练损失 | 训练日志 |
| BLEU/ROUGE | 文本生成质量 | 自动评估 |
| 人工评估 | 输出质量主观评估 | 人工打分 |
| 特定任务指标 | 准确率、F1等 | 任务相关评估 |

#### 部署方式

- **本地部署**：使用 vLLM、Text Generation Inference 等
- **云端部署**：AWS SageMaker、Azure ML、阿里云 PAI
- **容器化**：Docker + Kubernetes
- **API 服务**：FastAPI、Flask 等框架包装

## 5. 常见问题与解决方案

### 5.1 灾难性遗忘

模型在微调后忘记了预训练学到的知识。

- 使用较低的学习率（1e-5 ~ 5e-5）
- 混合通用数据一起训练
- 使用 LoRA 等参数高效方法
- 保存检查点，选择最优模型

### 5.2 过拟合

模型在训练数据上表现好，但泛化能力差。

- 增加训练数据量
- 使用正则化（dropout、weight decay）
- 减少训练轮数
- 使用验证集监控过拟合

### 5.3 训练不稳定

- 使用梯度累积
- 降低学习率
- 添加学习率预热（warmup）
- 检查数据质量

### 5.4 显存不足

- 使用量化（4bit/8bit）
- 使用 LoRA 微调
- 减小 batch size
- 使用梯度 checkpointing

## 6. 总结

微调是定制化 LLM 的重要手段，但成本较高。建议先尝试提示词工程，只有在提示词无法满足需求时才考虑微调。LoRA 是目前最流行的参数高效微调方法，可以在消费级 GPU 上完成训练。

### 方法对比

| 方法 | 显存需求 | 训练时间 | 效果 |
|------|----------|----------|------|
| 全参数微调 | 极高（多卡） | 长 | 最好 |
| LoRA | 中（单卡） | 中 | 较好 |
| QLoRA | 低（消费级） | 较长 | 较好 |
| Prefix Tuning | 低 | 短 | 一般 |

---

## 参考资料

- [Hugging Face Transformers 训练文档](https://huggingface.co/docs/transformers/en/training)
- [PEFT GitHub](https://github.com/huggingface/peft)
- [Unsloth 官网](https://unsloth.ai/)
- [LoRA 论文](https://arxiv.org/abs/2106.09685)

---

*&copy; 2024 AI DevOps 研究 | [返回目录](./index.md) | [返回首页](../index.html)*