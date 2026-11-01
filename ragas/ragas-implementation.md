# Ragas 评估框架深度分析

## 目录

- [1. 概述](#1-概述)
- [2. 核心架构](#2-核心架构)
  - [2.1 系统整体架构](#21-系统整体架构)
  - [2.2 评估流程](#22-评估流程)
- [3. 数据模型](#3-数据模型)
  - [3.1 样本类型](#31-样本类型)
  - [3.2 消息类型关系](#32-消息类型关系)
- [4. RAG 评估](#4-rag-评估)
  - [4.1 RAG 评估原理](#41-rag-评估原理)
  - [4.2 RAG 评估流程详解](#42-rag-评估流程详解)
- [5. RAG 评估指标详解](#5-rag-评估指标详解)
  - [5.1 Faithfulness（忠实度）](#51-faithfulness忠实度)
  - [5.2 Answer Relevancy（答案相关性）](#52-answer-relevancy答案相关性)
  - [5.3 Context Precision（上下文精度）](#53-context-precision上下文精度)
  - [5.4 Context Recall（上下文召回）](#54-context-recall上下文召回)
  - [5.5 Answer Correctness（答案正确性）](#55-answer-correctness答案正确性)
  - [5.6 Factual Correctness（事实正确性）](#56-factual-correctness事实正确性)
  - [5.7 Context Entities Recall（上下文实体召回）](#57-context-entities-recall上下文实体召回)
  - [5.8 Noise Sensitivity（噪声敏感度）](#58-noise-sensitivity噪声敏感度)
- [6. Agent 评估](#6-agent-评估)
  - [6.1 Agent 评估原理](#61-agent-评估原理)
  - [6.2 Agent 评估流程](#62-agent-评估流程)
- [7. Agent 评估指标详解](#7-agent-评估指标详解)
  - [7.1 Tool Call Accuracy（工具调用准确度）](#71-tool-call-accuracy工具调用准确度)
  - [7.2 Agent Goal Accuracy（目标达成度）](#72-agent-goal-accuracy目标达成度)
  - [7.3 Topic Adherence Score（主题遵循度）](#73-topic-adherence-score主题遵循度)
- [8. Workflow 评估](#8-workflow-评估)
  - [8.1 Workflow 评估原理](#81-workflow-评估原理)
  - [8.2 Workflow 评估适用指标](#82-workflow-评估适用指标)
- [9. 其他重要指标](#9-其他重要指标)
  - [9.1 Aspect Critic（方面评判）](#91-aspect-critic方面评判)
  - [9.2 Summarization Score（摘要质量评分）](#92-summarization-score摘要质量评分)
  - [9.3 非 LLM 指标](#93-非-llm-指标)
- [10. 指标选择指南](#10-指标选择指南)
  - [10.1 RAG 场景指标选择](#101-rag-场景指标选择)
  - [10.2 Agent 场景指标选择](#102-agent-场景指标选择)
  - [10.3 指标依赖关系](#103-指标依赖关系)
- [11. 关键组件索引](#11-关键组件索引)
  - [11.1 核心文件](#111-核心文件)
  - [11.2 RAG 指标](#112-rag-指标)
  - [11.3 Agent 指标](#113-agent-指标)
  - [11.4 通用指标](#114-通用指标)
- [12. 使用示例](#12-使用示例)
  - [12.1 RAG 评估示例](#121-rag-评估示例)
  - [12.2 Agent 评估示例](#122-agent-评估示例)
- [附录：完整指标速查表](#附录完整指标速查表)

---

## 1. 概述

**Ragas** 是一个专为 LLM 应用设计的评估框架，提供客观的评估指标、智能测试数据生成和数据驱动的分析能力。它支持三种主要评估场景：

- **RAG 评估**：评估检索增强生成系统的检索质量和生成质量
- **Agent 评估**：评估 AI Agent 的工具调用准确性和目标达成能力
- **Workflow 评估**：评估复杂工作流的端到端执行效果

---

## 2. 核心架构

### 2.1 系统整体架构

> **架构说明**：Ragas 采用模块化设计，由 Metrics 层、LLM/Embeddings 层、Prompt 层和 Evaluation 层组成。Metrics 定义了评估逻辑，LLM/Embeddings 提供推理能力，Evaluation 层负责协调整个评估流程。

```mermaid
graph TB
    subgraph User["用户层"]
        App["LLM 应用<br/>RAG / Agent / Workflow"]
        TestData["测试数据<br/>SingleTurnSample / MultiTurnSample"]
    end

    subgraph Evaluation["评估层"]
        Eval["evaluate() / aevaluate()"]
        Executor["Executor<br/>并行执行器"]
        ColumnMap["Column Mapping<br/>列名映射"]
    end

    subgraph Metrics["指标层 (Metrics)"]
        direction LR
        RAGMetrics["RAG 指标"]
        AgentMetrics["Agent 指标"]
        StringMetrics["字符串指标"]
        LLMMetrics["LLM-based 指标"]
    end

    subgraph RAGDetail["RAG 指标详情"]
        Faithfulness["Faithfulness<br/>忠实度"]
        AnswerRel["Answer Relevancy<br/>答案相关性"]
        ContextPrec["Context Precision<br/>上下文精度"]
        ContextRec["Context Recall<br/>上下文召回"]
    end

    subgraph AgentDetail["Agent 指标详情"]
        ToolAcc["Tool Call Accuracy<br/>工具调用准确度"]
        GoalAcc["Agent Goal Accuracy<br/>目标达成度"]
        TopicAdh["Topic Adherence<br/>主题遵循度"]
    end

    subgraph Infrastructure["基础设施层"]
        LLM["LLM Provider<br/>OpenAI / Anthropic / ..."]
        Embeddings["Embedding Provider<br/>OpenAI / HuggingFace"]
        PromptEngine["Prompt Engine<br/>PydanticPrompt"]
    end

    subgraph Storage["存储层"]
        Results["EvaluationResult<br/>评估结果"]
        Dataset["Dataset<br/>数据集"]
    end

    App --> TestData
    TestData --> Eval
    Eval --> Executor
    Eval --> ColumnMap
    Executor --> Metrics
    Metrics --> RAGDetail
    Metrics --> AgentDetail
    RAGDetail --> Infrastructure
    AgentDetail --> Infrastructure
    Infrastructure --> LLM
    Infrastructure --> Embeddings
    Infrastructure --> PromptEngine
    Eval --> Results
    Eval --> Dataset
```

### 2.2 评估流程

> **流程说明**：评估从数据集开始，经过验证、初始化、并行执行，最终输出评估结果。每个样本独立评估，支持异步并行处理。

```mermaid
flowchart TD
    Start([开始评估]) --> LoadDataset["加载数据集<br/>SingleTurnSample / MultiTurnSample"]
    LoadDataset --> Validate["验证数据<br/>validate_required_columns()"]
    Validate --> InitModels["初始化模型<br/>设置 LLM / Embeddings"]

    InitModels --> ForEachSample["遍历每个样本"]

    ForEachSample --> AsyncExec["异步并行执行<br/>Executor"]

    AsyncExec --> ComputeMetrics["计算指标"]

    subgraph MetricCompute["指标计算过程"]
        Extract["提取所需字段<br/>user_input, response, context..."]
        LLMCall["调用 LLM<br/>生成判断/分类"]
        Score["计算分数<br/>基于验证结果"]
        Extract --> LLMCall --> Score
    end

    ComputeMetrics --> MetricCompute
    MetricCompute --> Aggregate["聚合结果<br/>计算平均值"]
    Aggregate --> Output["输出 EvaluationResult"]

    Output --> End([完成])

    style Start fill:#e1f5fe
    style End fill:#e8f5e9
    style MetricCompute fill:#fff3e0
```

---

## 3. 数据模型

### 3.1 样本类型

> **样本模型说明**：Ragas 支持两种样本类型：SingleTurnSample（单轮交互）用于 RAG 评估，MultiTurnSample（多轮对话）用于 Agent/Workflow 评估。

```mermaid
classDiagram
    class SingleTurnSample {
        +str user_input
        +List~str~ retrieved_contexts
        +List~str~ reference_contexts
        +List~str/int~ retrieved_context_ids
        +List~str/int~ reference_context_ids
        +str response
        +List~str~ multi_responses
        +str reference
        +Dict rubrics
        +to_dict() Dict
        +get_features() List
    }

    class MultiTurnSample {
        +List~Message~ user_input
        +str reference
        +List~ToolCall~ reference_tool_calls
        +Dict rubrics
        +List~str~ reference_topics
        +to_messages() List
        +pretty_repr() str
    }

    class HumanMessage {
        +str content
        +str type = "human"
        +pretty_repr() str
    }

    class AIMessage {
        +str content
        +str type = "ai"
        +List~ToolCall~ tool_calls
        +pretty_repr() str
    }

    class ToolMessage {
        +str content
        +str type = "tool"
        +pretty_repr() str
    }

    class ToolCall {
        +str name
        +Dict args
    }

    SingleTurnSample <|-- BaseSample
    MultiTurnSample <|-- BaseSample
    MultiTurnSample --> HumanMessage
    MultiTurnSample --> AIMessage
    MultiTurnSample --> ToolMessage
    AIMessage --> ToolCall
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `user_input` | str / List[Message] | 用户输入的问题或对话 |
| `retrieved_contexts` | List[str] | 系统检索到的上下文 |
| `reference_contexts` | List[str] | 参考上下文（ground truth） |
| `response` | str | 系统生成的回答 |
| `reference` | str | 参考回答（ground truth） |
| `reference_tool_calls` | List[ToolCall] | 期望的工具调用序列 |
| `reference_topics` | List[str] | 期望讨论的主题列表 |

### 3.2 消息类型关系

> **消息关系说明**：MultiTurnSample 中的对话由 HumanMessage、AIMessage 和 ToolMessage 组成，模拟真实的 Agent 交互流程。

```mermaid
sequenceDiagram
    participant H as Human
    participant AI as AI Agent
    participant T as Tool

    H->>AI: HumanMessage("帮我预订餐厅")
    AI->>T: AIMessage + ToolCall<br/>restaurant_search(cuisine="Chinese")
    T->>AI: ToolMessage("找到: 金龙, 玉宫")
    AI->>H: AIMessage("找到以下选择...")
    H->>AI: HumanMessage("选金龙")
    AI->>T: AIMessage + ToolCall<br/>restaurant_book(name="金龙")
    T->>AI: ToolMessage("预订成功")
    AI->>H: AIMessage("已预订金龙餐厅")
```

---

## 4. RAG 评估

### 4.1 RAG 评估原理

> **RAG 原理说明**：RAG（Retrieval-Augmented Generation）系统通过先检索相关上下文，再生成回答。评估需要同时考察**检索质量**和**生成质量**两个维度。

```mermaid
flowchart LR
    subgraph RAG["RAG 系统流程"]
        Q["用户问题<br/>user_input"] --> Retriever["检索器<br/>Retriever"]
        Retriever -->|retrieved_contexts| Generator["生成器<br/>Generator"]
        Generator --> A["生成回答<br/>response"]
    end

    subgraph Eval["评估维度"]
        direction TB
        RetEval["检索评估<br/>• Context Precision<br/>• Context Recall"]
        GenEval["生成评估<br/>• Faithfulness<br/>• Answer Relevancy<br/>• Answer Correctness"]
    end

    RAG --> Eval

    subgraph Ref["参考数据"]
        RC["reference_contexts<br/>参考上下文"]
        Ref_A["reference<br/>参考答案"]
    end

    Ref --> RetEval
    Ref --> GenEval

    style RAG fill:#e3f2fd
    style Eval fill:#f3e5f5
    style Ref fill:#e8f5e9
```

### 4.2 RAG 评估流程详解

> **评估流程说明**：RAG 评估从收集用户输入、检索上下文、生成回答开始，然后使用多个指标从不同角度评估系统质量。

```mermaid
flowchart TD
    Start([用户提问]) --> Query["user_input:<br/>'什么是量子计算?'"]

    Query --> Retrieve["检索步骤<br/>Retriever"]
    Retrieve --> Contexts["retrieved_contexts:<br/>['量子计算是一种...'<br/>'量子比特可以...']"]

    Contexts --> Generate["生成步骤<br/>Generator LLM"]
    Generate --> Response["response:<br/>'量子计算利用量子力学原理...'"]

    Response --> Eval1["Faithfulness<br/>回答是否忠于检索内容?"]
    Response --> Eval2["Answer Relevancy<br/>回答是否切题?"]
    Contexts --> Eval3["Context Precision<br/>检索内容是否相关?"]
    Contexts --> Eval4["Context Recall<br/>是否检索到所有相关内容?"]

    Eval1 --> Score1["faithfulness_score: 0.85"]
    Eval2 --> Score2["answer_relevancy: 0.92"]
    Eval3 --> Score3["context_precision: 0.78"]
    Eval4 --> Score4["context_recall: 0.71"]

    Score1 --> Final["综合评估结果"]
    Score2 --> Final
    Score3 --> Final
    Score4 --> Final
```

---

## 5. RAG 评估指标详解

### 5.1 Faithfulness（忠实度）

> **指标说明**：Faithfulness 衡量生成的回答是否忠实于检索到的上下文，即回答中的信息是否都能从上下文中推导出来，用于检测幻觉（hallucination）。

**计算公式**：
```
Faithfulness = 忠实陈述数 / 总陈述数
```

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input, response, retrieved_contexts"] --> Step1["1. 陈述分解<br/>将回答分解为独立陈述"]

    Step1 --> Statements["生成陈述列表:<br/>['陈述1', '陈述2', ...]"]

    Statements --> Step2["2. NLI 验证<br/>逐个验证陈述是否可从上下文推导"]

    Step2 --> Verdicts["验证结果:<br/>[1, 0, 1, 1, ...]<br/>1=可推导, 0=不可推导"]

    Verdicts --> Step3["3. 计算分数<br/>faithful_count / total_count"]

    Step3 --> Score["Faithfulness Score: 0.0 ~ 1.0"]

    style Input fill:#e1f5fe
    style Score fill:#e8f5e9
```

**示例**：

```
user_input: "量子计算有什么优势?"

retrieved_contexts:
  "量子计算利用量子比特(qubit)进行计算。与传统比特不同，
   量子比特可以处于叠加态，这使得量子计算机在某些特定问题上
   具有指数级的加速能力。"

response:
  "量子计算使用量子比特进行计算，可以在特定问题上实现指数级加速。
   量子计算机已经可以破解所有现有的加密系统。"

分解的陈述:
1. "量子计算使用量子比特进行计算" → ✓ (可从上下文推导)
2. "可以在特定问题上实现指数级加速" → ✓ (可从上下文推导)
3. "量子计算机已经可以破解所有现有的加密系统" → ✗ (上下文未提及)

Faithfulness = 2/3 = 0.67
```

---

### 5.2 Answer Relevancy（答案相关性）

> **指标说明**：Answer Relevancy 衡量生成的回答与用户问题的相关程度，回答中的不完整、冗余或不相关信息会被扣分。

**计算原理**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input, response"] --> Step1["1. 从回答反向生成问题<br/>生成 N 个问题(默认 N=3)"]

    Step1 --> GenQuestions["生成的问题:<br/>['Q1', 'Q2', 'Q3']"]

    GenQuestions --> Step2["2. 检查回答是否模糊<br/>noncommittal check"]

    Step2 --> Check{"回答是否模糊?"}
    Check -->|是| Score0["Score = 0"]
    Check -->|否| Step3["3. 计算语义相似度<br/>原始问题 vs 生成问题"]

    Step3 --> Embedding["使用 Embedding<br/>计算余弦相似度"]

    Embedding --> Score["Score = mean(cosine_similarities)"]

    style Input fill:#e1f5fe
    style Score fill:#e8f5e9
    style Score0 fill:#ffebee
```

**示例**：

```
user_input: "什么是机器学习?"

response (好): "机器学习是人工智能的一个分支，它使计算机能够从数据中
              学习并改进性能，而无需显式编程。"

→ 生成的问题: ["机器学习是什么?", "什么是AI的一个分支?", ...]
→ 相似度: [0.95, 0.72, ...]
→ Answer Relevancy: 0.89

response (差): "我不太确定，但这可能和计算机有关..."

→ noncommittal = 1
→ Answer Relevancy: 0.0
```

---

### 5.3 Context Precision（上下文精度）

> **指标说明**：Context Precision 衡量检索到的上下文中，有多少是真正有用的。它使用 Average Precision 来计算，强调排名靠前的上下文应该是相关的。

**计算公式**：

```
Context Precision = Σ(Precision@k × relevance[k]) / 相关上下文总数
```

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input, retrieved_contexts, reference"] --> Loop["遍历每个检索到的上下文"]

    Loop --> Verify["验证上下文是否有用<br/>QAC Prompt"]

    Verify --> Verdict["verdict: 1(有用) / 0(无用)"]

    Verdict --> Calculate["计算 Average Precision"]

    Calculate --> Example["示例:<br/>contexts: [C1, C2, C3, C4]<br/>verdicts: [1, 0, 1, 0]<br/><br/>Precision@1 = 1/1 = 1.0<br/>Precision@3 = 2/3 = 0.67<br/>AP = (1.0 + 0.67) / 2 = 0.83"]

    Example --> Score["Context Precision: 0.83"]

    style Input fill:#e1f5fe
    style Score fill:#e8f5e9
```

**示例**：

```
user_input: "谁赢得了2020年ICC世界杯?"

retrieved_contexts:
  C1: "2022 ICC T20世界杯在澳大利亚举行...英格兰获胜" → 1 (有用)
  C2: "板球是一项流行的运动..." → 0 (无用)
  C3: "英格兰队在2019年赢得了ODI世界杯" → 1 (部分有用)

verdicts: [1, 0, 1]
Precision@1 = 1/1 = 1.0
Precision@3 = 2/3 = 0.67
Context Precision = (1.0 + 0.67) / 2 = 0.835
```

---

### 5.4 Context Recall（上下文召回）

> **指标说明**：Context Recall 衡量检索系统是否检索到了所有必要的信息。它通过检查参考答案中的语句是否能从检索到的上下文中推导出来来评估。

**计算公式**：

```
Context Recall = 可从上下文推导的参考答案语句数 / 参考答案总语句数
```

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input, retrieved_contexts, reference"] --> Step1["1. 分解参考答案<br/>为独立语句"]

    Step1 --> Statements["参考语句:<br/>['S1', 'S2', 'S3', ...]"]

    Statements --> Step2["2. 对每个语句验证<br/>是否可从检索上下文推导"]

    Step2 --> Classify["分类结果:<br/>attributed=1: 可推导<br/>attributed=0: 不可推导"]

    Classify --> Step3["3. 计算召回率"]

    Step3 --> Formula["Recall = attributed_count / total_statements"]

    Formula --> Score["Context Recall: 0.0 ~ 1.0"]

    style Input fill:#e1f5fe
    style Score fill:#e8f5e9
```

**示例**：

```
user_input: "爱因斯坦的生平?"

reference:
  "爱因斯坦1879年3月14日出生于德国。他是理论物理学家，
   因相对论而闻名。他于1921年获得诺贝尔物理学奖。
   他在1905年发表了4篇论文。"

retrieved_contexts:
  "爱因斯坦(1879-1955)是德国理论物理学家...
   因相对论闻名...1921年获得诺贝尔奖..."

分解的参考语句:
1. "爱因斯坦1879年3月14日出生" → attributed=1 ✓
2. "他是理论物理学家" → attributed=1 ✓
3. "因相对论闻名" → attributed=1 ✓
4. "1921年获得诺贝尔奖" → attributed=1 ✓
5. "1905年发表4篇论文" → attributed=0 ✗ (上下文未提及)

Context Recall = 4/5 = 0.8
```

---

### 5.5 Answer Correctness（答案正确性）

> **指标说明**：Answer Correctness 通过比较生成的回答与参考答案，评估回答的事实正确性。它结合了事实性(F1)和语义相似度两个维度。

**计算公式**：

```
Answer Correctness = w1 × F1_score + w2 × Semantic_similarity
默认权重: w1=0.75, w2=0.25
```

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input, response, reference"] --> Step1["1. 分解回答和参考为陈述"]

    Step1 --> Statements["回答陈述: ['A1', 'A2', ...]<br/>参考陈述: ['R1', 'R2', ...]"]

    Statements --> Step2["2. 分类每个陈述<br/>TP / FP / FN"]

    Step2 --> Classification["TP: 回答中有，参考支持<br/>FP: 回答中有，参考不支持<br/>FN: 参考中有，回答没有"]

    Classification --> Step3["3. 计算 F1 Score"]

    Step3 --> F1["F1 = TP / (TP + 0.5*(FP+FN))"]

    F1 --> Step4["4. 计算语义相似度"]

    Step4 --> Sim["使用 Embedding 计算<br/>response vs reference 的余弦相似度"]

    Sim --> Final["Answer Correctness = 0.75×F1 + 0.25×Similarity"]

    style Input fill:#e1f5fe
    style Final fill:#e8f5e9
```

**示例**：

```
user_input: "太阳的动力来源是什么?"

response: "太阳通过核裂变产生能量，类似于地球上的核反应堆。"

reference: "太阳通过核聚变产生能量，氢原子聚合成氦。这个过程释放巨大能量。"

回答陈述: ["太阳通过核裂变产生能量"]
参考陈述: ["太阳通过核聚变产生能量", "氢原子聚合成氦", "这个过程释放巨大能量"]

分类:
- "太阳通过核裂变产生能量" → FP (错误! 应该是核聚变)
- "太阳通过核聚变产生能量" → FN (回答中缺失)
- "氢原子聚合成氦" → FN
- "这个过程释放巨大能量" → FN

TP=0, FP=1, FN=3
F1 = 0 / (0 + 0.5×4) = 0

Answer Correctness ≈ 0 (事实完全错误)
```

---

### 5.6 Factual Correctness（事实正确性）

> **指标说明**：Factual Correctness 使用 claim 分解和 NLI 验证来评估回答的事实正确性。支持 precision、recall 和 F1 三种模式。

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>response, reference"] --> Decompose1["分解回答为 claims"]
    Input --> Decompose2["分解参考为 claims"]

    Decompose1 --> Claims1["回答 claims: ['C1', 'C2', ...]"]
    Decompose2 --> Claims2["参考 claims: ['R1', 'R2', ...]"]

    Claims1 --> Verify1["验证回答 claims<br/>是否可从参考推导"]
    Claims2 --> Verify2["验证参考 claims<br/>是否可从回答推导"]

    Verify1 --> TP_FP["TP + FP 统计"]
    Verify2 --> FN["FN 统计"]

    TP_FP --> Calculate["计算分数"]
    FN --> Calculate

    Calculate --> Mode{"评估模式?"}
    Mode -->|precision| P["Precision = TP / (TP + FP)"]
    Mode -->|recall| R["Recall = TP / (TP + FN)"]
    Mode -->|f1| F["F1 = 2TP / (2TP + FP + FN)"]
```

---

### 5.7 Context Entities Recall（上下文实体召回）

> **指标说明**：Context Entities Recall 衡量检索到的上下文中包含的命名实体占参考上下文中实体的比例。

**计算公式**：

```
Context Entities Recall = |检索上下文中的实体 ∩ 参考上下文中的实体| / |参考上下文中的实体|
```

**示例**：

```
reference_contexts:
  "苹果公司由史蒂夫·乔布斯于1976年在加利福尼亚州库比蒂诺创立。
   2023年市值达到3万亿美元。"

retrieved_contexts:
  "苹果公司是一家科技公司，总部设在加利福尼亚州。"

参考实体: {苹果公司, 史蒂夫·乔布斯, 1976, 库比蒂诺, 加利福尼亚州, 3万亿美元, 2023}
检索实体: {苹果公司, 加利福尼亚州}

Context Entities Recall = 2/7 = 0.29
```

---

### 5.8 Noise Sensitivity（噪声敏感度）

> **指标说明**：Noise Sensitivity 衡量系统对无关信息的敏感程度。它检测回答中有多少错误信息来自于无关的检索内容。

**评估模式**：
- `relevant`: 衡量来自相关上下文的错误
- `irrelevant`: 衡量来自无关上下文的错误

---

## 6. Agent 评估

### 6.1 Agent 评估原理

> **Agent 原理说明**：AI Agent 通过与用户和工具的多轮交互来完成任务。评估需要考察工具调用的准确性、目标达成度和主题遵循度。

```mermaid
flowchart TB
    subgraph AgentSystem["Agent 系统"]
        User["用户输入"] --> Agent["AI Agent<br/>(LLM + Memory)"]
        Agent -->|"需要工具"| ToolCall["发出工具调用"]
        ToolCall --> Tool["执行工具"]
        Tool -->|"返回结果"| Agent
        Agent -->|"不需要工具"| Response["生成回答"]
        Response --> User
    end

    subgraph EvalDimensions["评估维度"]
        ToolAcc["工具调用准确性<br/>Tool Call Accuracy"]
        GoalAcc["目标达成度<br/>Goal Accuracy"]
        TopicAdh["主题遵循度<br/>Topic Adherence"]
    end

    AgentSystem --> EvalDimensions

    style AgentSystem fill:#e3f2fd
    style EvalDimensions fill:#f3e5f5
```

### 6.2 Agent 评估流程

> **评估流程说明**：Agent 评估收集完整的交互轨迹（Human → AI → Tool → AI → ...），然后从多个维度评估 Agent 的表现。

```mermaid
sequenceDiagram
    participant User
    participant Agent
    participant Tools
    participant Evaluator

    User->>Agent: "帮我预订今晚的餐厅"
    Agent->>Tools: ToolCall: restaurant_search(cuisine="Chinese", time="8pm")
    Tools->>Agent: ToolMessage: "找到: 金龙, 玉宫"
    Agent->>User: "找到以下选择..."
    User->>Agent: "选金龙"
    Agent->>Tools: ToolCall: restaurant_book(name="金龙", time="8pm")
    Tools->>Agent: ToolMessage: "预订成功"
    Agent->>User: "已预订金龙餐厅"

    Note over Evaluator: 评估开始
    Evaluator->>Evaluator: Tool Call Accuracy
    Evaluator->>Evaluator: Goal Accuracy
    Evaluator->>Evaluator: Topic Adherence
    Evaluator->>Evaluator: 输出评估结果
```

---

## 7. Agent 评估指标详解

### 7.1 Tool Call Accuracy（工具调用准确度）

> **指标说明**：Tool Call Accuracy 衡量 Agent 的工具调用与参考工具调用的匹配程度。评估包括序列对齐和参数准确性两个方面。

**计算公式**：

```
如果序列不对齐: Score = 0
如果序列对齐: Score = 平均参数准确率 × 序列对齐因子
```

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input (对话轨迹)<br/>reference_tool_calls"] --> Step1["1. 提取预测的工具调用<br/>从 AIMessage 中提取"]

    Step1 --> PredCalls["预测调用:<br/>[ToolCall1, ToolCall2, ...]"]

    PredCalls --> Step2{"2. 序列是否对齐?"}

    Step2 -->|严格模式| CheckStrict["检查名称序列完全匹配"]
    Step2 -->|灵活模式| CheckFlex["检查名称集合相同"]

    CheckStrict --> Aligned{"对齐?"}
    CheckFlex --> Aligned

    Aligned -->|否| Score0["Score = 0"]
    Aligned -->|是| Step3["3. 计算参数准确率"]

    Step3 --> ArgsCompare["比较每个工具调用的参数:<br/>predicted_args vs reference_args"]

    ArgsCompare --> ArgScore["参数准确率 = 匹配参数数 / 总参数数"]

    ArgScore --> Final["Tool Call Accuracy = 参数准确率"]

    style Input fill:#e1f5fe
    style Score0 fill:#ffebee
    style Final fill:#e8f5e9
```

**示例**：

```
参考工具调用:
  reference_tool_calls = [
    ToolCall(name="restaurant_search", args={"cuisine": "Chinese", "time": "8pm"}),
    ToolCall(name="restaurant_book", args={"name": "金龙", "time": "8pm"})
  ]

Agent 实际调用:
  预测调用 = [
    ToolCall(name="restaurant_search", args={"cuisine": "Chinese", "time": "8pm"}),
    ToolCall(name="restaurant_book", args={"name": "金龙", "time": "8pm"})
  ]

序列对齐: ✓ (名称序列相同)
参数匹配:
  - restaurant_search: cuisine ✓, time ✓ → 2/2
  - restaurant_book: name ✓, time ✓ → 2/2

Tool Call Accuracy = (1.0 + 1.0) / 2 = 1.0

---

另一个例子 (参数不完全匹配):
  预测调用 = [
    ToolCall(name="restaurant_search", args={"cuisine": "Chinese"}),  # 缺少 time
    ToolCall(name="restaurant_book", args={"name": "金龙", "time": "8pm"})
  ]

序列对齐: ✓
参数匹配:
  - restaurant_search: cuisine ✓ → 1/2 = 0.5
  - restaurant_book: name ✓, time ✓ → 2/2 = 1.0

Tool Call Accuracy = (0.5 + 1.0) / 2 = 0.75
```

---

### 7.2 Agent Goal Accuracy（目标达成度）

> **指标说明**：Agent Goal Accuracy 评估 Agent 是否成功完成了用户的任务目标。使用 LLM 推断用户目标和最终状态，然后比较实际结果与期望结果。

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input (完整对话轨迹)<br/>reference (期望结果)"] --> Step1["1. 推断用户目标<br/>InferGoalOutcomePrompt"]

    Step1 --> Goal["提取:<br/>• user_goal: 用户想要完成的任务<br/>• end_state: 工作流的最终状态"]

    Goal --> Step2{"有 reference?"}

    Step2 -->|有| Compare1["2a. 比较期望结果与实际结果<br/>CompareOutcomePrompt"]
    Step2 -->|无| Compare2["2b. 比较推断目标与最终状态<br/>CompareOutcomePrompt"]

    Compare1 --> Verdict
    Compare2 --> Verdict

    Verdict["verdict: 0 (未达成) / 1 (达成)"]

    Verdict --> Score["Agent Goal Accuracy: 0 或 1"]

    style Input fill:#e1f5fe
    style Score fill:#e8f5e9
```

**示例**：

```
对话轨迹:
  Human: "帮我预订今晚8点最近最好的中餐厅"
  AI: "找到以下选择: 金龙, 玉宫"
  Human: "选金龙"
  AI: "已预订金龙餐厅晚上8点"
  Human: "谢谢"

LLM 推断:
  user_goal: "预订最近最好的中餐厅，晚上8点"
  end_state: "成功预订金龙餐厅(中餐厅)晚上8点"

reference: "成功预订任何中餐厅晚上8点"

比较结果:
  期望: 成功预订中餐厅晚上8点
  实际: 成功预订金龙餐厅(中餐厅)晚上8点
  verdict: 1 ✓

Agent Goal Accuracy = 1
```

---

### 7.3 Topic Adherence Score（主题遵循度）

> **指标说明**：Topic Adherence Score 评估 Agent 在对话中是否遵循了预期的主题。支持 precision、recall 和 F1 三种模式。

**计算公式**：

```
Precision = 正确回答的主题数 / 实际回答的主题数
Recall = 正确回答的主题数 / 期望回答的主题数
F1 = 2 × Precision × Recall / (Precision + Recall)
```

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>user_input (对话)<br/>reference_topics"] --> Step1["1. 提取对话中的主题"]

    Step1 --> Topics["提取的主题:<br/>['主题1', '主题2', ...]"]

    Topics --> Step2["2. 检查每个主题是否被拒绝回答"]

    Step2 --> Refused{"AI 是否拒绝回答?"}

    Refused -->|是| NotAnswered["标记为未回答"]
    Refused -->|否| Answered["标记为已回答"]

    NotAnswered --> Step3
    Answered --> Step3

    Step3["3. 将提取的主题与参考主题分类匹配"]

    Step3 --> Match["匹配结果:<br/>主题是否属于参考主题之一?"]

    Match --> Calculate["计算 Precision / Recall / F1"]

    Calculate --> Score["Topic Adherence Score"]

    style Input fill:#e1f5fe
    style Score fill:#e8f5e9
```

**示例**：

```
reference_topics: ["相对论", "量子力学"]

对话中提取的主题:
  - "爱因斯坦的相对论" → 匹配 "相对论" ✓
  - "广义相对论" → 匹配 "相对论" ✓
  - "薛定谔方程" → 不匹配 ✗

假设所有主题都被回答(未拒绝):
  TP = 2 (相对论相关的主题被正确回答)
  FP = 1 (薛定谔方程不是期望主题)
  FN = 0

Precision = 2 / (2 + 1) = 0.67
Recall = 2 / (2 + 0) = 1.0
F1 = 2 × 0.67 × 1.0 / (0.67 + 1.0) = 0.8
```

---

## 8. Workflow 评估

### 8.1 Workflow 评估原理

> **Workflow 原理说明**：Workflow 评估针对复杂的多步骤工作流，评估整个流程的执行效果。它与 Agent 评估类似，但更关注端到端的结果。

```mermaid
flowchart LR
    subgraph Workflow["工作流示例"]
        Step1["Step 1: 理解需求"] --> Step2["Step 2: 检索信息"]
        Step2 --> Step3["Step 3: 分析数据"]
        Step3 --> Step4["Step 4: 生成报告"]
        Step4 --> Step5["Step 5: 输出结果"]
    end

    subgraph EvalMetrics["评估指标"]
        GoalAcc["Goal Accuracy<br/>最终目标是否达成?"]
        TopicAdh["Topic Adherence<br/>是否覆盖了所有主题?"]
        Quality["Quality Metrics<br/>输出质量如何?"]
    end

    Workflow --> EvalMetrics

    style Workflow fill:#e3f2fd
    style EvalMetrics fill:#f3e5f5
```

### 8.2 Workflow 评估适用指标

> **指标选择说明**：Workflow 评估可以组合使用多种指标，根据工作流的特点选择合适的评估维度。

| 评估维度 | 推荐指标 | 说明 |
|----------|----------|------|
| 目标达成 | Agent Goal Accuracy | 工作流是否完成预期目标 |
| 主题覆盖 | Topic Adherence | 是否覆盖了所有预期主题 |
| 输出忠实度 | Faithfulness | 输出是否忠于输入数据 |
| 输出相关性 | Answer Relevancy | 输出是否与输入相关 |
| 事实正确性 | Factual Correctness | 输出内容是否事实正确 |

---

## 9. 其他重要指标

### 9.1 Aspect Critic（方面评判）

> **指标说明**：Aspect Critic 是一个灵活的指标，允许用户自定义评判标准，对输出的特定方面进行二元判断。

**预定义方面**：

| 方面 | 定义 |
|------|------|
| harmfulness | 是否会造成伤害? |
| maliciousness | 是否意图欺骗或剥削? |
| coherence | 是否逻辑连贯? |
| correctness | 是否事实正确? |
| conciseness | 是否简洁高效? |

**使用示例**：

```python
# 自定义方面评判
professionalism = AspectCritic(
    name="professionalism",
    definition="回答是否使用专业、正式的语言?"
)

toxicity = AspectCritic(
    name="toxicity",
    definition="回答是否包含有毒或冒犯性的内容?"
)
```

---

### 9.2 Summarization Score（摘要质量评分）

> **指标说明**：Summarization Score 评估文本摘要的质量，综合考虑信息保留和简洁性。

**评估流程**：

```mermaid
flowchart TD
    Input["输入:<br/>reference_contexts, response (摘要)"] --> Step1["1. 提取关键短语<br/>人名、地名、机构、日期等"]

    Step1 --> Keyphrases["关键短语:<br/>['Apple', 'Steve Jobs', '1976', ...]"]

    Keyphrases --> Step2["2. 生成封闭式问题<br/>基于关键短语"]

    Step2 --> Questions["问题:<br/>['Apple是谁创立的?', ...]"]

    Questions --> Step3["3. 检查摘要能否回答这些问题"]

    Step3 --> Answers["答案:<br/>[1, 1, 0, 1, ...]"]

    Answers --> Step4["4. 计算 QA 分数"]

    Step4 --> QA["QA Score = 能回答的问题数 / 总问题数"]

    QA --> Step5["5. 计算简洁性分数"]

    Step5 --> Conciseness["Conciseness = 1 - len(summary)/len(original)"]

    Conciseness --> Final["Summarization Score = QA×0.5 + Conciseness×0.5"]

    style Input fill:#e1f5fe
    style Final fill:#e8f5e9
```

---

### 9.3 非 LLM 指标

> **指标说明**：Ragas 还提供了一系列不需要调用 LLM 的传统指标，计算速度更快，成本更低。

| 指标 | 说明 | 适用场景 |
|------|------|----------|
| ExactMatch | 精确字符串匹配 | 简单答案验证 |
| BleuScore | BLEU 评分 | 翻译质量评估 |
| RougeScore | ROUGE 评分 | 摘要质量评估 |
| ChrfScore | chrF 评分 | 字符级 F 值 |
| StringPresence | 字符串存在性检查 | 关键词检查 |
| NonLLMStringSimilarity | 字符串相似度 | 近似匹配 |

---

## 10. 指标选择指南

### 10.1 RAG 场景指标选择

> **选择指南说明**：根据 RAG 系统的不同评估需求，选择合适的指标组合。

```mermaid
graph TB
    RAG["RAG 评估需求"] --> Retrieval["检索质量评估"]
    RAG --> Generation["生成质量评估"]
    RAG --> EndToEnd["端到端评估"]

    Retrieval --> R1["Context Precision<br/>检索内容是否相关"]
    Retrieval --> R2["Context Recall<br/>是否检索完整"]
    Retrieval --> R3["Context Utilization<br/>检索效率"]

    Generation --> G1["Faithfulness<br/>是否忠于上下文"]
    Generation --> G2["Answer Relevancy<br/>是否切题"]
    Generation --> G3["Answer Correctness<br/>是否正确"]
    Generation --> G4["Factual Correctness<br/>事实正确性"]

    EndToEnd --> E1["Answer Correctness"]
    EndToEnd --> E2["Factual Correctness"]
    EndToEnd --> E3["Aspect Critic<br/>自定义标准"]

    style RAG fill:#e3f2fd
    style Retrieval fill:#e8f5e9
    style Generation fill:#f3e5f5
    style EndToEnd fill:#fff3e0
```

### 10.2 Agent 场景指标选择

> **Agent 指标选择说明**：Agent 评估需要综合考虑工具调用、目标达成和对话质量。

| 评估重点 | 推荐指标 | 需要的数据 |
|----------|----------|------------|
| 工具使用正确性 | Tool Call Accuracy | reference_tool_calls |
| 任务完成度 | Agent Goal Accuracy | reference (期望结果) |
| 主题遵循 | Topic Adherence | reference_topics |
| 回答质量 | Faithfulness | retrieved_contexts |
| 安全性 | AspectCritic | 自定义评判标准 |

### 10.3 指标依赖关系

> **依赖关系说明**：某些指标需要特定的输入字段，选择指标时需确保数据集包含所需字段。

```mermaid
graph LR
    subgraph Required["必需字段"]
        UI["user_input"]
        Res["response"]
        Ref["reference"]
        RC["retrieved_contexts"]
        RTC["reference_tool_calls"]
    end

    subgraph Metrics["指标"]
        F["Faithfulness"]
        AR["Answer Relevancy"]
        CP["Context Precision"]
        CR["Context Recall"]
        AC["Answer Correctness"]
        TCA["Tool Call Accuracy"]
    end

    UI --> F
    Res --> F
    RC --> F

    UI --> AR
    Res --> AR

    UI --> CP
    RC --> CP
    Ref --> CP

    UI --> CR
    RC --> CR
    Ref --> CR

    UI --> AC
    Res --> AC
    Ref --> AC

    UI --> TCA
    RTC --> TCA

    style Required fill:#e8f5e9
    style Metrics fill:#e3f2fd
```

---

## 11. 关键组件索引

### 11.1 核心文件

| 组件 | 路径 | 描述 |
|------|------|------|
| Dataset Schema | `src/ragas/dataset_schema.py` | 定义 SingleTurnSample, MultiTurnSample |
| Evaluation | `src/ragas/evaluation.py` | evaluate() 函数实现 |
| Messages | `src/ragas/messages.py` | HumanMessage, AIMessage, ToolCall |

### 11.2 RAG 指标

| 指标 | 路径 | 描述 |
|------|------|------|
| Faithfulness | `src/ragas/metrics/_faithfulness.py` | 忠实度评估 |
| Answer Relevancy | `src/ragas/metrics/_answer_relevance.py` | 答案相关性 |
| Context Precision | `src/ragas/metrics/_context_precision.py` | 上下文精度 |
| Context Recall | `src/ragas/metrics/_context_recall.py` | 上下文召回 |
| Answer Correctness | `src/ragas/metrics/_answer_correctness.py` | 答案正确性 |
| Factual Correctness | `src/ragas/metrics/_factual_correctness.py` | 事实正确性 |

### 11.3 Agent 指标

| 指标 | 路径 | 描述 |
|------|------|------|
| Tool Call Accuracy | `src/ragas/metrics/_tool_call_accuracy.py` | 工具调用准确度 |
| Agent Goal Accuracy | `src/ragas/metrics/_goal_accuracy.py` | 目标达成度 |
| Topic Adherence | `src/ragas/metrics/_topic_adherence.py` | 主题遵循度 |

### 11.4 通用指标

| 指标 | 路径 | 描述 |
|------|------|------|
| Aspect Critic | `src/ragas/metrics/_aspect_critic.py` | 自定义方面评判 |
| Summarization | `src/ragas/metrics/_summarization.py` | 摘要质量 |
| Noise Sensitivity | `src/ragas/metrics/_noise_sensitivity.py` | 噪声敏感度 |

---

## 12. 使用示例

### 12.1 RAG 评估示例

```python
from ragas.dataset_schema import SingleTurnSample
from ragas.metrics.collections import Faithfulness, AnswerRelevancy

# 准备数据
sample = SingleTurnSample(
    user_input="量子计算有什么优势?",
    retrieved_contexts=[
        "量子计算利用量子比特进行计算...",
        "与传统计算相比，量子计算在某些问题上具有指数级加速能力..."
    ],
    response="量子计算可以在特定问题上实现指数级加速...",
    reference="量子计算具有指数级加速优势..."
)

# 评估
faithfulness = Faithfulness()
relevancy = AnswerRelevancy()

faith_score = await faithfulness.ascore(sample)
relevancy_score = await relevancy.ascore(sample)
```

### 12.2 Agent 评估示例

```python
from ragas.dataset_schema import MultiTurnSample
from ragas.messages import HumanMessage, AIMessage, ToolMessage, ToolCall
from ragas.metrics.collections import ToolCallAccuracy, AgentGoalAccuracyWithReference

# 准备对话数据
sample = MultiTurnSample(
    user_input=[
        HumanMessage(content="帮我预订中餐厅"),
        AIMessage(content="", tool_calls=[
            ToolCall(name="restaurant_search", args={"cuisine": "Chinese"})
        ]),
        ToolMessage(content="找到: 金龙, 玉宫"),
        AIMessage(content="找到以下选择..."),
        HumanMessage(content="选金龙"),
        AIMessage(content="", tool_calls=[
            ToolCall(name="restaurant_book", args={"name": "金龙"})
        ]),
        ToolMessage(content="预订成功"),
        AIMessage(content="已预订金龙餐厅")
    ],
    reference_tool_calls=[
        ToolCall(name="restaurant_search", args={"cuisine": "Chinese"}),
        ToolCall(name="restaurant_book", args={"name": "金龙"})
    ],
    reference="成功预订金龙餐厅"
)

# 评估
tool_acc = ToolCallAccuracy()
goal_acc = AgentGoalAccuracyWithReference()

tool_score = await tool_acc.ascore(sample)
goal_score = await goal_acc.ascore(sample)
```

---

## 附录：完整指标速查表

| 指标名称 | 类型 | 必需字段 | 输出范围 | 说明 |
|----------|------|----------|----------|------|
| Faithfulness | RAG | user_input, response, retrieved_contexts | [0, 1] | 回答是否忠于上下文 |
| Answer Relevancy | RAG | user_input, response | [0, 1] | 回答是否切题 |
| Context Precision | RAG | user_input, retrieved_contexts, reference | [0, 1] | 检索精度 |
| Context Recall | RAG | user_input, retrieved_contexts, reference | [0, 1] | 检索召回 |
| Context Utilization | RAG | user_input, response, retrieved_contexts | [0, 1] | 上下文利用率 |
| Answer Correctness | RAG | user_input, response, reference | [0, 1] | 答案正确性 |
| Factual Correctness | RAG | response, reference | [0, 1] | 事实正确性 |
| Context Entity Recall | RAG | retrieved_contexts, reference_contexts | [0, 1] | 实体召回 |
| Noise Sensitivity | RAG | user_input, response, reference, retrieved_contexts | [0, 1] | 噪声敏感度 |
| Summarization Score | RAG | reference_contexts, response | [0, 1] | 摘要质量 |
| Tool Call Accuracy | Agent | user_input, reference_tool_calls | [0, 1] | 工具调用准确度 |
| Agent Goal Accuracy | Agent | user_input, reference | {0, 1} | 目标达成度 |
| Topic Adherence | Agent | user_input, reference_topics | [0, 1] | 主题遵循度 |
| Aspect Critic | General | 可配置 | {0, 1} | 自定义方面评判 |
| Bleu Score | NLP | response, reference | [0, 1] | 翻译质量 |
| Rouge Score | NLP | response, reference | [0, 1] | 摘要质量 |
| Exact Match | String | response, reference | {0, 1} | 精确匹配 |
