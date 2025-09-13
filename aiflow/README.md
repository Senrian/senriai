# Sen AI Flow

一个现代化的Java AI应用框架，基于Spring Boot和Project Reactor构建，专为高性能、可扩展的AI应用而设计。

## 🚀 特性

- **响应式架构**: 基于Project Reactor的非阻塞响应式编程
- **组件化设计**: 高度模块化的组件系统，支持热插拔
- **流式处理**: 原生支持AI模型的实时流式输出
- **企业级监控**: 内置完整的监控、追踪、告警等企业级特性
- **AI优化**: 专门针对AI应用场景的性能优化
- **易于扩展**: 简洁的API设计，易于扩展和定制
- **RAG支持**: 内置检索增强生成系统
- **向量存储**: 支持多种向量数据库和嵌入模型
- **MCP集成**: 支持Model Context Protocol

## 📦 模块结构

```
sen-aiflow/
├── sen-aiflow-core/           # 核心模块 - 组件系统、注解、配置
├── sen-aiflow-engine/         # 执行引擎 - DAG引擎、管道管理、任务调度
├── sen-aiflow-client/         # 客户端 - 框架客户端API
├── sen-aiflow-service/        # 服务层 - LLM服务、AI服务集成
├── sen-aiflow-storage/        # 存储模块 - 通用仓储、数据持久化
├── sen-aiflow-embedding/      # 嵌入模块 - 向量嵌入、特征提取
├── sen-aiflow-rag/            # RAG模块 - 检索增强生成
├── sen-aiflow-monitor/        # 监控模块 - 指标收集、性能监控
├── sen-aiflow-reactive/       # 响应式模块 - 响应式上下文、流管理
├── sen-aiflow-stream/         # 流式处理 - 实时流处理管道
├── sen-aiflow-web/            # Web接口 - REST API、WebSocket
├── sen-aiflow-spring-boot-starter/  # Spring Boot Starter
├── sen-aiflow-mcp/            # MCP集成 - Model Context Protocol
└── examples/                  # 示例应用
```

## 🛠️ 快速开始

### 环境要求

- Java 17+
- Maven 3.8+
- Spring Boot 3.2+

### 1. 克隆项目

```bash
git clone https://github.com/senrian/sen-aiflow.git
cd sen-aiflow
```

### 2. 构建项目

```bash
# 使用构建脚本
./build.sh

# 或手动构建
mvn clean install
```

### 3. 添加依赖

```xml
<dependency>
    <groupId>com.senrian.ai</groupId>
    <artifactId>sen-aiflow-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 4. 启用Sen AI Flow

```java
@SpringBootApplication
@EnableSenAiFlow
public class MyAiApp {
    public static void main(String[] args) {
        SpringApplication.run(MyAiApp.class, args);
    }
}
```

### 5. 创建组件

```java
@Component
@SenAiFlowComponent(
    type = SenAiFlowComponent.ComponentType.PROCESSOR,
    description = "文本处理器"
)
public class TextProcessor extends AbstractComponent implements Processor<String, String> {
    
    public TextProcessor() {
        super("TextProcessor", "文本处理器", "1.0.0");
    }
    
    @Override
    public Mono<String> process(String input) {
        return Mono.just(input.toUpperCase());
    }
    
    @Override
    public Class<String> getInputType() {
        return String.class;
    }
    
    @Override
    public Class<String> getOutputType() {
        return String.class;
    }
}
```

### 6. 配置应用

```yaml
# application.yml
sen-aiflow:
  monitoring:
    enabled: true
  reactive:
    enabled: true
  streaming:
    enabled: true
    buffer-size: 1000
  storage:
    type: memory
    max-size: 1000
  embedding:
    provider: openai
    model: text-embedding-ada-002
  rag:
    enabled: true
    max-retrieval: 10
```

## 🎯 核心概念

### 组件系统

Sen AI Flow采用组件化架构，支持以下类型的组件：

- **Processor**: 数据处理器
- **Generator**: 内容生成器
- **Retriever**: 信息检索器
- **Embedding**: 向量嵌入器
- **Storage**: 数据存储器
- **Monitor**: 监控组件
- **Pipeline**: 工作流管道
- **Adaptor**: 外部系统适配器

### 响应式编程

基于Project Reactor的响应式编程模型：

```java
// 流式处理
Flux<String> input = Flux.just("hello", "world");
input.transform(processor::processStream)
     .subscribe(result -> System.out.println(result));

// 异步处理
Mono<String> result = processor.process("input");
result.subscribe(System.out::println);
```

### 流式处理

支持实时流式数据处理：

```java
// 创建流式管道
StreamingPipeline<String> pipeline = new StreamingPipeline<>("textPipeline");
pipeline.addProcessor(new TextProcessor())
        .addProcessor(new SentimentAnalyzer());

// 执行流式处理
Flux<String> result = pipeline.execute(inputStream);
```

### RAG系统

内置检索增强生成功能：

```java
@Autowired
private RagRetriever ragRetriever;

// 检索相关文档
Mono<List<Document>> documents = ragRetriever.retrieve("查询文本", 5);

// 混合检索
Mono<List<Document>> hybridResults = ragRetriever.hybridRetrieve("查询文本", 10);
```

### 监控和观测

内置完整的监控体系：

- **指标收集**: 基于Micrometer的性能指标
- **链路追踪**: 分布式请求追踪
- **健康检查**: 组件和应用健康状态
- **告警通知**: 异常检测和告警

## 📊 监控端点

启动应用后，可以访问以下监控端点：

- **健康检查**: `http://localhost:8080/actuator/health`
- **应用信息**: `http://localhost:8080/actuator/info`
- **性能指标**: `http://localhost:8080/actuator/metrics`
- **Prometheus**: `http://localhost:8080/actuator/prometheus`

## 🔧 配置选项

### 监控配置

```yaml
sen-aiflow:
  monitoring:
    enabled: true              # 启用监控
    sampling-rate: 1.0         # 采样率
    record-parameters: false   # 记录参数
    record-return-value: false # 记录返回值
    record-exceptions: true    # 记录异常
```

### 响应式配置

```yaml
sen-aiflow:
  reactive:
    enabled: true              # 启用响应式
    thread-pool-size: 8        # 线程池大小
    queue-size: 10000          # 队列大小
```

### 流式处理配置

```yaml
sen-aiflow:
  streaming:
    enabled: true              # 启用流式处理
    buffer-size: 1000          # 缓冲区大小
    timeout: 30s               # 超时时间
    backpressure-strategy: BUFFER  # 背压策略
```

### 存储配置

```yaml
sen-aiflow:
  storage:
    type: memory               # 存储类型 (memory, redis, jpa)
    max-size: 1000            # 最大存储数量
```

### 嵌入配置

```yaml
sen-aiflow:
  embedding:
    provider: openai           # 提供商
    model: text-embedding-ada-002  # 模型名称
    dimension: 1536            # 向量维度
```

### RAG配置

```yaml
sen-aiflow:
  rag:
    enabled: true              # 启用RAG
    max-retrieval: 10          # 最大检索数量
    similarity-threshold: 0.7  # 相似度阈值
```

## 🚀 运行示例

### 1. 编译项目

```bash
mvn clean compile
```

### 2. 运行示例应用

```bash
cd examples
mvn spring-boot:run
```

### 3. 访问应用

- 应用地址: `http://localhost:8080`
- API接口: `http://localhost:8080/api/v1`
- 监控端点: `http://localhost:8080/actuator`
- H2数据库: `http://localhost:8080/h2-console`

## 📚 文档

- [架构设计](doc/01-架构设计.md)
- [快速开始](doc/02-快速开始.md)
- [RAG系统](doc/04-RAG检索增强生成系统.md)
- [响应式编程指南](doc/04-响应式编程指南.md)
- [组件开发指南](doc/components/README.md)
- [管道编排指南](doc/pipelines/README.md)

## 🔌 扩展开发

### 自定义组件

```java
@Component
@SenAiFlowComponent(
    type = ComponentType.PROCESSOR,
    description = "自定义处理器",
    tags = {"custom", "processor"}
)
public class CustomProcessor extends AbstractComponent implements Processor<String, String> {
    
    public CustomProcessor() {
        super("CustomProcessor", "自定义处理器", "1.0.0");
    }
    
    @Override
    public Mono<String> process(String input) {
        // 自定义处理逻辑
        return Mono.just("processed: " + input);
    }
}
```

### 自定义管道

```java
@Service
public class CustomPipelineService {
    
    @Autowired
    private StreamingPipeline<String> pipeline;
    
    public Flux<String> processText(Flux<String> input) {
        return pipeline.execute(input);
    }
}
```

## 🤝 贡献

欢迎贡献代码！请阅读[贡献指南](CONTRIBUTING.md)了解详情。

### 开发环境设置

1. Fork项目
2. 创建特性分支: `git checkout -b feature/amazing-feature`
3. 提交更改: `git commit -m 'Add amazing feature'`
4. 推送分支: `git push origin feature/amazing-feature`
5. 创建Pull Request

## 📄 许可证

本项目采用 [Apache License 2.0](LICENSE) 许可证。

## 🆘 支持

- 文档: [项目文档](https://senaiflow.com/docs)
- 问题: [GitHub Issues](https://github.com/senrian/sen-aiflow/issues)
- 讨论: [GitHub Discussions](https://github.com/senrian/sen-aiflow/discussions)
- 邮件: senrian@example.com

## 🏗️ 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Sen AI Flow Framework                    │
├─────────────────────────────────────────────────────────────┤
│  Web Layer (REST API, WebSocket)                           │
├─────────────────────────────────────────────────────────────┤
│  Service Layer (LLM, AI Services)                          │
├─────────────────────────────────────────────────────────────┤
│  Engine Layer (DAG Engine, Pipeline Manager)               │
├─────────────────────────────────────────────────────────────┤
│  Component Layer (Processors, Generators, Retrievers)      │
├─────────────────────────────────────────────────────────────┤
│  Core Layer (Annotations, Configuration, Monitoring)       │
├─────────────────────────────────────────────────────────────┤
│  Storage Layer (Vector DB, Document Store)                 │
└─────────────────────────────────────────────────────────────┘
```

---

**Sen AI Flow** - 让AI应用开发更简单、更高效！

*基于Spring Boot、Project Reactor和现代AI技术构建*
