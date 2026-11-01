# Langfuse Observability 实现分析

## 1. 概述

Langfuse 的 Observability（可观测性）系统是一个专为 LLM 应用设计的追踪和监控系统，支持 OpenTelemetry 协议，提供 Trace、Observation、Score 等核心概念来记录和分析 LLM 应用的执行过程。

---

## 2. 核心概念

### 2.1 数据模型层次

> **数据模型说明**：Langfuse Observability 系统的数据模型采用三层结构：Trace（追踪）→ Observation（观察）→ Score（评分）。Trace 代表一次完整的用户交互，Observation 记录其中的各个步骤，Score 用于对结果进行量化评估。

```mermaid
graph TB
    subgraph Trace["🔍 TRACE"]
        direction TB
        TraceDesc["代表一次完整的用户交互或请求<br/>如：一次对话、一次 API 调用"]
        TraceFields["id, projectId, name, timestamp<br/>userId, sessionId, environment<br/>metadata, input, output, tags"]
    end

    Trace -->|1:N| Observation

    subgraph Obs["📊 OBSERVATION"]
        direction TB
        ObsDesc["代表 Trace 内的单个步骤或操作"]

        subgraph ObsTypes["Observation 类型"]
            direction LR
            Generation["GENERATION<br/>LLM 调用<br/>model, tokens, cost"]
            Span["SPAN<br/>一般操作<br/>startTime, endTime"]
            Event["EVENT<br/>离散事件"]
            Agent["AGENT<br/>Agent 执行"]
            Tool["TOOL<br/>工具调用"]
            Chain["CHAIN<br/>链式调用"]
            Retriever["RETRIEVER<br/>检索操作"]
            Embedding["EMBEDDING<br/>嵌入计算"]
            Guardrail["GUARDRAIL<br/>护栏检查"]
        end
    end

    Observation -->|1:N| Score

    subgraph ScoreObj["⭐ SCORE"]
        direction TB
        ScoreDesc["与 Trace 或 Observation 关联的评分/指标"]
        ScoreFields["id, projectId, traceId, observationId<br/>name, value, dataType, comment<br/>source, configId"]
        ScoreTypes["DataType: NUMERIC | CATEGORICAL | BOOLEAN | TEXT<br/>Source: ANNOTATION | API | EVAL | CONFIG"]
    end
```

**核心概念解释**：
- **Trace**：最高级别的抽象，代表一次完整的请求或对话流程
- **Observation**：Trace 内部的原子操作，支持多种类型以适应不同场景
- **Score**：可附加到 Trace 或 Observation 上的量化评估结果

### 2.2 Observation 层级关系

> **层级关系说明**：Observation 通过 `parentObservationId` 字段构建父子关系，形成树状结构。顶层 Observation 的 `parentObservationId` 为 null，表示直接属于 Trace。

```mermaid
graph TD
    Trace["TRACE<br/>parentObservationId = null"] --> L1_1

    subgraph Level1["Level 1 Observations<br/>parentObservationId = null"]
        L1_1["SPAN: process_data"]
        L1_2["GENERATION: gpt-4-call"]
        L1_3["AGENT: researcher"]
    end

    L1_1 --> L2_1
    L1_2 --> L2_2
    L1_3 --> L2_3

    subgraph Level2["Level 2 Observations<br/>通过 parentObservationId 关联"]
        L2_1["TOOL: web_search"]
        L2_2["RETRIEVER: rag_retrieve"]
        L2_3["EMBEDDING: embed_query"]
    end

    L2_1 --> L3_1

    subgraph Level3["Level 3 Observations"]
        L3_1["GENERATION: summarize"]
    end

    style Trace fill:#e1f5fe
    style Level1 fill:#f3e5f5
    style Level2 fill:#e8f5e9
    style Level3 fill:#fff3e0
```

### 2.3 Observation 字段详解

> **字段说明**：Observation 包含通用字段和 GENERATION 特有字段。通用字段适用于所有类型，GENERATION 特有字段用于记录 LLM 调用的详细信息。

```mermaid
classDiagram
    class Observation {
        +String id
        +String traceId
        +String parentObservationId
        +String projectId
        +String type
        +String name
        +DateTime startTime
        +DateTime endTime
        +JSON metadata
        +JSON input
        +JSON output
        +String level
        +String statusMessage
        +String version
        +String environment
    }

    class Generation {
        +String model
        +JSON modelParameters
        +Int promptTokens
        +Int completionTokens
        +Int totalTokens
        +DateTime completionStartTime
        +String promptName
        +String promptVersion
        +Decimal calculatedTotalCost
        +Decimal calculatedInputCost
        +Decimal calculatedOutputCost
    }

    Observation <|-- Generation : 继承
```

**Level 枚举值**：
| Level | 说明 |
|-------|------|
| DEFAULT | 默认级别 |
| DEBUG | 调试信息 |
| WARNING | 警告 |
| ERROR | 错误 |

---

## 3. 系统架构

### 3.1 整体架构图

> **架构说明**：Langfuse Observability 系统采用典型的分层架构，从客户端 SDK 到存储层共五层。数据从客户端流入，经过 API 层验证，通过队列异步处理，最终持久化到 ClickHouse。

```mermaid
graph TB
    subgraph Client["🔗 客户端 SDK"]
        direction LR
        PythonSDK["Python SDK<br/>@langfuse"]
        JSSDK["JS/TS SDK<br/>langfuse"]
        OTel["OpenTelemetry<br/>OTLP Exporter"]
        REST["REST API<br/>POST /ingest"]
        Langchain["Langchain<br/>Integration"]
    end

    subgraph API["🌐 Web API Layer (Next.js)"]
        IngestAPI["/api/public/ingestion<br/>POST - 批量摄入<br/>认证: API Key"]
        OTelAPI["/api/public/otel/v1/traces<br/>OpenTelemetry OTLP<br/>Protobuf / JSON"]
        CRUDAPI["/api/public/traces<br/>/api/public/observations<br/>/api/public/scores<br/>GET / POST / PATCH / DELETE"]
    end

    subgraph Shared["📦 Shared Package Layer"]
        ProcessBatch["processEventBatch.ts<br/>- 验证事件 schema<br/>- 写入 S3<br/>- 入队 IngestionQueue"]
        EventTypes["types.ts<br/>- 事件类型定义<br/>- Schema 验证 (Zod)"]
        CHSchema["clickhouse/schema.ts<br/>- CH 表定义<br/>- 实体类型映射"]
    end

    subgraph Queue["📨 Queue Layer (BullMQ + Redis)"]
        IngestionQ["IngestionQueue<br/>处理原始事件<br/>写入 ClickHouse"]
        TraceUpsertQ["TraceUpsertQueue<br/>触发下游处理<br/>创建评估任务"]
        SecondaryQ["SecondaryIngestionQueue<br/>处理 S3 限流<br/>负载均衡"]
        OTelQ["OtelIngestionQueue<br/>OTLP 处理<br/>格式转换"]
        TraceBatchQ["TraceBatchQueue<br/>批量 Trace 写入"]
    end

    subgraph Worker["⚙️ Worker Layer"]
        IngestionService["IngestionService<br/>- mergeAndWrite()<br/>- processTraceEvent()<br/>- processObservationEvent()<br/>- processScoreEvent()"]
        CHWriter["ClickhouseWriter<br/>- 批量写入<br/>- 连接池<br/>- 错误重试"]
    end

    subgraph Storage["💾 Storage Layer"]
        PostgreSQL["PostgreSQL<br/>配置和元数据<br/>Trace / TraceSession<br/>ScoreConfig"]
        ClickHouse["ClickHouse<br/>高容量分析数据<br/>traces / observations<br/>scores / dataset_run_items"]
        S3["S3<br/>大对象存储<br/>事件数据 / 媒体文件"]
    end

    Client --> API
    API --> Shared
    Shared --> Queue
    Queue --> Worker
    Worker --> Storage
```

**各层职责**：
| 层 | 职责 | 技术栈 |
|----|------|--------|
| Client | 数据采集和发送 | Python/JS SDK, OTLP |
| API | 请求验证和路由 | Next.js, tRPC |
| Shared | 共享业务逻辑 | TypeScript, Zod |
| Queue | 异步任务处理 | BullMQ, Redis |
| Worker | 数据转换和写入 | Node.js |
| Storage | 持久化存储 | PostgreSQL, ClickHouse, S3 |

---

## 4. 数据摄入流程

### 4.1 Ingestion 流程

> **流程说明**：数据摄入是 Observability 系统的核心流程。事件从客户端到达后，经过验证、S3 存储、队列处理，最终写入 ClickHouse。整个流程采用异步设计，支持高并发和容错。

```mermaid
flowchart TD
    Start([客户端 SDK 调用]) --> Auth["1. 认证与验证"]

    subgraph Step1["认证与验证"]
        A1["验证 API Key<br/>(publicKey:secretKey)"]
        A2["检查项目权限"]
        A3["验证请求 schema (Zod)"]
        A4["应用速率限制"]
    end

    Auth --> Step1 --> Process

    subgraph Process["2. 事件处理 (processEventBatch.ts)"]
        Parse["a. 解析事件<br/>提取事件类型<br/>验证 body schema<br/>应用采样"]
        WriteS3["b. 写入 S3<br/>生成唯一 fileKey<br/>构建 bucket prefix<br/>上传 JSON 文件"]
        Enqueue["c. 入队 IngestionQueue<br/>添加 job payload<br/>应用延迟"]
        Parse --> WriteS3 --> Enqueue
    end

    Process --> QueueProcess

    subgraph QueueProcess["3. 队列处理 (ingestionQueue.ts)"]
        Dedup["a. Redis 去重检查<br/>Key: {projectId}:{type}:{entityId}:{fileKey}"]
        Download["b. 从 S3 下载事件<br/>List + Download + Parse"]
        Merge["c. IngestionService.mergeAndWrite()<br/>按 entityId 分组<br/>合并更新<br/>模型匹配<br/>写入 ClickHouse"]
        PostProcess["d. 后处理<br/>标记已处理<br/>入队 TraceUpsert<br/>记录指标"]
        Dedup --> Download --> Merge --> PostProcess
    end

    QueueProcess --> CHWrite

    subgraph CHWrite["4. ClickHouse 写入"]
        Batch["批量插入<br/>缓冲记录<br/>达到阈值或超时后刷新<br/>INSERT INTO traces/observations/scores"]
    end

    CHWrite --> End([完成])
```

**关键设计要点**：
- **S3 缓冲**：事件先写入 S3，再异步处理，解耦 API 响应和数据处理
- **去重机制**：Redis 缓存防止重复处理
- **批量写入**：ClickHouse 使用批量插入提高写入性能
- **延迟处理**：UTC 日期边界附近添加延迟，避免日期重复

---

### 4.2 OpenTelemetry 摄入流程

> **OTLP 流程说明**：Langfuse 支持标准的 OpenTelemetry 协议，可以将 OTLP 格式的数据转换为 Langfuse 格式并存储。支持 Protobuf 和 JSON 两种编码格式。

```mermaid
flowchart TD
    Start([OTLP Exporter SDK]) -->|POST /api/public/otel/v1/traces| OTLPRequest

    OTLPRequest["1. OTLP 请求处理"] --> Parse

    subgraph Parse["请求解析"]
        P1["解析 Protobuf/JSON payload"]
        P2["提取 ResourceSpans → ScopeSpans → Spans"]
        P3["验证 OTLP schema"]
    end

    Parse --> Convert

    subgraph Convert["2. OTLP → Langfuse 转换"]
        subgraph Resource["Resource Attributes → Trace"]
            R1["service.name → trace.name"]
            R2["service.version → trace.version"]
            R3["deployment.environment → trace.environment"]
            R4["Custom attributes → trace.metadata"]
        end

        subgraph Span["Span → Observation"]
            SP1["span.traceId → observation.traceId"]
            SP2["span.spanId → observation.id"]
            SP3["span.parentSpanId → observation.parentObservationId"]
            SP4["span.name → observation.name"]
            SP5["span.startTime → observation.startTime"]
            SP6["span.endTime → observation.endTime"]
            SP7["span.kind → observation.type"]
        end

        subgraph LLM["GenAI 语义属性"]
            G1["gen_ai.system → model provider"]
            G2["gen_ai.request.model → observation.model"]
            G3["gen_ai.request.temperature → modelParameters"]
            G4["gen_ai.usage.input_tokens → promptTokens"]
            G5["gen_ai.usage.output_tokens → completionTokens"]
        end
    end

    Convert --> Enqueue["3. 入队 OtelIngestionQueue"] --> SameProcess["同原生摄入处理<br/>写入 S3 → IngestionService → ClickHouse"]
```

**Span Kind 映射**：
| OTel Span Kind | Langfuse Observation Type |
|----------------|---------------------------|
| CLIENT / SERVER (含 LLM 属性) | GENERATION |
| INTERNAL | SPAN |
| PRODUCER / CONSUMER | EVENT |

---

## 5. 查询架构

### 5.1 数据查询流程

> **查询说明**：Langfuse 使用 ClickHouse 作为主要分析数据库，支持高性能的时间序列查询。查询优化包括分区裁剪、索引优化和物化视图。

```mermaid
flowchart TD
    Start([UI / API 请求]) --> Plan["查询规划"]

    subgraph Plan["查询规划"]
        Source{"确定数据源"}
        Source -->|近期数据 < 30天| CH["ClickHouse"]
        Source -->|历史数据| PG["PostgreSQL fallback"]
        Source -->|聚合查询| MV["物化视图"]
    end

    Plan --> Execute

    subgraph Execute["ClickHouse 查询执行"]
        Tables["查询表:<br/>• traces: 追踪数据<br/>• observations: 观察数据<br/>• scores: 评分数据<br/>• dataset_run_items: 数据集运行项"]

        Optimizations["查询优化:<br/>• 按 project_id, date 分区<br/>• ORDER BY (project_id, timestamp DESC)<br/>• metadata/tags 跳过索引<br/>• 预聚合物化视图"]

        Tables --> Optimizations
    end

    Execute --> Result

    subgraph Result["结果处理"]
        R1["JSON 字段解析<br/>(metadata, input, output)"]
        R2["时间戳转换为 UTC"]
        R3["Score 值类型转换"]
        R4["分页处理"]
    end

    Result --> Response([返回响应])
```

### 5.2 ClickHouse Schema

> **Schema 设计说明**：ClickHouse 表使用 ReplacingMergeTree 引擎，支持最终一致性。按月份分区，按 (project_id, timestamp DESC, id) 排序，优化时间序列查询。

```mermaid
erDiagram
    TRACES {
        string id
        datetime timestamp
        string name
        string project_id
        string user_id
        string session_id
        string environment
        json metadata
        string input
        string output
        array tags
        boolean bookmarked
        boolean public
        datetime event_ts
        uint8 is_deleted
    }

    OBSERVATIONS {
        string id
        string trace_id
        string project_id
        string type
        string parent_observation_id
        datetime start_time
        datetime end_time
        string name
        json metadata
        string level
        string input
        string output
        string model
        string model_parameters
        uint64 prompt_tokens
        uint64 completion_tokens
        uint64 total_tokens
        datetime completion_start_time
        decimal calculated_total_cost
        decimal calculated_input_cost
        decimal calculated_output_cost
        datetime event_ts
        uint8 is_deleted
    }

    SCORES {
        string id
        string project_id
        string trace_id
        string observation_id
        datetime timestamp
        string name
        float64 value
        string string_value
        string data_type
        string comment
        string source
        string config_id
        datetime event_ts
        uint8 is_deleted
    }

    TRACES ||--o{ OBSERVATIONS : "1:N"
    TRACES ||--o{ SCORES : "1:N"
    OBSERVATIONS ||--o{ SCORES : "1:N"
```

**表引擎配置**：
```sql
ENGINE = ReplacingMergeTree(event_ts, is_deleted)
PARTITION BY toYYYYMM(timestamp)
ORDER BY (project_id, timestamp DESC, id)
SETTINGS index_granularity = 8192;
```

---

## 6. 事件类型

### 6.1 Ingestion Event Types

> **事件类型说明**：Langfuse 定义了多种事件类型，按实体分为 Trace 事件、Observation 事件、Score 事件和 Dataset 事件。每种事件都有 create 和 update 两种操作。

```mermaid
graph TB
    EventTypes["eventTypes 枚举"] --> Categories

    subgraph Categories["事件分类"]
        TraceEvents["Trace 事件"]
        ObsEvents["Observation 事件"]
        ScoreEvents["Score 事件"]
        DatasetEvents["Dataset 事件"]
    end

    subgraph TraceDetail["Trace 事件"]
        TC["trace-create<br/>创建新追踪"]
        TU["trace-update<br/>更新追踪"]
    end

    subgraph ObsDetail["Observation 事件"]
        GC["generation-create/update<br/>LLM 调用"]
        SC["span-create/update<br/>Span 操作"]
        EC["event-create<br/>离散事件"]
        AC["agent/tool/chain-create<br/>其他类型"]
    end

    subgraph ScoreDetail["Score 事件"]
        SCr["score-create<br/>创建评分"]
        SU["score-update<br/>更新评分"]
    end

    subgraph DatasetDetail["Dataset 事件"]
        DRC["dataset-run-item-create<br/>关联数据集运行项"]
    end

    TraceEvents --> TraceDetail
    ObsEvents --> ObsDetail
    ScoreEvents --> ScoreDetail
    DatasetEvents --> DatasetDetail
```

**事件处理顺序**：
1. 非 update 事件优先处理
2. update 事件按时间戳排序
3. 相同 entityId 的事件合并处理

---

## 7. 成本计算

### 7.1 Model Matching & Cost Calculation

> **成本计算说明**：GENERATION 类型的 Observation 支持自动成本计算。系统根据模型名称查找定价信息，结合 token 使用量计算输入/输出成本。

```mermaid
flowchart TD
    Start([Generation Event<br/>model + tokens]) --> Lookup["1. 模型查找"]

    subgraph Lookup["模型查找"]
        L1["findModel(model, projectId)"]
        L2["搜索项目配置的模型"]
        L3["回退到全局模型注册表"]
        L4["按模型名称模式匹配"]
    end

    Lookup --> Pricing

    subgraph Pricing["2. 定价解析"]
        P1["根据模型获取定价层级"]
        P2["获取:<br/>• inputPrice (每 1K tokens)<br/>• outputPrice (每 1K tokens)<br/>• totalPrice (如果仅有总价)"]
    end

    Pricing --> Calculate

    subgraph Calculate["3. 成本计算"]
        Calc1{"定价类型?"}
        Calc1 -->|有 input/output 价格| Formula1["inputCost = (promptTokens / 1000) × inputPrice<br/>outputCost = (completionTokens / 1000) × outputPrice<br/>totalCost = inputCost + outputCost"]
        Calc1 -->|仅有 total 价格| Formula2["totalCost = (totalTokens / 1000) × totalPrice"]
    end

    Calculate --> Write["4. 写入 ClickHouse<br/>calculated_total_cost<br/>calculated_input_cost<br/>calculated_output_cost"]
```

**Token 计算策略**：
| 模型类型 | Tokenizer |
|----------|-----------|
| OpenAI 模型 | tiktoken |
| 其他已知模型 | 模型特定 tokenizer |
| 未知模型 | 字符估算 |

---

## 8. Trace 处理与评估触发

### 8.1 Trace Upsert 流程

> **Trace Upsert 说明**：当 Trace 数据写入完成后，系统会触发 TraceUpsert 事件，用于启动下游处理流程，包括评估任务创建和自动化操作。

```mermaid
flowchart TD
    Start([Ingestion 完成]) --> Enqueue["入队 TraceUpsert"]

    Enqueue --> Payload["Payload:<br/>{ projectId, traceId,<br/>  exactTimestamp, traceEnvironment }"]

    Payload --> Processor["TraceUpsert Queue Processor"]

    subgraph Processor["处理器逻辑"]
        CheckEval["1. 检查评估配置<br/>hasNoEvalConfigsCache 检查<br/>无评估器则跳过"]
        CreateEval["2. 创建评估任务<br/>createEvalJobs()<br/>匹配规则 → 采样 → 入队"]
        Downstream["3. 触发下游处理<br/>Topics 自动化<br/>Webhook 投递<br/>分析集成"]
        CheckEval --> CreateEval --> Downstream
    end

    Processor --> End([完成])
```

---

## 9. 关键组件索引

### 9.1 Web API 层

| 组件 | 路径 | 描述 |
|------|------|------|
| Ingestion API | `web/src/pages/api/public/ingestion.ts` | 批量摄入端点 |
| OTLP Endpoint | `web/src/pages/api/public/otel/` | OpenTelemetry 接收 |
| Traces API | `web/src/pages/api/public/traces/` | Trace CRUD |
| Observations API | `web/src/pages/api/public/v2/observations/` | Observation API |
| Scores API | `web/src/pages/api/public/scores/` | Score API |

### 9.2 Worker 层

| 组件 | 路径 | 描述 |
|------|------|------|
| Ingestion Queue | `worker/src/queues/ingestionQueue.ts` | 摄入队列处理 |
| OTel Queue | `worker/src/queues/otelIngestionQueue.ts` | OTLP 处理 |
| Trace Batch Queue | `worker/src/queues/traceBatchQueue.ts` | 批量写入 |
| IngestionService | `worker/src/services/IngestionService/index.ts` | 核心摄入逻辑 |
| ClickhouseWriter | `worker/src/services/ClickhouseWriter/` | ClickHouse 写入 |

### 9.3 Shared 层

| 组件 | 路径 | 描述 |
|------|------|------|
| Event Types | `packages/shared/src/server/ingestion/types.ts` | 事件类型定义 |
| Process Batch | `packages/shared/src/server/ingestion/processEventBatch.ts` | 批量处理 |
| Model Match | `packages/shared/src/server/ingestion/modelMatch.ts` | 模型匹配 |
| CH Schema | `packages/shared/src/server/clickhouse/schema.ts` | CH 表定义 |
| Queue Types | `packages/shared/src/server/queues.ts` | 队列类型 |

### 9.4 UI 层

| 组件 | 路径 | 描述 |
|------|------|------|
| Traces Page | `web/src/features/traces/` | Trace 列表和详情 |
| Trace Table | `web/src/features/traces/TracesTable.tsx` | Trace 表格组件 |
| Trace Review | `web/src/features/traces/TracePage.tsx` | Trace 审阅页 |
| Observations | `web/src/features/tracing-tables/observations/` | Observation 展示 |
| Scores | `web/src/features/scores/` | Score 相关组件 |

---

## 10. 性能优化

### 10.1 写入优化

> **写入优化说明**：Langfuse 采用多种策略优化写入性能，包括 S3 批量上传、Redis 去重、ClickHouse 批量写入和队列分区。

```mermaid
graph LR
    subgraph S3Opt["S3 事件缓冲"]
        S3_1["按 entityId 分组"]
        S3_2["单次 S3 PUT 批量上传"]
        S3_3["减少 S3 API 调用"]
    end

    subgraph RedisOpt["Redis 去重"]
        R1["已处理事件缓存"]
        R2["Key: {projectId}:{type}:{entityId}:{fileKey}"]
        R3["TTL: 24 小时"]
        R4["减少重复写入"]
    end

    subgraph CHOpt["ClickHouse 批量写入"]
        C1["内存缓冲记录"]
        C2["达到 batch size (1000) 或 timeout (5s) 刷新"]
        C3["单次 INSERT 刷新"]
        C4["减少连接开销"]
    end

    subgraph QueueOpt["队列分区"]
        Q1["高容量项目使用二级队列"]
        Q2["按 projectId 分片"]
        Q3["防止单项目拥堵"]
        Q4["独立扩展"]
    end
```

### 10.2 查询优化

> **查询优化说明**：ClickHouse 查询通过分区裁剪、排序索引、物化视图和缓存机制进行优化。

```mermaid
graph TB
    subgraph PartitionOpt["分区裁剪"]
        P1["按月份分区 (toYYYYMM)"]
        P2["日期范围查询跳过无关分区"]
        P3["显著减少扫描数据量"]
    end

    subgraph OrderOpt["ORDER BY 优化"]
        O1["(project_id, timestamp DESC, id)"]
        O2["时间序列查询使用索引前缀"]
        O3["项目隔离在索引中"]
        O4["高效分页"]
    end

    subgraph MV["物化视图"]
        M1["预聚合指标用于仪表板"]
        M2["日/小时级汇总"]
        M3["减少查询延迟"]
    end

    subgraph Cache["读取跳过缓存"]
        CA1["缓存空查询结果"]
        CA2["减少空项目的 ClickHouse 负载"]
        CA3["TTL 失效"]
    end
```

---

## 11. 监控与指标

> **监控指标说明**：Observability 系统提供完整的监控指标，涵盖摄入、队列、ClickHouse 和缓存。

```mermaid
mindmap
  root((Observability 监控指标))
    Ingestion
      langfuse.ingestion.count
      langfuse.ingestion.batch_size
      langfuse.ingestion.s3_file_size_bytes
      langfuse.ingestion.duration
      langfuse.ingestion.error_count
    Queue
      langfuse.queue.ingestion.waiting
      langfuse.queue.ingestion.active
      langfuse.queue.ingestion.delayed
      langfuse.queue.ingestion.failed
    ClickHouse
      langfuse.clickhouse.write.count
      langfuse.clickhouse.write.duration
      langfuse.clickhouse.write.batch_size
      langfuse.clickhouse.query.duration
    Cache
      langfuse.ingestion.recently_processed_cache
      langfuse.clickhouse.read_skip_cache
```

---

## 12. 环境变量配置

| 变量 | 描述 | 默认值 |
|------|------|--------|
| `DATABASE_URL` | PostgreSQL 连接 | - |
| `CLICKHOUSE_URL` | ClickHouse 连接 | - |
| `LANGFUSE_S3_EVENT_UPLOAD_BUCKET` | S3 事件存储桶 | - |
| `LANGFUSE_REDIS_CONNECTION_STRING` | Redis 连接 | - |
| `LANGFUSE_INGESTION_QUEUE_DELAY_MS` | 摄入队列延迟 | 5000 |
| `LANGFUSE_ENABLE_REDIS_SEEN_EVENT_CACHE` | 启用去重缓存 | true |
| `LANGFUSE_SECONDARY_INGESTION_QUEUE_ENABLED_PROJECT_IDS` | 二级队列项目 | - |
| `LANGFUSE_S3_EVENT_KEY_MAX_SEGMENT_BYTES` | S3 key 最大段大小 | - |

---

## 附录：完整数据摄入时序图

```mermaid
sequenceDiagram
    participant Client as 客户端 SDK
    participant API as API Layer
    participant S3 as S3 Storage
    participant Redis as Redis
    participant Queue as BullMQ Queue
    participant Worker as Worker
    participant CH as ClickHouse

    Client->>API: POST /api/public/ingestion
    API->>API: 认证 + 验证 schema
    API->>S3: 上传事件 JSON
    API->>Queue: Enqueue IngestionJob
    API-->>Client: 201 Created

    Queue->>Worker: Process IngestionJob
    Worker->>Redis: 检查去重缓存
    alt 已处理
        Worker-->>Queue: Skip (已处理)
    else 未处理
        Worker->>S3: 下载事件文件
        S3-->>Worker: 事件数据
        Worker->>Worker: mergeAndWrite()
        Worker->>Worker: 模型匹配 + 成本计算
        Worker->>CH: 批量写入
        Worker->>Redis: 标记已处理
        Worker->>Queue: Enqueue TraceUpsert
    end

    Queue->>Worker: Process TraceUpsert
    Worker->>Worker: createEvalJobs()
    Worker->>Queue: Enqueue EvalExecution
```
