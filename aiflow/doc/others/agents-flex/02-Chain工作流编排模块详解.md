# Agents-Flex - Chain工作流编排模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Agents-Flex的Chain模块是一个强大的工作流编排系统，支持复杂的业务逻辑编排、并行执行、条件分支和多语言脚本执行。该模块提供了可视化的工作流设计能力，让开发者可以通过节点和连线的方式构建复杂的AI应用逻辑。

### 核心特性

- **可视化编排**: 支持节点和边的图形化工作流设计
- **多语言执行**: 支持Java、Groovy、JavaScript、QLExpress等多种脚本语言
- **并行执行**: 智能的并行执行调度和同步机制
- **条件分支**: 基于表达式的动态流程控制
- **事件驱动**: 完整的工作流执行事件系统
- **状态管理**: 工作流执行状态的持久化和恢复
- **错误处理**: 完善的异常处理和重试机制

## 2. 核心架构

### 工作流执行架构

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Chain Definition  │───▶│   Execution Engine   │───▶│   Result Handler    │
│   (工作流定义)      │    │   (执行引擎)         │    │   (结果处理)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          ▼                            ▼                           ▼
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Node Repository   │    │   Context Manager    │    │   Event Publisher   │
│   (节点仓库)        │    │   (上下文管理)       │    │   (事件发布)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          └────────────────────────────┼───────────────────────────┘
                                       ▼
                           ┌──────────────────────┐
                           │   Script Executors   │
                           │   (脚本执行器)       │
                           └──────────────────────┘
```

### 节点类型体系

```
ChainNode (抽象节点)
├── BaseNode (基础节点)
│   ├── StartNode (开始节点)
│   ├── EndNode (结束节点)
│   └── ConditionNode (条件节点)
├── CodeNode (代码节点)
│   ├── GroovyExecNode (Groovy执行节点)
│   ├── JsExecNode (JavaScript执行节点)
│   └── QLExpressExecNode (QLExpress执行节点)
├── LlmNode (LLM调用节点)
├── HttpNode (HTTP请求节点)
└── CustomNode (自定义节点)
```

## 3. 核心实现分析

### 3.1 Chain工作流引擎

```java
public class Chain extends ChainNode {
    protected List<ChainNode> nodes;
    protected List<ChainEdge> edges;
    protected Map<String, Object> executeResult;
    protected Map<String, NodeContext> nodeContexts;
    protected ExecutorService asyncNodeExecutors;
    protected Phaser phaser;
    
    // 同步执行
    public Map<String, Object> executeForResult(Map<String, Object> variables) {
        logger.info("Starting chain execution with variables: {}", variables);
        
        try {
            // 1. 初始化执行上下文
            initializeExecution(variables);
            
            // 2. 查找起始节点
            List<ChainNode> startNodes = findStartNodes();
            if (startNodes.isEmpty()) {
                throw new ChainExecutionException("No start nodes found");
            }
            
            // 3. 执行起始节点
            for (ChainNode startNode : startNodes) {
                executeNode(startNode);
            }
            
            // 4. 等待所有节点完成
            phaser.arriveAndAwaitAdvance();
            
            // 5. 收集执行结果
            return collectExecutionResults();
            
        } catch (Exception e) {
            logger.error("Chain execution failed", e);
            handleChainError(e);
            throw new ChainExecutionException("Chain execution failed", e);
        }
    }
    
    // 异步执行节点
    private void executeNode(ChainNode node) {
        if (node.isAsync()) {
            // 异步执行
            phaser.register();
            asyncNodeExecutors.submit(() -> {
                try {
                    executeNodeInternal(node);
                } finally {
                    phaser.arriveAndDeregister();
                }
            });
        } else {
            // 同步执行
            executeNodeInternal(node);
        }
    }
    
    private void executeNodeInternal(ChainNode node) {
        try {
            logger.debug("Executing node: {}", node.getName());
            
            // 1. 检查节点执行条件
            if (!checkNodeCondition(node)) {
                logger.debug("Node condition not met, skipping: {}", node.getName());
                return;
            }
            
            // 2. 构建节点上下文
            NodeContext nodeContext = buildNodeContext(node);
            
            // 3. 执行节点逻辑
            Object result = node.execute(nodeContext);
            
            // 4. 更新节点上下文
            updateNodeContext(node.getId(), result);
            
            // 5. 触发后续节点
            triggerDownstreamNodes(node);
            
            // 6. 发布节点执行事件
            publishNodeExecutionEvent(node, result);
            
        } catch (Exception e) {
            logger.error("Node execution failed: {}", node.getName(), e);
            handleNodeError(node, e);
        }
    }
}
```

**核心机制**:
- **并发调度**: 使用Phaser实现复杂的并发同步
- **条件执行**: 支持基于表达式的条件判断
- **上下文传递**: 节点间的数据传递和状态管理
- **异常处理**: 完善的错误处理和恢复机制

### 3.2 ChainNode节点抽象

```java
public abstract class ChainNode {
    protected String id;
    protected String name;
    protected String description;
    protected boolean async = false;
    protected NodeStatus nodeStatus = NodeStatus.READY;
    protected Memory memory;
    
    // 执行节点逻辑
    public abstract Object execute(NodeContext context) throws Exception;
    
    // 检查节点执行条件
    public boolean checkCondition(NodeContext context) {
        if (condition == null) {
            return true;
        }
        
        try {
            // 使用表达式引擎评估条件
            ExpressionEngine engine = ExpressionEngineFactory.getEngine();
            return engine.evaluate(condition, context.getVariables(), Boolean.class);
        } catch (Exception e) {
            logger.warn("Condition evaluation failed for node: {}, condition: {}", 
                       name, condition, e);
            return false;
        }
    }
    
    // 获取输出参数
    public Map<String, Object> getOutputs(NodeContext context) {
        Map<String, Object> outputs = new HashMap<>();
        
        if (outputDefs != null) {
            for (OutputDef outputDef : outputDefs) {
                Object value = extractOutputValue(outputDef, context);
                outputs.put(outputDef.getName(), value);
            }
        }
        
        return outputs;
    }
}
```

### 3.3 多语言脚本执行

#### Groovy执行节点

```java
public class GroovyExecNode extends CodeNode {
    
    @Override
    protected Map<String, Object> executeCode(String code, Chain chain) {
        Binding binding = new Binding();
        
        // 注入参数
        Map<String, Object> parameterValues = chain.getParameterValues(this);
        if (parameterValues != null) {
            parameterValues.forEach(binding::setVariable);
        }
        
        // 注入内置对象
        Map<String, Object> result = new HashMap<>();
        binding.setVariable("_result", result);
        binding.setVariable("_chain", chain);
        binding.setVariable("_context", chain.getNodeContext(this.id));
        binding.setVariable("_logger", LoggerFactory.getLogger(this.getClass()));
        
        try {
            GroovyShell shell = new GroovyShell(binding);
            shell.evaluate(code);
            
            logger.debug("Groovy code executed successfully, result: {}", result);
            return result;
            
        } catch (Exception e) {
            logger.error("Groovy execution failed: {}", e.getMessage(), e);
            throw new RuntimeException("Groovy execution failed", e);
        }
    }
}
```

#### JavaScript执行节点

```java
public class JsExecNode extends CodeNode {
    // 使用GraalVM的JavaScript引擎
    private static final Context.Builder CONTEXT_BUILDER = Context.newBuilder("js")
        .option("engine.WarnInterpreterOnly", "false")
        .allowHostAccess(HostAccess.ALL)
        .allowHostClassLookup(className -> false)
        .option("js.ecmascript-version", "2021");
    
    @Override
    protected Map<String, Object> executeCode(String code, Chain chain) {
        try (Context context = CONTEXT_BUILDER.build()) {
            Value bindings = context.getBindings("js");
            
            // 注入全局变量
            Map<String, Object> all = chain.getMemory().getAll();
            all.forEach((key, value) -> {
                if (!key.contains(".")) {
                    bindings.putMember(key, JsInteropUtils.wrapJavaValueForJS(context, value));
                }
            });
            
            // 注入参数
            Map<String, Object> parameterValues = chain.getParameterValues(this);
            if (parameterValues != null) {
                parameterValues.forEach((key, value) -> {
                    bindings.putMember(key, JsInteropUtils.wrapJavaValueForJS(context, value));
                });
            }
            
            // 创建结果对象
            context.eval("js", "var _result = {};");
            bindings.putMember("_chain", chain);
            bindings.putMember("_context", chain.getNodeContext(this.id));
            
            // 执行用户代码
            context.eval("js", code);
            
            // 提取结果
            Value resultValue = bindings.getMember("_result");
            return GraalToFastjsonUtils.toJSONObject(resultValue);
            
        } catch (Exception e) {
            logger.error("JavaScript execution failed: {}", e.getMessage(), e);
            throw new RuntimeException("JavaScript execution failed", e);
        }
    }
}
```

#### QLExpress执行节点

```java
public class QLExpressExecNode extends CodeNode {
    
    @Override
    protected Map<String, Object> executeCode(String code, Chain chain) {
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        
        // 注入参数
        Map<String, Object> parameterValues = chain.getParameterValues(this);
        if (parameterValues != null) {
            context.putAll(parameterValues);
        }
        
        // 注入内置对象
        Map<String, Object> result = new HashMap<>();
        context.put("_result", result);
        context.put("_chain", chain);
        context.put("_context", chain.getNodeContext(this.id));
        
        try {
            runner.execute(code, context, null, true, false);
            return result;
        } catch (Exception e) {
            logger.error("QLExpress execution failed: {}", e.getMessage(), e);
            throw new RuntimeException("QLExpress execution failed", e);
        }
    }
}
```

## 4. 事件系统

### 4.1 工作流事件类型

```java
// 工作流开始事件
public class ChainStartEvent extends ChainEvent {
    private final String chainId;
    private final Map<String, Object> initialVariables;
    
    public ChainStartEvent(String chainId, Map<String, Object> variables) {
        super("CHAIN_START");
        this.chainId = chainId;
        this.initialVariables = new HashMap<>(variables);
    }
}

// 节点执行事件
public class NodeExecutionEvent extends ChainEvent {
    private final String nodeId;
    private final String nodeName;
    private final Object result;
    private final long executionTime;
    
    public NodeExecutionEvent(String nodeId, String nodeName, Object result, long executionTime) {
        super("NODE_EXECUTION");
        this.nodeId = nodeId;
        this.nodeName = nodeName;
        this.result = result;
        this.executionTime = executionTime;
    }
}

// 工作流完成事件
public class ChainCompleteEvent extends ChainEvent {
    private final String chainId;
    private final Map<String, Object> finalResult;
    private final long totalExecutionTime;
    
    public ChainCompleteEvent(String chainId, Map<String, Object> result, long executionTime) {
        super("CHAIN_COMPLETE");
        this.chainId = chainId;
        this.finalResult = new HashMap<>(result);
        this.totalExecutionTime = executionTime;
    }
}
```

### 4.2 事件监听器

```java
// 工作流监控监听器
public class ChainMonitoringListener implements ChainEventListener {
    
    @Override
    public void onEvent(ChainEvent event) {
        switch (event.getEventType()) {
            case "CHAIN_START":
                handleChainStart((ChainStartEvent) event);
                break;
            case "NODE_EXECUTION":
                handleNodeExecution((NodeExecutionEvent) event);
                break;
            case "CHAIN_COMPLETE":
                handleChainComplete((ChainCompleteEvent) event);
                break;
            case "CHAIN_ERROR":
                handleChainError((ChainErrorEvent) event);
                break;
        }
    }
    
    private void handleChainStart(ChainStartEvent event) {
        logger.info("Chain started: {}, variables: {}", 
                   event.getChainId(), event.getInitialVariables());
        
        // 记录执行指标
        meterRegistry.counter("chain.executions.started", 
                             "chain_id", event.getChainId())
                     .increment();
    }
    
    private void handleNodeExecution(NodeExecutionEvent event) {
        logger.debug("Node executed: {} in {}ms", 
                    event.getNodeName(), event.getExecutionTime());
        
        // 记录节点执行时间
        meterRegistry.timer("node.execution.time",
                           "node_type", event.getNodeType(),
                           "node_name", event.getNodeName())
                     .record(Duration.ofMillis(event.getExecutionTime()));
    }
}
```

## 5. 条件分支与流程控制

### 5.1 条件表达式引擎

```java
public class ExpressionEngine {
    private final ScriptEngine scriptEngine;
    
    public ExpressionEngine() {
        ScriptEngineManager manager = new ScriptEngineManager();
        this.scriptEngine = manager.getEngineByName("javascript");
    }
    
    public <T> T evaluate(String expression, Map<String, Object> variables, Class<T> resultType) {
        try {
            // 注入变量到脚本上下文
            Bindings bindings = scriptEngine.createBindings();
            bindings.putAll(variables);
            
            // 执行表达式
            Object result = scriptEngine.eval(expression, bindings);
            
            // 类型转换
            return convertResult(result, resultType);
            
        } catch (ScriptException e) {
            throw new ExpressionEvaluationException("Expression evaluation failed: " + expression, e);
        }
    }
    
    @SuppressWarnings("unchecked")
    private <T> T convertResult(Object result, Class<T> resultType) {
        if (result == null) {
            return null;
        }
        
        if (resultType.isAssignableFrom(result.getClass())) {
            return (T) result;
        }
        
        // 基本类型转换
        if (resultType == Boolean.class || resultType == boolean.class) {
            return (T) Boolean.valueOf(result.toString());
        }
        
        if (resultType == Integer.class || resultType == int.class) {
            return (T) Integer.valueOf(result.toString());
        }
        
        if (resultType == String.class) {
            return (T) result.toString();
        }
        
        throw new IllegalArgumentException("Cannot convert result to type: " + resultType);
    }
}
```

### 5.2 条件节点实现

```java
public class ConditionNode extends BaseNode {
    private String condition;
    private String trueNextNodeId;
    private String falseNextNodeId;
    
    @Override
    public Object execute(NodeContext context) throws Exception {
        ExpressionEngine engine = new ExpressionEngine();
        boolean conditionResult = engine.evaluate(condition, context.getVariables(), Boolean.class);
        
        logger.debug("Condition '{}' evaluated to: {}", condition, conditionResult);
        
        // 根据条件结果设置下一个节点
        String nextNodeId = conditionResult ? trueNextNodeId : falseNextNodeId;
        context.setNextNodeId(nextNodeId);
        
        return conditionResult;
    }
    
    // 重写边的获取逻辑
    @Override
    public List<ChainEdge> getOutwardEdges() {
        List<ChainEdge> edges = new ArrayList<>();
        
        if (trueNextNodeId != null) {
            ChainEdge trueEdge = new ChainEdge(this.id, trueNextNodeId);
            trueEdge.setCondition("result == true");
            edges.add(trueEdge);
        }
        
        if (falseNextNodeId != null) {
            ChainEdge falseEdge = new ChainEdge(this.id, falseNextNodeId);
            falseEdge.setCondition("result == false");
            edges.add(falseEdge);
        }
        
        return edges;
    }
}
```

## 6. 状态管理与持久化

### 6.1 工作流状态

```java
public enum ChainStatus {
    READY,      // 准备就绪
    RUNNING,    // 执行中
    SUSPENDED,  // 已暂停
    COMPLETED,  // 已完成
    FAILED,     // 执行失败
    CANCELLED   // 已取消
}

public class ChainExecution {
    private String chainId;
    private ChainStatus status;
    private Map<String, Object> variables;
    private Map<String, NodeContext> nodeContexts;
    private List<ExecutionStep> executionSteps;
    private Exception lastError;
    private long startTime;
    private long endTime;
    
    // 保存执行状态
    public void saveState() {
        ChainStateRepository repository = ChainStateRepository.getInstance();
        repository.saveChainExecution(this);
    }
    
    // 恢复执行状态
    public static ChainExecution loadState(String chainId) {
        ChainStateRepository repository = ChainStateRepository.getInstance();
        return repository.loadChainExecution(chainId);
    }
}
```

### 6.2 状态持久化

```java
public interface ChainStateRepository {
    void saveChainExecution(ChainExecution execution);
    ChainExecution loadChainExecution(String chainId);
    List<ChainExecution> findByStatus(ChainStatus status);
    void deleteChainExecution(String chainId);
}

public class DatabaseChainStateRepository implements ChainStateRepository {
    private final JdbcTemplate jdbcTemplate;
    
    @Override
    public void saveChainExecution(ChainExecution execution) {
        String sql = """
            INSERT INTO chain_executions (chain_id, status, variables, node_contexts, 
                                         execution_steps, last_error, start_time, end_time)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE 
                status = VALUES(status),
                variables = VALUES(variables),
                node_contexts = VALUES(node_contexts),
                execution_steps = VALUES(execution_steps),
                last_error = VALUES(last_error),
                end_time = VALUES(end_time)
            """;
            
        jdbcTemplate.update(sql,
            execution.getChainId(),
            execution.getStatus().name(),
            JsonUtils.toJson(execution.getVariables()),
            JsonUtils.toJson(execution.getNodeContexts()),
            JsonUtils.toJson(execution.getExecutionSteps()),
            execution.getLastError() != null ? execution.getLastError().getMessage() : null,
            new Timestamp(execution.getStartTime()),
            execution.getEndTime() > 0 ? new Timestamp(execution.getEndTime()) : null
        );
    }
}
```

## 7. 工作流设计器

### 7.1 工作流定义DSL

```java
// 流畅式API构建工作流
public class ChainBuilder {
    private Chain chain;
    
    public static ChainBuilder create(String chainName) {
        return new ChainBuilder(chainName);
    }
    
    public ChainBuilder addNode(ChainNode node) {
        chain.addNode(node);
        return this;
    }
    
    public ChainBuilder addEdge(String fromNodeId, String toNodeId) {
        chain.addEdge(new ChainEdge(fromNodeId, toNodeId));
        return this;
    }
    
    public ChainBuilder addConditionalEdge(String fromNodeId, String toNodeId, String condition) {
        ChainEdge edge = new ChainEdge(fromNodeId, toNodeId);
        edge.setCondition(condition);
        chain.addEdge(edge);
        return this;
    }
    
    public Chain build() {
        validateChain();
        return chain;
    }
}

// 使用示例
Chain workflow = ChainBuilder.create("user-onboarding")
    .addNode(new StartNode("start"))
    .addNode(new GroovyExecNode("validate-user")
        .setCode("""
            def user = _context.getVariable('user')
            def isValid = user.email != null && user.name != null
            _result.put('isValid', isValid)
            _result.put('user', user)
            """))
    .addNode(new ConditionNode("check-validation")
        .setCondition("isValid == true")
        .setTrueNextNodeId("send-welcome")
        .setFalseNextNodeId("send-error"))
    .addNode(new HttpNode("send-welcome")
        .setUrl("http://api/send-welcome")
        .setMethod("POST"))
    .addNode(new HttpNode("send-error")
        .setUrl("http://api/send-error")
        .setMethod("POST"))
    .addNode(new EndNode("end"))
    .addEdge("start", "validate-user")
    .addEdge("validate-user", "check-validation")
    .addConditionalEdge("check-validation", "send-welcome", "result == true")
    .addConditionalEdge("check-validation", "send-error", "result == false")
    .addEdge("send-welcome", "end")
    .addEdge("send-error", "end")
    .build();
```

### 7.2 JSON工作流定义

```json
{
  "chainId": "user-onboarding-v1",
  "name": "用户注册工作流",
  "description": "处理用户注册流程",
  "nodes": [
    {
      "id": "start",
      "type": "StartNode",
      "name": "开始",
      "position": { "x": 100, "y": 100 }
    },
    {
      "id": "validate-user",
      "type": "GroovyExecNode",
      "name": "用户验证",
      "position": { "x": 300, "y": 100 },
      "properties": {
        "code": "def user = _context.getVariable('user')\ndef isValid = user.email != null && user.name != null\n_result.put('isValid', isValid)\n_result.put('user', user)"
      }
    },
    {
      "id": "check-validation",
      "type": "ConditionNode", 
      "name": "验证检查",
      "position": { "x": 500, "y": 100 },
      "properties": {
        "condition": "isValid == true"
      }
    },
    {
      "id": "send-welcome",
      "type": "HttpNode",
      "name": "发送欢迎邮件",
      "position": { "x": 700, "y": 50 },
      "properties": {
        "url": "http://api/send-welcome",
        "method": "POST"
      }
    },
    {
      "id": "send-error",
      "type": "HttpNode", 
      "name": "发送错误通知",
      "position": { "x": 700, "y": 150 },
      "properties": {
        "url": "http://api/send-error",
        "method": "POST"
      }
    },
    {
      "id": "end",
      "type": "EndNode",
      "name": "结束",
      "position": { "x": 900, "y": 100 }
    }
  ],
  "edges": [
    { "from": "start", "to": "validate-user" },
    { "from": "validate-user", "to": "check-validation" },
    { "from": "check-validation", "to": "send-welcome", "condition": "result == true" },
    { "from": "check-validation", "to": "send-error", "condition": "result == false" },
    { "from": "send-welcome", "to": "end" },
    { "from": "send-error", "to": "end" }
  ]
}
```

## 8. 性能优化与监控

### 8.1 执行性能优化

```java
public class ChainPerformanceOptimizer {
    
    // 节点执行优化
    public void optimizeNodeExecution(Chain chain) {
        // 1. 分析节点依赖关系
        Map<ChainNode, Set<ChainNode>> dependencies = analyzeDependencies(chain);
        
        // 2. 构建执行层级
        List<List<ChainNode>> executionLevels = buildExecutionLevels(dependencies);
        
        // 3. 并行执行同一层级的节点
        for (List<ChainNode> level : executionLevels) {
            executeNodesInParallel(level);
        }
    }
    
    // 资源池优化
    public void optimizeResourcePool(Chain chain) {
        // 根据工作流复杂度动态调整线程池大小
        int nodeCount = chain.getNodes().size();
        int parallelism = Math.min(nodeCount, Runtime.getRuntime().availableProcessors());
        
        ForkJoinPool customPool = new ForkJoinPool(parallelism);
        chain.setExecutorService(customPool);
    }
    
    // 内存优化
    public void optimizeMemoryUsage(Chain chain) {
        // 1. 清理不再需要的节点上下文
        chain.getNodeContexts().entrySet().removeIf(entry -> {
            NodeContext context = entry.getValue();
            return context.isCompleted() && !context.hasDownstreamDependencies();
        });
        
        // 2. 压缩大对象
        chain.getMemory().getAll().forEach((key, value) -> {
            if (value instanceof String && ((String) value).length() > 10000) {
                // 压缩大字符串
                String compressed = CompressionUtils.compress((String) value);
                chain.getMemory().put(key + "_compressed", compressed);
                chain.getMemory().remove(key);
            }
        });
    }
}
```

### 8.2 监控指标

```java
@Component
public class ChainMetrics {
    
    private final MeterRegistry meterRegistry;
    
    // 执行计数器
    private final Counter chainExecutionCounter;
    private final Counter nodeExecutionCounter;
    
    // 执行时间计时器
    private final Timer chainExecutionTimer;
    private final Timer nodeExecutionTimer;
    
    // 当前活跃执行数
    private final Gauge activeChainGauge;
    
    public ChainMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.chainExecutionCounter = Counter.builder("chain.executions.total")
            .description("Total chain executions")
            .register(meterRegistry);
            
        this.nodeExecutionCounter = Counter.builder("node.executions.total")
            .description("Total node executions")
            .register(meterRegistry);
            
        this.chainExecutionTimer = Timer.builder("chain.execution.duration")
            .description("Chain execution duration")
            .register(meterRegistry);
            
        this.nodeExecutionTimer = Timer.builder("node.execution.duration")
            .description("Node execution duration")
            .register(meterRegistry);
            
        this.activeChainGauge = Gauge.builder("chain.active.count")
            .description("Active chain executions")
            .register(meterRegistry, this, ChainMetrics::getActiveChainCount);
    }
    
    public void recordChainExecution(String chainId, Duration duration, boolean success) {
        chainExecutionCounter.increment(
            Tags.of(
                "chain_id", chainId,
                "status", success ? "success" : "failure"
            )
        );
        
        chainExecutionTimer.record(duration, Tags.of("chain_id", chainId));
    }
    
    public void recordNodeExecution(String nodeId, String nodeType, Duration duration, boolean success) {
        nodeExecutionCounter.increment(
            Tags.of(
                "node_id", nodeId,
                "node_type", nodeType,
                "status", success ? "success" : "failure"
            )
        );
        
        nodeExecutionTimer.record(duration, 
            Tags.of(
                "node_type", nodeType,
                "status", success ? "success" : "failure"
            )
        );
    }
}
```

## 9. 高级特性

### 9.1 工作流版本管理

```java
public class ChainVersionManager {
    
    public void saveChainVersion(Chain chain, String version) {
        ChainDefinition definition = new ChainDefinition();
        definition.setChainId(chain.getId());
        definition.setVersion(version);
        definition.setDefinition(JsonUtils.toJson(chain));
        definition.setCreateTime(System.currentTimeMillis());
        
        chainRepository.saveDefinition(definition);
    }
    
    public Chain loadChainVersion(String chainId, String version) {
        ChainDefinition definition = chainRepository.findDefinition(chainId, version);
        if (definition == null) {
            throw new ChainNotFoundException("Chain not found: " + chainId + "@" + version);
        }
        
        return JsonUtils.fromJson(definition.getDefinition(), Chain.class);
    }
    
    public List<String> getChainVersions(String chainId) {
        return chainRepository.findVersions(chainId);
    }
    
    public void rollbackToVersion(String chainId, String version) {
        Chain chain = loadChainVersion(chainId, version);
        deployChain(chain);
    }
}
```

### 9.2 工作流调试

```java
public class ChainDebugger {
    
    public void enableDebugMode(Chain chain) {
        // 添加调试监听器
        chain.addEventListener(new DebugEventListener());
        
        // 启用详细日志
        chain.setDebugMode(true);
        
        // 添加断点支持
        chain.getNodes().forEach(node -> {
            if (node instanceof DebuggableNode) {
                ((DebuggableNode) node).enableBreakpoint();
            }
        });
    }
    
    public class DebugEventListener implements ChainEventListener {
        @Override
        public void onEvent(ChainEvent event) {
            if (event instanceof NodeExecutionEvent) {
                NodeExecutionEvent nodeEvent = (NodeExecutionEvent) event;
                
                logger.info("=== Debug Info ===");
                logger.info("Node: {} ({})", nodeEvent.getNodeName(), nodeEvent.getNodeId());
                logger.info("Input: {}", nodeEvent.getInput());
                logger.info("Output: {}", nodeEvent.getResult());
                logger.info("Duration: {}ms", nodeEvent.getExecutionTime());
                logger.info("Memory: {}", getCurrentMemorySnapshot());
                logger.info("==================");
            }
        }
    }
}
```

## 10. 对Sen-AiFlow的启发

### 核心设计理念

1. **可视化编排**: 提供图形化的工作流设计界面
2. **多语言支持**: 支持多种脚本语言的节点执行
3. **并发调度**: 智能的并发执行和同步机制
4. **事件驱动**: 完整的事件系统支持业务监控
5. **状态管理**: 工作流执行状态的持久化和恢复

### Sen-AiFlow响应式工作流设计

```java
// Sen-AiFlow中的响应式工作流设计
@SenAiFlowComponent("reactive-workflow")
public class ReactiveChain {
    
    public Flux<ChainExecutionEvent> execute(Map<String, Object> initialContext) {
        return Flux.create(sink -> {
            // 1. 初始化执行上下文
            ChainContext context = new ChainContext(initialContext);
            sink.next(new ChainStartEvent(context));
            
            // 2. 构建执行图
            Flux<ReactiveChainNode> nodeStream = buildExecutionStream(context);
            
            // 3. 响应式执行节点
            nodeStream
                .flatMap(node -> executeNode(node, context))
                .doOnNext(sink::next)
                .doOnComplete(() -> sink.next(new ChainCompleteEvent(context)))
                .doOnError(error -> sink.next(new ChainErrorEvent(error)))
                .subscribe();
        });
    }
    
    private Mono<NodeExecutionEvent> executeNode(ReactiveChainNode node, ChainContext context) {
        return node.execute(context)
            .map(result -> new NodeExecutionEvent(node.getId(), result))
            .doOnNext(event -> updateContext(context, event));
    }
}
```

Agents-Flex的Chain模块展示了工作流编排系统的完整解决方案，其多语言支持、并发调度和可视化设计等特性为Sen-AiFlow的响应式工作流系统提供了重要参考，特别是在保持灵活性的同时提供强大功能方面的设计理念。 