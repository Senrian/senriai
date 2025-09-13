# Spring AI Alibaba - Prompt提示管理模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Spring AI Alibaba的Prompt管理模块提供了动态、可配置的提示模板管理能力，支持从外部配置中心（如Nacos）动态加载和更新提示模板。该模块解决了传统AI应用中提示模板硬编码、难以动态调整的问题。

### 核心特性

- **动态配置**: 支持从Nacos配置中心动态加载提示模板
- **模板缓存**: 内存缓存机制，提高模板访问效率
- **实时更新**: 监听配置变化，自动更新模板内容
- **变量替换**: 支持模板变量动态替换
- **多种创建方式**: 支持从字符串、资源文件等多种方式创建模板

## 2. 核心架构

### 架构组件图

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Nacos配置中心     │───▶│ ConfigurablePrompt   │───▶│   PromptTemplate    │
│                     │    │ TemplateFactory      │    │                     │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
                                       │
                                       ▼
                           ┌──────────────────────┐
                           │ ConfigurablePrompt   │
                           │ Template             │
                           └──────────────────────┘
```

### 核心类结构

1. **ConfigurablePromptTemplateFactory**: 提示模板工厂
2. **ConfigurablePromptTemplate**: 可配置的提示模板
3. **PromptTemplateBuilderConfigure**: 模板构建器配置
4. **PromptTemplateCustomizer**: 模板自定义器

## 3. 核心实现分析

### 3.1 ConfigurablePromptTemplateFactory

```java
public class ConfigurablePromptTemplateFactory {
    private final Map<String, ConfigurablePromptTemplate> templates = new ConcurrentHashMap<>();
    private final PromptTemplateBuilderConfigure promptTemplateBuilderConfigure;

    // 多种创建方式
    public ConfigurablePromptTemplate create(String name, Resource resource);
    public ConfigurablePromptTemplate create(String name, String template);
    public ConfigurablePromptTemplate create(String name, String template, Map<String, Object> model);
    
    // Nacos配置监听器
    @NacosConfigListener(dataId = "spring.ai.alibaba.configurable.prompt", 
                        group = "DEFAULT_GROUP", 
                        initNotify = true)
    protected void onConfigChange(List<ConfigurablePromptTemplateModel> configList);
}
```

**特性分析**:
- **缓存机制**: 使用`ConcurrentHashMap`缓存模板，避免重复创建
- **动态监听**: 通过`@NacosConfigListener`注解监听配置变化
- **多重载**: 提供多种create方法，支持不同的模板创建方式

### 3.2 ConfigurablePromptTemplate

```java
public class ConfigurablePromptTemplate implements PromptTemplateActions, PromptTemplateMessageActions {
    private final PromptTemplate promptTemplate;
    private final String name;

    // 实现PromptTemplateActions接口
    public Prompt create();
    public Prompt create(ChatOptions modelOptions);
    public Prompt create(Map<String, Object> model);
    
    // 实现PromptTemplateMessageActions接口
    public Message createMessage();
    public Message createMessage(Map<String, Object> model);
    public String render(Map<String, Object> model);
}
```

**设计优势**:
- **接口实现**: 实现标准的Spring AI接口，保证兼容性
- **组合模式**: 内部组合`PromptTemplate`，提供装饰器功能
- **统一命名**: 每个模板都有唯一名称标识

### 3.3 动态配置机制

```java
@NacosConfigListener(dataId = "spring.ai.alibaba.configurable.prompt", 
                    group = "DEFAULT_GROUP", 
                    initNotify = true)
protected void onConfigChange(List<ConfigurablePromptTemplateModel> configList) {
    for (ConfigurablePromptTemplateModel configuration : configList) {
        if (!StringUtils.hasText(configuration.name()) || 
            !StringUtils.hasText(configuration.template())) {
            continue;
        }
        
        PromptTemplate.Builder promptTemplateBuilder = promptTemplateBuilderConfigure
            .configure(PromptTemplate.builder()
                .template(configuration.template())
                .variables(configuration.model()));

        templates.put(configuration.name(),
                new ConfigurablePromptTemplate(configuration.name(), promptTemplateBuilder.build()));
    }
}
```

**核心机制**:
- **实时监听**: 配置变更时自动触发更新
- **批量处理**: 支持一次更新多个模板
- **灵活配置**: 支持模板内容和变量的动态配置

## 4. 使用场景

### 4.1 RAG系统提示模板

```yaml
# Nacos配置示例
spring:
  ai:
    alibaba:
      configurable:
        prompt:
          - name: "rag-system-prompt"
            template: |
              你是一个专业的AI助手，请基于以下上下文信息回答用户问题：
              
              上下文：{context}
              
              问题：{question}
              
              请确保答案准确、简洁，如果上下文中没有相关信息，请明确说明。
            model:
              context: ""
              question: ""
```

### 4.2 多轮对话提示模板

```yaml
- name: "multi-turn-chat"
  template: |
    你是{name}，一个{role}的AI助手。
    
    对话历史：
    {history}
    
    用户：{user_input}
    助手：
  model:
    name: "小明"
    role: "友善"
    history: ""
    user_input: ""
```

### 4.3 代码示例

```java
@Service
public class PromptService {
    
    @Autowired
    private ConfigurablePromptTemplateFactory factory;
    
    public String generateResponse(String context, String question) {
        ConfigurablePromptTemplate template = factory.getTemplate("rag-system-prompt");
        
        Map<String, Object> variables = Map.of(
            "context", context,
            "question", question
        );
        
        return template.render(variables);
    }
}
```

## 5. 配置与集成

### 5.1 Maven依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-prompt-nacos</artifactId>
    <version>${spring-ai-alibaba.version}</version>
</dependency>
```

### 5.2 配置文件

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        namespace: spring-ai
        group: DEFAULT_GROUP
        
  ai:
    alibaba:
      configurable:
        prompt:
          enabled: true
          data-id: spring.ai.alibaba.configurable.prompt
          group: DEFAULT_GROUP
```

### 5.3 自定义配置器

```java
@Component
public class CustomPromptTemplateCustomizer implements PromptTemplateCustomizer {
    
    @Override
    public PromptTemplate.Builder customize(PromptTemplate.Builder builder) {
        return builder
            .defaultVariable("timestamp", System.currentTimeMillis())
            .defaultVariable("version", "1.0");
    }
}
```

## 6. 高级特性

### 6.1 模板继承与复用

```yaml
# 基础模板
- name: "base-system-prompt"
  template: |
    你是一个AI助手，请遵循以下原则：
    1. 准确回答问题
    2. 保持友善态度
    3. 承认知识局限性

# 专业模板（继承基础模板）
- name: "technical-assistant"
  template: |
    {base-system-prompt}
    
    你专门处理技术问题，擅长：
    - 编程语言
    - 系统设计  
    - 架构分析
```

### 6.2 条件模板

```java
public class ConditionalPromptTemplate extends ConfigurablePromptTemplate {
    
    @Override
    public String render(Map<String, Object> model) {
        String userType = (String) model.get("userType");
        
        if ("expert".equals(userType)) {
            return factory.getTemplate("expert-prompt").render(model);
        } else {
            return factory.getTemplate("basic-prompt").render(model);
        }
    }
}
```

## 7. 性能优化

### 7.1 缓存策略

```java
@Component
public class PromptCacheManager {
    
    private final LoadingCache<String, String> renderedPromptCache = 
        Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(30))
            .build(this::renderPrompt);
            
    public String getCachedPrompt(String templateName, Map<String, Object> variables) {
        String cacheKey = templateName + "-" + variables.hashCode();
        return renderedPromptCache.get(cacheKey);
    }
}
```

### 7.2 异步更新

```java
@Component
public class AsyncPromptUpdater {
    
    @Async
    @EventListener
    public void handlePromptConfigChange(PromptConfigChangeEvent event) {
        // 异步处理模板更新
        factory.refreshTemplate(event.getTemplateName());
        
        // 预热常用模板
        preloadFrequentTemplates();
    }
}
```

## 8. 最佳实践

### 8.1 模板命名规范

```
{domain}-{type}-{version}
例如：
- rag-system-v1
- chat-multi-turn-v2  
- code-generation-v1
```

### 8.2 变量命名约定

```yaml
# 推荐的变量命名
template: |
  用户信息：{user_profile}
  查询内容：{query_text}
  上下文：{context_documents}
  时间戳：{current_timestamp}
```

### 8.3 错误处理

```java
@Component
public class SafePromptRenderer {
    
    public String safeRender(String templateName, Map<String, Object> variables) {
        try {
            ConfigurablePromptTemplate template = factory.getTemplate(templateName);
            return template != null ? template.render(variables) : getDefaultPrompt();
        } catch (Exception e) {
            log.error("Failed to render template: {}", templateName, e);
            return getFallbackPrompt(templateName);
        }
    }
}
```

## 9. 监控与诊断

### 9.1 模板使用统计

```java
@Component
public class PromptMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter templateUsageCounter;
    
    public void recordTemplateUsage(String templateName) {
        templateUsageCounter.increment(
            Tags.of("template", templateName)
        );
    }
}
```

### 9.2 健康检查

```java
@Component
public class PromptHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // 检查关键模板是否可用
            factory.getTemplate("system-prompt");
            return Health.up()
                .withDetail("templates", factory.getTemplateCount())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

## 10. 对Sen-AiFlow的启发

### 设计借鉴

1. **动态配置管理**: 实现提示模板的外部化配置
2. **模板缓存机制**: 提高模板访问性能
3. **实时更新能力**: 支持配置的热更新
4. **统一接口设计**: 保证模板系统的可扩展性

### 实现建议

```java
// Sen-AiFlow中的提示管理器设计
@SenAiFlowComponent("prompt-manager")
public class ReactivePromptManager {
    
    public Mono<PromptTemplate> getTemplate(String name);
    public Flux<PromptTemplate> getAllTemplates();
    public Mono<Void> updateTemplate(String name, String content);
    public Flux<PromptUpdateEvent> subscribeToUpdates();
}
```

Spring AI Alibaba的Prompt管理模块展示了企业级AI应用中提示模板管理的最佳实践，其动态配置、缓存优化和实时更新机制为Sen-AiFlow的提示管理系统提供了重要参考。 