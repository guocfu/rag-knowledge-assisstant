# 基于RAG的企业级智能知识助手

项目链接：[基于RAG的企业级智能知识助手](https://github.com/guocfu/rag-knowledge-assisstant)

企业级 Agentic RAG 平台，覆盖从文档入库到智能问答的完整链路，支持多路并行检索、意图识别、模型路由容错、MCP 工具调用和会话记忆管理。

## 技术栈

| 层级 | 技术 |
|:---|:---|
| 后端框架 | Java 17, Spring Boot 3.5, MyBatis-Plus |
| 前端框架 | React 18, TypeScript, Vite |
| 关系型数据库 | PostgreSQL (pgvector 扩展) |
| 向量数据库 | Milvus 2.6 / pgvector |
| 缓存/限流 | Redis + Redisson |
| 对象存储 | S3 兼容 (RustFS) |
| 消息队列 | RocketMQ |
| 文档解析 | Apache Tika |
| 模型供应商 | 百炼 (阿里云) / SiliconFlow / Ollama (本地) |
| 认证鉴权 | Sa-Token |
| 代码格式化 | Spotless |

## 架构设计

采用前后端分离的架构模式，后端按职责分为四个 Maven 模块：

```
ragent/
├── framework/     # 业务无关的通用基础设施层
├── infra-ai/      # AI 模型抽象层（依赖 framework）
├── bootstrap/     # 业务逻辑层（依赖 framework + infra-ai）
└── mcp-server/    # 独立 MCP 协议服务
```

**依赖流向**: `bootstrap` → `infra-ai` → `framework`

分层不是为了炫技，而是解决实际问题：`framework` 层提供与业务无关的通用能力，`infra-ai` 层屏蔽不同模型供应商的差异，`bootstrap` 层专注业务逻辑。换模型供应商不用改业务代码，换业务逻辑不用动基础设施。

### 核心请求链路

一次用户提问在系统中经过的完整链路：

```
用户问题 → 查询重写 → 意图分类 → 多通道并行检索 → 后处理流水线 → 模型生成 → SSE 流式输出
```

## 核心功能

### 1. 多通道检索引擎

设计多路并行检索 + 后处理流水线架构：

- **意图定向检索**: 仅当知识库类意图被识别时激活，按意图节点并行检索对应集合
- **向量全局检索**: 当意图置信度低于阈值时作为补充，跨所有知识库并行检索
- **后处理器链**: 去重合并多通道结果 → Rerank 重排序精炼

每个通道独立执行、互不影响，通过线程池并行调度。后处理器按顺序串联，像流水线一样逐步精炼检索结果。

### 2. 树形意图识别

构建三层意图分类树（领域 → 分类 → 话题）：

- 基于 LLM 进行意图打分和置信度评估，temperature=0.1, topP=0.3 确保确定性
- 置信度不足时主动引导用户澄清
- 支持多子问题并行意图识别
- 公平配额算法控制意图数量上限，保证每个子问题至少获得一个意图

### 3. 模型路由与熔断降级

实现三态熔断器（CLOSED → OPEN → HALF_OPEN）：

- **CLOSED**: 正常状态，允许所有请求
- **OPEN**: 熔断状态，拒绝所有请求，冷却期后进入 HALF_OPEN
- **HALF_OPEN**: 探测状态，仅放行一个探测请求，成功恢复 CLOSED，失败重新 OPEN

基于 `ConcurrentHashMap.compute()` 实现原子状态转换，配合优先级降级链，一个模型挂了自动切到下一个候选，业务层无感知。

### 4. 首包探测机制

设计 `ProbeStreamBridge` 装饰器模式实现流式首包检测：

- 缓冲所有流式事件（onContent/onThinking/onComplete/onError）
- 使用 `CompletableFuture` 作为探测信号
- 60 秒超时等待首个数据包
- 成功则 `commit()` 回放所有缓冲事件
- 失败则取消当前流，切换到下一个模型候选

### 5. SSE 流式响应引擎

基于 Spring SseEmitter 实现结构化流式推送：

- 封装线程安全的 `SseEmitterSender`，CAS 原子关闭防止重复
- 设计 6 种事件类型：META / MESSAGE / FINISH / DONE / CANCEL / REJECT
- `StreamChatEventHandler` 支持分块消息投递和思考/回答分离展示
- `StreamTaskManager` 绑定 SSE 生命周期实现任务取消和断连自动清理

### 6. 分布式排队限流

基于 Redis 三件套实现分布式 FIFO 排队：

- **ZSET**: 作为 FIFO 等待队列，单调递增 score 保证公平排序
- **Lua 脚本**: 原子执行僵尸清理和出队判定，检测 entry marker 存活性
- **Pub/Sub**: 广播唤醒跨实例等待者，本地合并通知防惊群

请求先入 ZSET 排队，通过 Lua 脚本原子判断是否在队头窗口内再出队，信号量控制最大并发数。SSE 生命周期绑定实现客户端断连自动释放。

### 7. 文档入库 Pipeline

设计节点链式编排的入库流水线：

```
Fetcher → Parser → Enhancer → Enricher → Chunker → Indexer
```

- **Fetcher**: 支持本地文件、HTTP、S3、飞书四种来源
- **Parser**: 基于 Apache Tika 的多格式文档解析
- **Enhancer**: LLM 驱动的文档级增强（关键词提取、示例问题生成、元数据抽取）
- **Enricher**: LLM 驱动的块级增强（关键词、摘要、元数据）
- **Chunker**: 固定大小 / 结构感知两种分块策略
- **Indexer**: 向量化写入 Milvus/pgvector

每个节点支持条件执行，引擎自动检测环引用和缺失节点。

### 8. 会话记忆管理

实现滑动窗口 + 异步摘要压缩机制：

- **滑动窗口**: 保留最近 N 轮对话，超出部分自动压缩
- **增量摘要**: 仅处理新消息区间，LLM 合并新旧摘要并设置书签实现断点续传
- **分布式锁**: Redisson 保证多实例下摘要不重复
- **并行加载**: 摘要和历史记录并行查询，结果合并注入上下文

### 9. MCP 工具集成

基于 MCP 协议实现工具自动发现与注册：

- **本地工具**: Spring Bean 自动注册到 `DefaultMcpToolRegistry`
- **远程工具**: 通过 MCP SDK 连接远程 Server，`tools/list` 自动发现
- **参数提取**: LLM 驱动的参数提取将自然语言问题转换为结构化工具调用参数
- **执行器**: 统一的 `McpToolExecutor` 接口封装本地和远程两种执行方式

### 10. 全链路可观测性

基于 AOP 实现树结构分布式追踪：

- `@RagTraceRoot` 标记请求入口，`@RagTraceNode` 标记子节点
- TTL 透传 Trace 上下文跨全部线程池不丢失
- 支持嵌套树结构，通过 `Deque<String>` 栈维护父子关系
- 每个节点记录耗时、输入输出、异常信息

### 11. 基础框架

构建独立 framework 模块，包含：

- **异常体系**: `AbstractException` → `ClientException` / `ServiceException` / `RemoteException`，三级错误码规范
- **分布式 ID**: Snowflake 算法，Redis Lua 脚本分配 workerId/dataCenterId
- **幂等注解**: `@IdempotentSubmit` (HTTP 提交去重) / `@IdempotentConsume` (MQ 消费去重)
- **用户上下文**: TTL-based `UserContext` 跨线程透传登录信息
- **统一响应体**: `Result<T>` + `BaseErrorCode` 错误码规范

## 设计模式

| 设计模式 | 应用场景 | 解决的问题 |
|:---|:---|:---|
| 策略模式 | SearchChannel, PostProcessor, MCPToolExecutor | 检索通道、后处理器、MCP 工具可插拔替换 |
| 工厂模式 | IntentTreeFactory, ChunkingStrategyFactory | 复杂对象的创建逻辑集中管理 |
| 注册表模式 | MCPToolRegistry, IntentNodeRegistry | 组件自动发现与注册，新增工具零配置 |
| 模板方法 | IngestionNode 基类, AbstractParallelRetriever | 统一执行流程，子类只关注核心逻辑 |
| 装饰器模式 | ProbeStreamBridge | 在不修改原有回调的前提下增加首包探测能力 |
| 责任链模式 | 后处理器链、模型降级链 | 多个处理步骤按顺序串联，灵活组合 |
| 观察者模式 | StreamCallback | 流式事件的异步通知 |
| AOP | @RagTraceNode, @IdempotentSubmit | 链路追踪和幂等逻辑与业务代码解耦 |

## 线程池配置

按工作负载特征配置了 8 个独立线程池，所有线程池都用 `TtlExecutors` 包装确保上下文透传：

| 线程池 | 用途 | 队列类型 | 拒绝策略 |
|:---|:---|:---|:---|
| mcpBatchExecutor | MCP 批量调用 | SynchronousQueue | CallerRuns |
| ragContextExecutor | 子问题并行检索 + MCP | SynchronousQueue | CallerRuns |
| ragRetrievalExecutor | 通道级并行检索 | SynchronousQueue | CallerRuns |
| innerRetrievalExecutor | 内部检索子任务 | LinkedBlocking(100) | CallerRuns |
| intentClassifyExecutor | 意图分类并行 | SynchronousQueue | CallerRuns |
| memorySummaryExecutor | 会话记忆摘要 | LinkedBlocking(200) | CallerRuns |
| modelStreamExecutor | 模型流式输出 | LinkedBlocking(200) | Abort |
| chatEntryExecutor | 对话入口 | SynchronousQueue | Abort |

## 快速启动

### 后端

```bash
# 构建整个项目
./mvnw clean install

# 构建单个模块
./mvnw clean install -pl bootstrap

# 运行测试
./mvnw test

# 运行单个测试类
./mvnw test -Dtest=IntentTreeServiceTests

# 跳过测试构建
./mvnw clean install -DskipTests

# 代码格式化（编译时自动执行）
./mvnw spotless:apply
```

### 前端

```bash
cd frontend
npm install
npm run dev      # 启动开发服务器
npm run build    # 生产构建
npm run lint     # 运行 ESLint
npm run format   # 运行 Prettier
```

## 配置说明

主配置文件: `bootstrap/src/main/resources/application.yaml`

关键配置项：

```yaml
rag:
  vector:
    type: pg                    # 向量存储后端 (pg / milvus)
  search:
    channels:
      vector-global:
        confidence-threshold: 0.6  # 意图置信度阈值
      intent-directed:
        min-intent-score: 0.4     # 最低意图分数
  memory:
    history-keep-turns: 4         # 保留历史轮数
    summary-start-turns: 5        # 触发摘要的轮数
  rate-limit:
    global:
      max-concurrent: 50          # 最大并发数

ai:
  chat:
    default-model: qwen3-max
    candidates:                   # 模型候选列表
      - id: qwen-plus
        provider: bailian
        priority: 1
  embedding:
    default-model: qwen-emb-8b
```

### 环境变量

```bash
BAILIAN_API_KEY=your_api_key      # 阿里云百炼 API Key
SILICONFLOW_API_KEY=your_api_key  # SiliconFlow API Key
```

## 扩展指南

### 新增检索通道

1. 实现 `SearchChannel` 接口
2. 注册为 Spring Bean (`@Component`)
3. 配置阈值 `application.yaml` → `rag.search.channels`

### 新增 MCP 工具

1. 实现 `McpToolExecutor` 接口
2. 注册为 Spring Bean — 自动被 `DefaultMcpToolRegistry` 发现

### 新增入库节点

1. 继承 `IngestionNode` 基类
2. 实现 `doExecute()` 方法
3. 注册到 Pipeline 配置

### 新增模型供应商

1. 在 `infra-ai` 层实现 `ChatClient` 接口
2. 配置候选列表即可参与路由

## 环境要求

- Java 17+
- Node.js 18+
- PostgreSQL (pgvector 扩展)
- Redis
- RocketMQ (可选)
- Milvus (可选)
- MinIO/S3 (可选)

## 许可证

[Apache License 2.0](./LICENSE)
