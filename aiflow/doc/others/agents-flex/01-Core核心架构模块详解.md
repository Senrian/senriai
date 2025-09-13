# Agents-Flex - Core核心架构模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Agents-Flex Core是整个框架的核心模块，提供了AI Agent开发的基础架构和核心组件。该模块采用轻量级设计，支持多种AI模型、多种编程语言集成，以及灵活的工作流编排能力。

### 核心特性

- **统一抽象**: 提供统一的AI模型抽象接口
- **ReAct模式**: 内置Reasoning + Action的智能Agent模式
- **链式工作流**: 支持复杂的工作流编排
- **多语言支持**: 支持Java、Groovy、JavaScript等多种脚本语言
- **轻量级设计**: 最小依赖，易于集成
- **事件驱动**: 完整的事件监听和处理机制

## 2. 核心架构

### 整体架构图

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Message System    │───▶│     LLM Provider     │───▶│   Function Calling  │
│   (消息抽象)        │    │     (模型抽象)       │    │   (工具调用)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          ▼                            ▼                           ▼
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   ReAct Agent       │    │    Chain Workflow    │    │   Document Parser   │
│   (推理行动)        │    │    (工作流编排)      │    │   (文档处理)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          └────────────────────────────┼───────────────────────────┘
                                       ▼
                           ┌──────────────────────┐
                           │   Memory & Context   │
                           │   (记忆与上下文)     │
                           └──────────────────────┘
```

### 核心组件层次

```java
// 核心接口层次结构
Llm (统一LLM接口)
├── BaseLlm (基础实现)
├── OpenAI, Qwen, Spark等 (具体实现)

Message (消息抽象)
├── HumanMessage (用户消息)
├── AiMessage (AI响应)
├── SystemMessage (系统消息)
├── FunctionMessage (函数调用消息)

Chain (工作流)
├── ChainNode (节点抽象)
├── ChainEdge (边连接)
├── ChainContext (执行上下文)

Agent (智能体)
├── ReActAgent (推理行动Agent)
├── SimpleAgent (简单Agent)
```

## 3. 核心组件详解

### 3.1 LLM统一抽象

```java
public interface Llm {
    // 同步聊天
    AiMessageResponse chat(ChatContext context, ChatOptions options);
    
    // 异步聊天
    void chatAsync(ChatContext context, ChatOptions options, StreamResponseListener listener);
    
    // 流式聊天
    void chatStream(ChatContext context, ChatOptions options, StreamResponseListener listener);
}
```

**设计优势**:
- **统一接口**: 所有LLM提供商使用相同的API
- **多种模式**: 支持同步、异步、流式三种调用方式
- **灵活配置**: 通过ChatOptions进行细粒度配置

### 3.2 Message消息系统

```java
// 消息基类
public abstract class Message {
    protected MessageType messageType;
    protected Object content;
    protected Map<String, Object> metadata;
    
    public abstract String getContent();
}

// 具体消息类型
public class HumanMessage extends Message {
    public HumanMessage(String content) {
        this.messageType = MessageType.HUMAN;
        this.content = content;
    }
}

public class AiMessage extends Message {
    private List<FunctionCall> functionCalls;
    
    public AiMessage(String content) {
        this.messageType = MessageType.AI;
        this.content = content;
    }
}
```

**核心特性**:
- **类型安全**: 强类型的消息系统
- **元数据支持**: 每个消息都可以携带元数据
- **Function Calling**: 内置函数调用支持

### 3.3 ReAct智能体

```java
public class ReActAgent {
    private static final String DEFAULT_PROMPT_TEMPLATE = """
        你是一个 ReAct Agent，结合 Reasoning（推理）和 Action（行动）来解决问题。
        
        如果问题可以通过常识直接回答 → 请直接输出自然语言回答。
        如果问题需要调用工具才能解决 → 请严格按照 ReAct 格式响应。
        
        ReAct 格式：
        Thought: 描述你对当前问题的理解和下一步计划
        Action: 选择一个工具名称
        Action Input: 工具所需的JSON格式参数
        
        如果已获得足够信息：
        Final Answer: [你的最终回答]
        
        可用工具列表：
        {tools}
        
        用户问题：{user_input}
        """;
    
    private Llm llm;
    private List<Function> functions;
    private int maxIterations = 5;
    
    public AiMessageResponse chat(String userInput) {
        ChatContext context = new ChatContext();
        
        // 构建系统提示
        String systemPrompt = buildSystemPrompt(userInput, functions);
        context.addMessage(new SystemMessage(systemPrompt));
        
        // 迭代执行ReAct循环
        for (int i = 0; i < maxIterations; i++) {
            AiMessageResponse response = llm.chat(context, ChatOptions.defaults());
            
            if (isFinished(response)) {
                return response;
            }
            
            // 解析Action并执行
            ActionResult actionResult = executeAction(response);
            context.addMessage(new HumanMessage("Observation: " + actionResult.getResult()));
        }
        
        return AiMessageResponse.error("达到最大迭代次数");
    }
}
```

**核心机制**:
- **推理循环**: 自动执行Thought→Action→Observation循环
- **工具集成**: 无缝集成外部工具和函数
- **智能判断**: 自动判断是否需要工具调用
- **迭代控制**: 防止无限循环的安全机制

### 3.4 Chain工作流系统

```java
public class Chain extends ChainNode {
    protected List<ChainNode> nodes;
    protected List<ChainEdge> edges;
    protected Map<String, Object> executeResult;
    protected Map<String, NodeContext> nodeContexts;
    
    // 执行工作流
    public Map<String, Object> executeForResult(Map<String, Object> variables) {
        // 1. 初始化上下文
        ChainContext context = new ChainContext(variables);
        
        // 2. 找到起始节点
        List<ChainNode> startNodes = findStartNodes();
        
        // 3. 并行执行起始节点
        startNodes.parallelStream().forEach(node -> {
            executeNode(node, context);
        });
        
        // 4. 等待所有节点完成
        phaser.arriveAndAwaitAdvance();
        
        return context.getMemory().getAll();
    }
    
    private void executeNode(ChainNode node, ChainContext context) {
        try {
            // 执行节点逻辑
            Object result = node.execute(context);
            
            // 更新上下文
            updateNodeContext(node.getId(), result);
            
            // 触发下游节点
            triggerDownstreamNodes(node, context);
            
        } catch (Exception e) {
            handleNodeError(node, e);
        }
    }
}
```

**工作流特性**:
- **并行执行**: 支持节点的并行执行
- **条件分支**: 支持基于条件的流程控制
- **错误处理**: 完善的异常处理和恢复机制
- **状态管理**: 维护整个工作流的执行状态

### 3.5 Memory记忆系统

```java
public interface Memory {
    void put(String key, Object value);
    <T> T get(String key, Class<T> clazz);
    Map<String, Object> getAll();
    void clear();
}

public class DefaultContextMemory implements Memory {
    private final Map<String, Object> data = new ConcurrentHashMap<>();
    
    @Override
    public void put(String key, Object value) {
        data.put(key, value);
        
        // 触发内存更新事件
        eventBus.post(new MemoryUpdateEvent(key, value));
    }
    
    @Override
    public <T> T get(String key, Class<T> clazz) {
        Object value = data.get(key);
        return clazz.cast(value);
    }
    
    // 支持JSON路径查询
    public Object getByJsonPath(String jsonPath) {
        return JSONPath.eval(data, jsonPath);
    }
}
```

## 4. 事件系统

### 4.1 事件监听机制

```java
// 事件接口
public interface ChainEvent {
    String getEventType();
    long getTimestamp();
    Map<String, Object> getEventData();
}

// 具体事件类型
public class NodeExecutionEvent implements ChainEvent {
    private final String nodeId;
    private final Object result;
    private final long timestamp;
    
    @Override
    public String getEventType() {
        return "NODE_EXECUTION";
    }
}

// 事件监听器
public interface ChainEventListener<T extends ChainEvent> {
    void onEvent(T event);
    Class<T> getEventType();
}

// 事件管理器
public class EventManager {
    private final Map<Class<?>, List<ChainEventListener>> listeners = new HashMap<>();
    
    public <T extends ChainEvent> void addEventListener(ChainEventListener<T> listener) {
        listeners.computeIfAbsent(listener.getEventType(), k -> new ArrayList<>())
                 .add(listener);
    }
    
    public void publishEvent(ChainEvent event) {
        List<ChainEventListener> eventListeners = listeners.get(event.getClass());
        if (eventListeners != null) {
            eventListeners.forEach(listener -> listener.onEvent(event));
        }
    }
}
```

### 4.2 内置事件类型

- **ChainStartEvent**: 工作流开始执行
- **ChainCompleteEvent**: 工作流执行完成
- **NodeExecutionEvent**: 节点执行事件
- **NodeErrorEvent**: 节点执行错误
- **ChainSuspendEvent**: 工作流暂停
- **MemoryUpdateEvent**: 内存更新事件

## 5. 函数调用系统

### 5.1 Function定义

```java
public class Function {
    private String name;
    private String description;
    private List<Parameter> parameters;
    private FunctionInvoker invoker;
    
    // 执行函数
    public Object invoke(Map<String, Object> arguments) {
        // 参数验证
        validateParameters(arguments);
        
        // 执行函数逻辑
        return invoker.invoke(arguments);
    }
}

// 函数调用器接口
@FunctionalInterface
public interface FunctionInvoker {
    Object invoke(Map<String, Object> arguments);
}
```

### 5.2 内置函数

```java
// 时间函数
Function getCurrentTime = Function.builder()
    .name("getCurrentTime")
    .description("获取当前时间")
    .invoke(args -> new Date().toString());

// 计算函数
Function calculate = Function.builder()
    .name("calculate")
    .description("执行数学计算")
    .parameter("expression", "数学表达式", String.class, true)
    .invoke(args -> {
        String expression = (String) args.get("expression");
        return evaluateExpression(expression);
    });

// HTTP请求函数
Function httpGet = Function.builder()
    .name("httpGet")
    .description("发送HTTP GET请求")
    .parameter("url", "请求URL", String.class, true)
    .invoke(args -> {
        String url = (String) args.get("url");
        return httpClient.get(url);
    });
```

## 6. 文档处理系统

### 6.1 Document抽象

```java
public class Document {
    private String content;
    private Map<String, Object> metadata;
    
    public Document(String content) {
        this.content = content;
        this.metadata = new HashMap<>();
    }
    
    // 添加元数据
    public void addMetadata(String key, Object value) {
        metadata.put(key, value);
    }
    
    // 获取元数据
    public <T> T getMetadata(String key, Class<T> clazz) {
        return clazz.cast(metadata.get(key));
    }
}
```

### 6.2 DocumentParser接口

```java
public interface DocumentParser {
    Document parse(InputStream stream);
    
    default List<Document> parseMultiple(InputStream stream) {
        return Arrays.asList(parse(stream));
    }
}

// PDF解析器
public class PdfBoxDocumentParser implements DocumentParser {
    @Override
    public Document parse(InputStream stream) {
        try (PDDocument pdfDocument = PDDocument.load(stream)) {
            PDFTextStripper stripper = new PDFTextStripper();
            String text = stripper.getText(pdfDocument);
            return new Document(text);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    
    // 按页解析
    public List<Document> parseWithPage(InputStream inputStream) {
        try (PDDocument pdfDocument = PDDocument.load(inputStream)) {
            List<Document> documents = new ArrayList<>();
            int pageCount = pdfDocument.getNumberOfPages();
            
            for (int pageNumber = 1; pageNumber <= pageCount; pageNumber++) {
                PDFTextStripper stripper = new PDFTextStripper();
                stripper.setStartPage(pageNumber);
                stripper.setEndPage(pageNumber);
                String content = stripper.getText(pdfDocument);
                
                Document document = new Document(content);
                document.addMetadata("pageNumber", pageNumber);
                documents.add(document);
            }
            return documents;
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 7. 并发与线程安全

### 7.1 线程池管理

```java
public class NamedThreadPools {
    private static final Map<String, ExecutorService> THREAD_POOLS = new ConcurrentHashMap<>();
    
    public static ExecutorService newFixedThreadPool(String poolName) {
        return THREAD_POOLS.computeIfAbsent(poolName, name -> {
            ThreadFactory threadFactory = r -> {
                Thread thread = new Thread(r);
                thread.setName(name + "-" + thread.getId());
                thread.setDaemon(false);
                return thread;
            };
            
            return Executors.newFixedThreadPool(
                Runtime.getRuntime().availableProcessors(),
                threadFactory
            );
        });
    }
    
    public static void shutdown() {
        THREAD_POOLS.values().forEach(ExecutorService::shutdown);
        THREAD_POOLS.clear();
    }
}
```

### 7.2 并发控制

```java
public class ConcurrentChainExecutor {
    private final Phaser phaser = new Phaser(1);
    private final ExecutorService executor = NamedThreadPools.newFixedThreadPool("chain-executor");
    
    public void executeAsync(ChainNode node, ChainContext context) {
        phaser.register();
        
        CompletableFuture.runAsync(() -> {
            try {
                node.execute(context);
            } finally {
                phaser.arriveAndDeregister();
            }
        }, executor);
    }
    
    public void waitForCompletion() {
        phaser.arriveAndAwaitAdvance();
    }
}
```

## 8. 配置与扩展

### 8.1 配置管理

```java
public class AgentsFlexConfig {
    private LlmConfig llmConfig;
    private ChainConfig chainConfig;
    private ThreadPoolConfig threadPoolConfig;
    
    public static class LlmConfig {
        private String defaultModel = "gpt-3.5-turbo";
        private int timeout = 30000;
        private int maxRetries = 3;
    }
    
    public static class ChainConfig {
        private int maxNodes = 1000;
        private int executionTimeout = 300000;
        private boolean enableAsync = true;
    }
}
```

### 8.2 SPI扩展机制

```java
// 扩展点接口
public interface LlmProvider {
    String getName();
    Llm createLlm(Map<String, Object> config);
    boolean supports(String modelType);
}

// 扩展加载器
public class ExtensionLoader {
    private static final Map<Class<?>, List<Object>> EXTENSIONS = new HashMap<>();
    
    @SuppressWarnings("unchecked")
    public static <T> List<T> getExtensions(Class<T> type) {
        return (List<T>) EXTENSIONS.computeIfAbsent(type, t -> {
            ServiceLoader<T> loader = ServiceLoader.load(type);
            List<T> extensions = new ArrayList<>();
            loader.forEach(extensions::add);
            return extensions;
        });
    }
}
```

## 9. 性能优化

### 9.1 对象池化

```java
public class ObjectPoolManager {
    private static final Map<Class<?>, ObjectPool<?>> POOLS = new ConcurrentHashMap<>();
    
    @SuppressWarnings("unchecked")
    public static <T> ObjectPool<T> getPool(Class<T> clazz, Supplier<T> factory) {
        return (ObjectPool<T>) POOLS.computeIfAbsent(clazz, k -> 
            new GenericObjectPool<>(new PooledObjectFactory<T>() {
                @Override
                public PooledObject<T> makeObject() {
                    return new DefaultPooledObject<>(factory.get());
                }
                
                @Override
                public void destroyObject(PooledObject<T> p) {
                    // 清理逻辑
                }
                
                @Override
                public boolean validateObject(PooledObject<T> p) {
                    return true;
                }
                
                @Override
                public void activateObject(PooledObject<T> p) {
                    // 激活逻辑
                }
                
                @Override
                public void passivateObject(PooledObject<T> p) {
                    // 钝化逻辑
                }
            })
        );
    }
}
```

### 9.2 缓存机制

```java
public class CacheManager {
    private static final Map<String, Cache<String, Object>> CACHES = new ConcurrentHashMap<>();
    
    public static Cache<String, Object> getCache(String cacheName) {
        return CACHES.computeIfAbsent(cacheName, name -> 
            Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(Duration.ofMinutes(30))
                .recordStats()
                .build()
        );
    }
}
```

## 10. 对Sen-AiFlow的启发

### 核心设计原则

1. **轻量级架构**: 最小依赖，易于集成和扩展
2. **统一抽象**: 提供统一的AI模型和消息抽象
3. **事件驱动**: 完整的事件系统支持复杂的业务逻辑
4. **并发友好**: 天然支持并发和异步处理
5. **工作流编排**: 灵活的链式工作流系统

### Sen-AiFlow实现借鉴

```java
// Sen-AiFlow中的核心架构设计
@SenAiFlowCore
public interface ReactiveAgent {
    Flux<AgentResponse> process(AgentRequest request);
    Mono<Void> addTool(ReactiveFunction function);
    Flux<ExecutionStep> getExecutionSteps();
}

@SenAiFlowCore
public interface ReactiveChain {
    Flux<ChainEvent> execute(Map<String, Object> context);
    Mono<ChainNode> addNode(ReactiveChainNode node);
    Mono<Void> addEdge(String fromNode, String toNode, Condition condition);
}

@SenAiFlowCore  
public interface ReactiveMemory {
    Mono<Void> store(String key, Object value);
    <T> Mono<T> retrieve(String key, Class<T> type);
    Flux<MemoryEvent> watchChanges();
}
```

Agents-Flex Core模块展示了轻量级AI框架的设计精髓，其统一抽象、事件驱动和工作流编排的设计理念为Sen-AiFlow的响应式架构提供了重要参考，特别是在保持简洁性的同时提供强大功能方面的平衡。 