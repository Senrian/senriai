# Agents-Flex - Core 核心模块详解

## 模块概述

**Agents-Flex Core** 是整个Agents-Flex框架的核心基础模块，提供了LLM应用开发的核心抽象和基础实现。该模块设计简洁而强大，为上层业务提供了统一的AI服务接口，支持多种LLM提供商和丰富的AI能力。

## 核心架构

### 1. 模块结构图

```
agents-flex-core/
├── com.agentsflex.core.llm/           # LLM抽象层
│   ├── Llm.java                       # LLM主接口
│   ├── BaseLlm.java                   # 基础LLM实现
│   ├── ChatOptions.java               # 对话配置
│   ├── LlmConfig.java                 # LLM配置
│   ├── MessageResponse.java           # 消息响应
│   ├── StreamResponseListener.java    # 流式响应监听器
│   ├── functions/                     # Function Calling
│   ├── embedding/                     # 嵌入模型
│   ├── rerank/                        # 重排序模型
│   ├── response/                      # 响应处理
│   ├── exception/                     # 异常处理
│   └── client/                        # 客户端抽象
├── com.agentsflex.core.prompt/        # 提示词管理
│   ├── Prompt.java                    # 提示词接口
│   ├── TextPrompt.java                # 文本提示词
│   ├── FunctionPrompt.java            # 函数提示词
│   └── HistoriesPrompt.java           # 历史对话提示词
├── com.agentsflex.core.message/       # 消息系统
│   ├── Message.java                   # 消息接口
│   ├── HumanMessage.java              # 人类消息
│   ├── AiMessage.java                 # AI消息
│   └── SystemMessage.java             # 系统消息
├── com.agentsflex.core.memory/        # 记忆系统
├── com.agentsflex.core.document/      # 文档处理
├── com.agentsflex.core.store/         # 向量存储
├── com.agentsflex.core.chain/         # 执行链
├── com.agentsflex.core.image/         # 图像处理
├── com.agentsflex.core.parser/        # 解析器
├── com.agentsflex.core.convert/       # 转换器
├── com.agentsflex.core.util/          # 工具类
└── com.agentsflex.core.react/         # 反应式处理
```

### 2. 设计理念

Agents-Flex Core模块遵循以下设计原则：

- **简洁至上**：API设计追求最小化样板代码
- **灵活扩展**：高度可扩展的插件化架构
- **类型安全**：强类型的接口设计
- **易于使用**：开箱即用的默认配置

## 核心接口详解

### 1. Llm 主接口

#### 1.1 接口设计

```java
public interface Llm extends EmbeddingModel {
  
    /**
     * 简单文本对话 - 最简洁的API
     */
    default String chat(String prompt) {
        return chat(prompt, ChatOptions.DEFAULT);
    }
  
    /**
     * 带配置的文本对话
     */
    default String chat(String prompt, ChatOptions options) {
        AbstractBaseMessageResponse<AiMessage> response = chat(new TextPrompt(prompt), options);
        if (response != null && response.isError()) {
            throw new LlmException(response.getErrorMessage());
        }
        return response != null && response.getMessage() != null ? 
               response.getMessage().getContent() : null;
    }
  
    /**
     * 结构化提示词对话
     */
    default AiMessageResponse chat(Prompt prompt) {
        return chat(prompt, ChatOptions.DEFAULT);
    }
  
    /**
     * 完整参数对话 - 核心方法
     */
    AiMessageResponse chat(Prompt prompt, ChatOptions options);
  
    /**
     * 简单流式对话
     */
    default void chatStream(String prompt, StreamResponseListener listener) {
        this.chatStream(new TextPrompt(prompt), listener, ChatOptions.DEFAULT);
    }
  
    /**
     * 带配置的流式对话
     */
    default void chatStream(String prompt, StreamResponseListener listener, ChatOptions options) {
        this.chatStream(new TextPrompt(prompt), listener, options);
    }
  
    /**
     * 结构化流式对话
     */
    default void chatStream(Prompt prompt, StreamResponseListener listener) {
        this.chatStream(prompt, listener, ChatOptions.DEFAULT);
    }
  
    /**
     * 完整参数流式对话 - 核心方法
     */
    void chatStream(Prompt prompt, StreamResponseListener listener, ChatOptions options);
}
```

#### 1.2 设计特点分析

**层次化API设计**：

- **Level 1**: `chat(String)` - 最简单的字符串输入输出
- **Level 2**: `chat(String, ChatOptions)` - 带配置参数
- **Level 3**: `chat(Prompt)` - 结构化提示词
- **Level 4**: `chat(Prompt, ChatOptions)` - 完整功能

这种设计让开发者可以根据需求选择合适的抽象级别，既有简洁性又保持了灵活性。

### 2. BaseLlm 基础实现

#### 2.1 抽象基类设计

```java
public abstract class BaseLlm implements Llm {
  
    protected LlmConfig config;
    protected ChatOptions defaultChatOptions;
  
    public BaseLlm(LlmConfig config) {
        this.config = config;
        this.defaultChatOptions = ChatOptions.DEFAULT;
    }
  
    /**
     * 子类需要实现的核心方法
     */
    @Override
    public abstract AiMessageResponse chat(Prompt prompt, ChatOptions options);
  
    /**
     * 子类需要实现的流式方法
     */
    @Override
    public abstract void chatStream(Prompt prompt, StreamResponseListener listener, 
                                   ChatOptions options);
  
    /**
     * 默认嵌入实现（可选）
     */
    @Override
    public EmbeddingResponse embedding(EmbeddingRequest request) {
        throw new UnsupportedOperationException("当前LLM不支持嵌入功能");
    }
  
    /**
     * 工具方法：构建请求头
     */
    protected Map<String, String> buildHeaders() {
        Map<String, String> headers = new HashMap<>();
        headers.put("Content-Type", "application/json");
        headers.put("User-Agent", "agents-flex/" + getVersion());
      
        if (config.getApiKey() != null) {
            headers.put("Authorization", "Bearer " + config.getApiKey());
        }
      
        return headers;
    }
  
    /**
     * 工具方法：处理API错误
     */
    protected void handleApiError(int statusCode, String responseBody) {
        switch (statusCode) {
            case 400:
                throw new LlmException("请求参数错误: " + responseBody);
            case 401:
                throw new LlmException("API密钥无效");
            case 429:
                throw new LlmException("请求频率超限");
            case 500:
                throw new LlmException("服务器内部错误");
            default:
                throw new LlmException("API调用失败，状态码: " + statusCode);
        }
    }
  
    /**
     * 工具方法：获取版本信息
     */
    protected String getVersion() {
        return getClass().getPackage().getImplementationVersion();
    }
}
```

### 3. ChatOptions 配置管理

#### 3.1 配置选项设计

```java
public class ChatOptions implements Serializable {
  
    /**
     * 默认配置实例
     */
    public static final ChatOptions DEFAULT = new ChatOptions();
  
    /**
     * 模型名称
     */
    private String model;
  
    /**
     * 温度参数 (0.0-2.0)
     */
    private Double temperature;
  
    /**
     * Top-p 采样参数
     */
    private Double topP;
  
    /**
     * Top-k 采样参数
     */
    private Integer topK;
  
    /**
     * 最大生成token数
     */
    private Integer maxTokens;
  
    /**
     * 停止词列表
     */
    private List<String> stop;
  
    /**
     * 流式输出
     */
    private Boolean stream;
  
    /**
     * 系统提示词
     */
    private String systemMessage;
  
    /**
     * 可用的函数列表
     */
    private List<ChatFunction> functions;
  
    /**
     * 函数调用模式
     */
    private String functionCall;
  
    /**
     * 扩展参数
     */
    private Map<String, Object> otherOptions;
  
    // 构建器模式
    public static class Builder {
        private ChatOptions options = new ChatOptions();
      
        public Builder model(String model) {
            options.model = model;
            return this;
        }
      
        public Builder temperature(Double temperature) {
            options.temperature = temperature;
            return this;
        }
      
        public Builder maxTokens(Integer maxTokens) {
            options.maxTokens = maxTokens;
            return this;
        }
      
        public Builder systemMessage(String systemMessage) {
            options.systemMessage = systemMessage;
            return this;
        }
      
        public Builder functions(List<ChatFunction> functions) {
            options.functions = functions;
            return this;
        }
      
        public Builder addFunction(ChatFunction function) {
            if (options.functions == null) {
                options.functions = new ArrayList<>();
            }
            options.functions.add(function);
            return this;
        }
      
        public Builder otherOption(String key, Object value) {
            if (options.otherOptions == null) {
                options.otherOptions = new HashMap<>();
            }
            options.otherOptions.put(key, value);
            return this;
        }
      
        public ChatOptions build() {
            return options;
        }
    }
  
    /**
     * 预设配置
     */
    public static ChatOptions create() {
        return new Builder().build();
    }
  
    public static ChatOptions createForCreative() {
        return new Builder()
            .temperature(0.9)
            .topP(0.9)
            .build();
    }
  
    public static ChatOptions createForPrecise() {
        return new Builder()
            .temperature(0.1)
            .topP(0.1)
            .build();
    }
  
    public static ChatOptions createForCode() {
        return new Builder()
            .temperature(0.2)
            .maxTokens(4000)
            .systemMessage("你是一个专业的编程助手，请提供准确、简洁的代码解答。")
            .build();
    }
}
```

### 4. 消息系统设计

#### 4.1 消息层次结构

```java
/**
 * 消息基类
 */
public abstract class Message implements Serializable {
  
    protected String content;
    protected Map<String, Object> metadata;
    protected long timestamp;
  
    public Message(String content) {
        this.content = content;
        this.metadata = new HashMap<>();
        this.timestamp = System.currentTimeMillis();
    }
  
    public abstract MessageType getMessageType();
  
    // Getters and Setters
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public Map<String, Object> getMetadata() { return metadata; }
    public long getTimestamp() { return timestamp; }
  
    // 元数据操作
    public void addMetadata(String key, Object value) {
        this.metadata.put(key, value);
    }
  
    public Object getMetadata(String key) {
        return this.metadata.get(key);
    }
}

/**
 * 人类消息
 */
public class HumanMessage extends Message {
  
    public HumanMessage(String content) {
        super(content);
    }
  
    @Override
    public MessageType getMessageType() {
        return MessageType.HUMAN;
    }
  
    // 便利构造方法
    public static HumanMessage of(String content) {
        return new HumanMessage(content);
    }
  
    public static HumanMessage withMetadata(String content, Map<String, Object> metadata) {
        HumanMessage message = new HumanMessage(content);
        message.metadata.putAll(metadata);
        return message;
    }
}

/**
 * AI消息
 */
public class AiMessage extends Message {
  
    private List<FunctionCall> functionCalls;
    private String finishReason;
    private Usage usage;
  
    public AiMessage(String content) {
        super(content);
    }
  
    @Override
    public MessageType getMessageType() {
        return MessageType.AI;
    }
  
    // 函数调用相关
    public boolean hasFunctionCalls() {
        return functionCalls != null && !functionCalls.isEmpty();
    }
  
    public void addFunctionCall(FunctionCall functionCall) {
        if (this.functionCalls == null) {
            this.functionCalls = new ArrayList<>();
        }
        this.functionCalls.add(functionCall);
    }
  
    // Getters and Setters
    public List<FunctionCall> getFunctionCalls() { return functionCalls; }
    public void setFunctionCalls(List<FunctionCall> functionCalls) { 
        this.functionCalls = functionCalls; 
    }
  
    public String getFinishReason() { return finishReason; }
    public void setFinishReason(String finishReason) { this.finishReason = finishReason; }
  
    public Usage getUsage() { return usage; }
    public void setUsage(Usage usage) { this.usage = usage; }
}

/**
 * 系统消息
 */
public class SystemMessage extends Message {
  
    public SystemMessage(String content) {
        super(content);
    }
  
    @Override
    public MessageType getMessageType() {
        return MessageType.SYSTEM;
    }
  
    // 预设系统消息
    public static SystemMessage assistant() {
        return new SystemMessage("你是一个有用的AI助手。");
    }
  
    public static SystemMessage coder() {
        return new SystemMessage("你是一个专业的编程助手，精通多种编程语言。");
    }
  
    public static SystemMessage translator() {
        return new SystemMessage("你是一个专业的翻译助手，能够准确翻译多种语言。");
    }
}

/**
 * 消息类型枚举
 */
public enum MessageType {
    HUMAN("human"),
    AI("ai"), 
    SYSTEM("system"),
    FUNCTION("function");
  
    private final String value;
  
    MessageType(String value) {
        this.value = value;
    }
  
    public String getValue() {
        return value;
    }
}
```

### 5. 提示词系统

#### 5.1 提示词抽象

```java
/**
 * 提示词接口 
 */
public interface Prompt {
  
    /**
     * 获取消息列表
     */
    List<Message> toMessages();
  
    /**
     * 获取变量映射
     */
    Map<String, Object> getVariables();
  
    /**
     * 变量替换
     */
    Prompt fillVariables(Map<String, Object> variables);
}

/**
 * 文本提示词实现
 */
public class TextPrompt implements Prompt {
  
    private final String text;
    private final Map<String, Object> variables;
  
    public TextPrompt(String text) {
        this.text = text;
        this.variables = new HashMap<>();
    }
  
    public TextPrompt(String text, Map<String, Object> variables) {
        this.text = text;
        this.variables = variables != null ? variables : new HashMap<>();
    }
  
    @Override
    public List<Message> toMessages() {
        String processedText = processTemplate(text, variables);
        return List.of(new HumanMessage(processedText));
    }
  
    @Override
    public Map<String, Object> getVariables() {
        return new HashMap<>(variables);
    }
  
    @Override
    public Prompt fillVariables(Map<String, Object> variables) {
        Map<String, Object> merged = new HashMap<>(this.variables);
        merged.putAll(variables);
        return new TextPrompt(text, merged);
    }
  
    /**
     * 简单的模板处理
     */
    private String processTemplate(String template, Map<String, Object> variables) {
        String result = template;
        for (Map.Entry<String, Object> entry : variables.entrySet()) {
            String placeholder = "{" + entry.getKey() + "}";
            result = result.replace(placeholder, String.valueOf(entry.getValue()));
        }
        return result;
    }
  
    // 便利工厂方法
    public static TextPrompt of(String text) {
        return new TextPrompt(text);
    }
  
    public static TextPrompt of(String text, Object... vars) {
        Map<String, Object> variables = new HashMap<>();
        for (int i = 0; i < vars.length; i += 2) {
            if (i + 1 < vars.length) {
                variables.put(vars[i].toString(), vars[i + 1]);
            }
        }
        return new TextPrompt(text, variables);
    }
}

/**
 * 历史对话提示词
 */
public class HistoriesPrompt implements Prompt {
  
    private final List<Message> messages;
  
    public HistoriesPrompt() {
        this.messages = new ArrayList<>();
    }
  
    public HistoriesPrompt(List<Message> messages) {
        this.messages = new ArrayList<>(messages);
    }
  
    @Override
    public List<Message> toMessages() {
        return new ArrayList<>(messages);
    }
  
    @Override
    public Map<String, Object> getVariables() {
        return new HashMap<>();
    }
  
    @Override
    public Prompt fillVariables(Map<String, Object> variables) {
        // 历史对话提示词不支持变量替换
        return this;
    }
  
    // 消息操作方法
    public HistoriesPrompt addMessage(Message message) {
        this.messages.add(message);
        return this;
    }
  
    public HistoriesPrompt addHumanMessage(String content) {
        return addMessage(new HumanMessage(content));
    }
  
    public HistoriesPrompt addAiMessage(String content) {
        return addMessage(new AiMessage(content));
    }
  
    public HistoriesPrompt addSystemMessage(String content) {
        return addMessage(new SystemMessage(content));
    }
  
    public HistoriesPrompt clearMessages() {
        this.messages.clear();
        return this;
    }
  
    public int getMessageCount() {
        return messages.size();
    }
  
    public Message getLastMessage() {
        return messages.isEmpty() ? null : messages.get(messages.size() - 1);
    }
  
    // 便利工厂方法
    public static HistoriesPrompt create() {
        return new HistoriesPrompt();
    }
  
    public static HistoriesPrompt withSystem(String systemMessage) {
        return new HistoriesPrompt().addSystemMessage(systemMessage);
    }
}
```

### 6. 响应处理系统

#### 6.1 响应类型层次

```java
/**
 * 基础消息响应
 */
public abstract class AbstractBaseMessageResponse<T extends Message> {
  
    protected T message;
    protected Usage usage;
    protected String errorMessage;
    protected Exception exception;
    protected Map<String, Object> metadata;
  
    public AbstractBaseMessageResponse() {
        this.metadata = new HashMap<>();
    }
  
    // 状态检查
    public boolean isError() {
        return exception != null || errorMessage != null;
    }
  
    public boolean isSuccess() {
        return !isError();
    }
  
    // Getters and Setters
    public T getMessage() { return message; }
    public void setMessage(T message) { this.message = message; }
  
    public Usage getUsage() { return usage; }
    public void setUsage(Usage usage) { this.usage = usage; }
  
    public String getErrorMessage() { return errorMessage; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
  
    public Exception getException() { return exception; }
    public void setException(Exception exception) { this.exception = exception; }
  
    public Map<String, Object> getMetadata() { return metadata; }
    public void setMetadata(Map<String, Object> metadata) { this.metadata = metadata; }
  
    // 元数据操作
    public void addMetadata(String key, Object value) {
        this.metadata.put(key, value);
    }
  
    public Object getMetadata(String key) {
        return this.metadata.get(key);
    }
}

/**
 * AI消息响应
 */
public class AiMessageResponse extends AbstractBaseMessageResponse<AiMessage> {
  
    public AiMessageResponse() {
        super();
    }
  
    public AiMessageResponse(AiMessage message) {
        super();
        this.message = message;
    }
  
    // 便利方法
    public String getContent() {
        return message != null ? message.getContent() : null;
    }
  
    public boolean hasFunctionCalls() {
        return message != null && message.hasFunctionCalls();
    }
  
    public List<FunctionCall> getFunctionCalls() {
        return message != null ? message.getFunctionCalls() : null;
    }
  
    // 静态工厂方法
    public static AiMessageResponse success(AiMessage message) {
        return new AiMessageResponse(message);
    }
  
    public static AiMessageResponse error(String errorMessage) {
        AiMessageResponse response = new AiMessageResponse();
        response.setErrorMessage(errorMessage);
        return response;
    }
  
    public static AiMessageResponse error(Exception exception) {
        AiMessageResponse response = new AiMessageResponse();
        response.setException(exception);
        response.setErrorMessage(exception.getMessage());
        return response;
    }
}

/**
 * 使用统计
 */
public class Usage {
  
    private Integer promptTokens;
    private Integer completionTokens;
    private Integer totalTokens;
  
    public Usage() {}
  
    public Usage(Integer promptTokens, Integer completionTokens) {
        this.promptTokens = promptTokens;
        this.completionTokens = completionTokens;
        this.totalTokens = (promptTokens != null ? promptTokens : 0) + 
                          (completionTokens != null ? completionTokens : 0);
    }
  
    // Getters and Setters
    public Integer getPromptTokens() { return promptTokens; }
    public void setPromptTokens(Integer promptTokens) { 
        this.promptTokens = promptTokens;
        updateTotalTokens();
    }
  
    public Integer getCompletionTokens() { return completionTokens; }
    public void setCompletionTokens(Integer completionTokens) { 
        this.completionTokens = completionTokens;
        updateTotalTokens();
    }
  
    public Integer getTotalTokens() { return totalTokens; }
    public void setTotalTokens(Integer totalTokens) { this.totalTokens = totalTokens; }
  
    private void updateTotalTokens() {
        this.totalTokens = (promptTokens != null ? promptTokens : 0) + 
                          (completionTokens != null ? completionTokens : 0);
    }
  
    @Override
    public String toString() {
        return String.format("Usage{prompt=%d, completion=%d, total=%d}", 
                           promptTokens, completionTokens, totalTokens);
    }
}
```

## 核心特性

### 1. 简洁的API设计

Agents-Flex最大的特色是其简洁的API设计：

```java
// 最简单的使用方式
String response = llm.chat("你好");

// 稍微复杂一点
String response = llm.chat("写一个冒泡排序", ChatOptions.createForCode());

// 流式响应
llm.chatStream("讲一个故事", (context, response) -> {
    System.out.print(response.getMessage().getContent());
});
```

### 2. 强大的扩展能力

通过接口和抽象类的设计，可以轻松扩展新的LLM提供商：

```java
public class CustomLlm extends BaseLlm {
  
    public CustomLlm(LlmConfig config) {
        super(config);
    }
  
    @Override
    public AiMessageResponse chat(Prompt prompt, ChatOptions options) {
        // 实现自定义LLM的对话逻辑
        return processChat(prompt, options);
    }
  
    @Override
    public void chatStream(Prompt prompt, StreamResponseListener listener, 
                          ChatOptions options) {
        // 实现自定义LLM的流式对话逻辑
        processStreamChat(prompt, listener, options);
    }
}
```

### 3. 灵活的配置管理

支持多层次的配置选项：

```java
// 全局默认配置
ChatOptions globalDefaults = ChatOptions.create()
    .temperature(0.7)
    .maxTokens(2000);

// 特定用途配置
ChatOptions codeOptions = ChatOptions.createForCode();
ChatOptions creativeOptions = ChatOptions.createForCreative();

// 运行时动态配置
ChatOptions dynamicOptions = ChatOptions.create()
    .model("gpt-4")
    .temperature(userPreference.getTemperature())
    .systemMessage(userPreference.getSystemPrompt());
```

## 总结

Agents-Flex Core模块通过简洁而强大的设计，为Java LLM应用开发提供了优秀的基础框架。其核心特点包括：

**设计优势**：

- **简洁性**：最小化样板代码，开发体验优秀
- **灵活性**：高度可扩展的架构设计
- **类型安全**：强类型接口，编译时错误检查
- **易用性**：合理的默认配置和预设选项

**核心能力**：

- 统一的LLM抽象接口
- 完整的消息和提示词系统
- 灵活的配置管理
- 强大的响应处理机制

该模块为构建复杂的AI应用提供了坚实的基础，是Agents-Flex框架成功的关键所在。

---

*作者：senrian*
*最后更新：2024年*
