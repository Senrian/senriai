# Spring AI Alibaba - NL2SQL自然语言转SQL模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Spring AI Alibaba的NL2SQL模块是一个企业级的自然语言转SQL解决方案，将阿里云析言GBI（Generate Business Intelligence）的核心能力进行了模块化改造。该模块解决了传统SQL查询需要专业技能的痛点，让业务人员可以用自然语言直接查询数据库。

### 核心特性

- **智能理解**: 基于大模型的自然语言理解和关键词提取
- **Schema召回**: 使用向量检索技术精准匹配相关表结构
- **SQL生成**: 智能生成语义准确的SQL查询语句
- **执行引擎**: 支持SQL直接执行和结果格式化
- **错误修复**: 自动检测和修复SQL执行错误
- **业务逻辑**: 支持嵌入业务evidence提升准确性

## 2. 技术架构

### 整体流程架构

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  自然语言输入   │───▶│   意图理解与     │───▶│   关键词提取    │
│  "查询销售数据" │    │   需求分类       │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
          │                        │                       │
          ▼                        ▼                       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Schema召回    │    │   业务逻辑       │    │   时间表达式    │
│   (向量检索)    │    │   Evidence       │    │   解析处理      │
└─────────────────┘    └──────────────────┘    └─────────────────┘
          │                        │                       │
          └────────────────────────┼───────────────────────┘
                                   ▼
                        ┌──────────────────┐
                        │   SQL生成引擎    │
                        │   (LLM推理)      │
                        └──────────────────┘
                                   │
                                   ▼
                        ┌──────────────────┐    ┌─────────────────┐
                        │   SQL执行        │───▶│   结果格式化    │
                        │   与错误处理     │    │   (Markdown)    │
                        └──────────────────┘    └─────────────────┘
```

### 核心组件

1. **BaseNl2SqlService**: 核心业务逻辑服务
2. **SchemaRetrieval**: Schema召回组件
3. **EvidenceExtractor**: 业务逻辑提取器
4. **SqlGenerator**: SQL生成器
5. **SqlExecutor**: SQL执行引擎
6. **PromptHelper**: 提示词工程工具

## 3. 核心实现分析

### 3.1 意图识别与需求重写

```java
public String rewrite(String query) throws Exception {
    logger.info("Starting rewrite for query: {}", query);
    
    // 1. 提取业务逻辑evidence
    List<String> evidences = extractEvidences(query);
    
    // 2. Schema召回
    SchemaDTO schemaDTO = select(query, evidences);
    
    // 3. 构建重写提示词
    String prompt = PromptHelper.buildRewritePrompt(query, schemaDTO, evidences);
    
    // 4. LLM推理
    String responseContent = aiService.call(prompt);
    
    // 5. 解析响应，判断需求类型
    String[] splits = responseContent.split("\n");
    for (String line : splits) {
        if (line.startsWith("需求类型：")) {
            String content = line.substring(5).trim();
            if ("《自由闲聊》".equals(content)) {
                return SMALL_TALK_REJECT;  // 拒绝闲聊
            }
            else if ("《需要澄清》".equals(content)) {
                return INTENT_UNCLEAR;     // 需要澄清
            }
        }
        else if (line.startsWith("需求内容：")) {
            query = line.substring(5);     // 提取重写后的查询
        }
    }
    
    return query;
}
```

**设计亮点**:
- **智能分类**: 自动区分数据查询、闲聊、不明确需求
- **需求重写**: 将口语化表达转换为标准化查询语句
- **业务上下文**: 结合业务evidence提升理解准确性

### 3.2 Schema召回机制

```java
public SchemaDTO select(String query, List<String> evidences) throws Exception {
    logger.info("Starting schema selection for query: {}", query);
    
    // 1. 向量化查询
    List<Document> documents = vectorStore.similaritySearch(
        SearchRequest.query(query).withTopK(topK)
    );
    
    // 2. 解析匹配的Schema
    Set<String> tableNames = new HashSet<>();
    for (Document document : documents) {
        String content = document.getContent();
        SchemaInfo schemaInfo = parseSchemaFromDocument(content);
        tableNames.add(schemaInfo.getTableName());
    }
    
    // 3. 构建完整Schema DTO
    SchemaDTO schemaDTO = new SchemaDTO();
    for (String tableName : tableNames) {
        TableSchema tableSchema = databaseMetadataService.getTableSchema(tableName);
        schemaDTO.addTable(tableSchema);
    }
    
    // 4. 添加表关系信息
    addTableRelations(schemaDTO, tableNames);
    
    logger.info("Selected {} tables for schema", tableNames.size());
    return schemaDTO;
}
```

**技术特色**:
- **向量检索**: 使用embedding技术进行语义匹配
- **智能筛选**: 自动筛选最相关的表结构
- **关系推理**: 自动推断表间关系

### 3.3 SQL生成引擎

```java
public String generateSql(List<String> evidenceList, String query, SchemaDTO schemaDTO) 
        throws Exception {
    logger.info("Generating SQL for query: {}", query);
    
    // 1. 时间表达式处理
    String dateTimeExtractPrompt = PromptHelper.buildDateTimeExtractPrompt(query);
    String content = aiService.call(dateTimeExtractPrompt);
    List<String> dateTimeExpressions = extractDateTimeExpressions(content);
    
    // 2. 构建SQL生成提示词
    List<String> prompts = PromptHelper.buildMixSqlGeneratorPrompt(
        query, dbConfig, schemaDTO, evidenceList);
    
    // 3. 调用LLM生成SQL
    String sqlResponse = aiService.callWithSystemPrompt(
        prompts.get(0),  // System Prompt
        prompts.get(1)   // User Prompt
    );
    
    // 4. 解析并清理SQL
    String cleanedSql = MarkdownParser.extractRawText(sqlResponse).trim();
    
    // 5. SQL语法验证
    validateSqlSyntax(cleanedSql);
    
    logger.info("Generated SQL: {}", cleanedSql);
    return cleanedSql;
}
```

**核心机制**:
- **分段处理**: 时间表达式、业务逻辑分别处理
- **提示工程**: 专业的SQL生成提示词模板
- **语法验证**: 生成后自动验证SQL语法

### 3.4 错误修复机制

```java
public String generateSql(List<String> evidenceList, String query, SchemaDTO schemaDTO, 
                         String failedSql, String exceptionMessage) throws Exception {
    
    if (failedSql != null && !failedSql.isEmpty()) {
        // 使用专业的SQL错误修复提示词
        String errorFixerPrompt = PromptHelper.buildSqlErrorFixerPrompt(
            query, dbConfig, schemaDTO, evidenceList, failedSql, exceptionMessage);
        
        String fixedSql = aiService.call(errorFixerPrompt);
        
        logger.info("Fixed SQL from: {} to: {}", failedSql, fixedSql);
        return MarkdownParser.extractRawText(fixedSql).trim();
    }
    
    // 正常生成流程...
    return generateSql(evidenceList, query, schemaDTO);
}
```

## 4. 提示词工程体系

### 4.1 核心提示词模板

#### 需求重写提示词

```java
public static String buildRewritePrompt(String query, SchemaDTO schema, List<String> evidences) {
    return """
        你是一个专业的数据分析师，请分析用户的自然语言查询需求。
        
        数据库Schema：
        {schema}
        
        业务逻辑说明：
        {evidences}
        
        用户原始查询：{query}
        
        请按以下格式输出：
        需求类型：《数据查询》/《自由闲聊》/《需要澄清》
        需求内容：[如果是数据查询，请重写为标准化的查询描述]
        """.replace("{schema}", schema.toString())
           .replace("{evidences}", String.join("\n", evidences))
           .replace("{query}", query);
}
```

#### SQL生成提示词

```java
public static List<String> buildMixSqlGeneratorPrompt(String query, DbConfig dbConfig, 
                                                     SchemaDTO schema, List<String> evidences) {
    String systemPrompt = """
        你是一个专业的SQL开发工程师，擅长根据自然语言需求生成准确的SQL查询语句。
        
        数据库类型：{dbType}
        数据库Schema：
        {schema}
        
        业务规则：
        {evidences}
        
        请严格按照以下要求：
        1. 只生成SELECT查询语句
        2. 使用标准SQL语法
        3. 字段名和表名必须与Schema完全一致
        4. 合理使用JOIN连接多表
        5. 添加适当的WHERE条件
        """;
        
    String userPrompt = """
        请根据以下需求生成SQL查询：
        {query}
        
        请直接输出SQL语句，不需要解释：
        """;
        
    return Arrays.asList(
        systemPrompt.replace("{dbType}", dbConfig.getDialectType())
                   .replace("{schema}", schema.toString())
                   .replace("{evidences}", String.join("\n", evidences)),
        userPrompt.replace("{query}", query)
    );
}
```

### 4.2 错误修复提示词

```java
public static String buildSqlErrorFixerPrompt(String query, DbConfig dbConfig, 
                                            SchemaDTO schema, List<String> evidences,
                                            String failedSql, String errorMessage) {
    return """
        你是SQL错误修复专家，请修复以下SQL语句的错误。
        
        原始需求：{query}
        数据库类型：{dbType}
        失败的SQL：{failedSql}
        错误信息：{errorMessage}
        
        数据库Schema：
        {schema}
        
        常见错误类型：
        1. 字段名或表名不存在
        2. 语法错误
        3. 数据类型不匹配
        4. JOIN条件错误
        
        请输出修复后的SQL语句：
        """.replace("{query}", query)
           .replace("{dbType}", dbConfig.getDialectType())
           .replace("{failedSql}", failedSql)
           .replace("{errorMessage}", errorMessage)
           .replace("{schema}", schema.toString());
}
```

## 5. 执行引擎实现

### 5.1 SQL执行器

```java
@Service
public class SqlExecutionService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public QueryResult executeQuery(String sql, int maxRows) {
        try {
            logger.info("Executing SQL: {}", sql);
            
            // 1. SQL安全检查
            validateSqlSafety(sql);
            
            // 2. 添加LIMIT限制
            String limitedSql = addLimitClause(sql, maxRows);
            
            // 3. 执行查询
            List<Map<String, Object>> rows = jdbcTemplate.queryForList(limitedSql);
            
            // 4. 构建结果
            QueryResult result = new QueryResult();
            result.setSuccess(true);
            result.setRows(rows);
            result.setRowCount(rows.size());
            result.setExecutionTime(System.currentTimeMillis() - startTime);
            
            logger.info("Query executed successfully, {} rows returned", rows.size());
            return result;
            
        } catch (Exception e) {
            logger.error("SQL execution failed: {}", e.getMessage(), e);
            
            QueryResult result = new QueryResult();
            result.setSuccess(false);
            result.setErrorMessage(e.getMessage());
            return result;
        }
    }
    
    private void validateSqlSafety(String sql) {
        String upperSql = sql.toUpperCase().trim();
        
        // 只允许SELECT语句
        if (!upperSql.startsWith("SELECT")) {
            throw new SecurityException("Only SELECT statements are allowed");
        }
        
        // 禁止危险操作
        List<String> dangerousKeywords = Arrays.asList(
            "DELETE", "UPDATE", "INSERT", "DROP", "CREATE", "ALTER", "TRUNCATE"
        );
        
        for (String keyword : dangerousKeywords) {
            if (upperSql.contains(keyword)) {
                throw new SecurityException("Dangerous SQL keyword detected: " + keyword);
            }
        }
    }
}
```

### 5.2 结果格式化

```java
@Service 
public class ResultFormatterService {
    
    public String formatToMarkdown(QueryResult result) {
        if (!result.isSuccess()) {
            return "❌ **查询执行失败**\n\n错误信息：" + result.getErrorMessage();
        }
        
        List<Map<String, Object>> rows = result.getRows();
        if (rows.isEmpty()) {
            return "📭 **查询无结果**\n\n未找到匹配的数据。";
        }
        
        StringBuilder markdown = new StringBuilder();
        markdown.append("✅ **查询执行成功**\n\n");
        markdown.append(String.format("📊 共查询到 %d 条记录\n\n", rows.size()));
        
        // 构建表格
        if (!rows.isEmpty()) {
            // 表头
            Set<String> columns = rows.get(0).keySet();
            markdown.append("| ");
            columns.forEach(col -> markdown.append(col).append(" | "));
            markdown.append("\n");
            
            // 分隔线
            markdown.append("| ");
            columns.forEach(col -> markdown.append("--- | "));
            markdown.append("\n");
            
            // 数据行
            for (Map<String, Object> row : rows) {
                markdown.append("| ");
                for (String column : columns) {
                    Object value = row.get(column);
                    String cellValue = value != null ? value.toString() : "";
                    markdown.append(cellValue).append(" | ");
                }
                markdown.append("\n");
            }
        }
        
        markdown.append(String.format("\n⏱️ 执行时间：%d ms", result.getExecutionTime()));
        
        return markdown.toString();
    }
}
```

## 6. 配置与集成

### 6.1 数据库配置

```yaml
# application.yml
chatBi:
  dbConfig:
    url: jdbc:mysql://localhost:3306/business_db?useUnicode=true&characterEncoding=utf8
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    schema: business_schema
    connection-type: jdbc
    dialect-type: mysql
    
  # 向量存储配置
  vectorstore:
    analytic:
      enabled: true
      collectName: nl2sql_schema
      defaultTopK: 6
      defaultSimilarityThreshold: 0.75
      
  # 执行限制
  execution:
    maxRows: 1000
    timeoutSeconds: 30
    enableCache: true
    cacheExpireMinutes: 10
```

### 6.2 Schema向量化

```java
@Service
public class SchemaVectorService {
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private EmbeddingClient embeddingClient;
    
    public void indexDatabaseSchema() {
        logger.info("Starting database schema indexing");
        
        // 1. 获取所有表信息
        List<TableInfo> tables = databaseMetadataService.getAllTables();
        
        // 2. 为每个表生成向量文档
        List<Document> documents = new ArrayList<>();
        for (TableInfo table : tables) {
            String description = buildTableDescription(table);
            
            Document document = new Document(description);
            document.getMetadata().put("table_name", table.getTableName());
            document.getMetadata().put("schema_type", "table");
            
            documents.add(document);
        }
        
        // 3. 批量存储到向量库
        vectorStore.add(documents);
        
        logger.info("Indexed {} tables to vector store", tables.size());
    }
    
    private String buildTableDescription(TableInfo table) {
        StringBuilder desc = new StringBuilder();
        desc.append("表名：").append(table.getTableName()).append("\n");
        desc.append("表注释：").append(table.getComment()).append("\n");
        desc.append("字段信息：\n");
        
        for (ColumnInfo column : table.getColumns()) {
            desc.append("- ").append(column.getColumnName())
                .append("(").append(column.getDataType()).append(") ")
                .append(column.getComment()).append("\n");
        }
        
        return desc.toString();
    }
}
```

## 7. API接口设计

### 7.1 RESTful接口

```java
@RestController
@RequestMapping("/nl2sql")
public class Nl2SqlController {
    
    @Autowired
    private BaseNl2SqlService nl2SqlService;
    
    // 简单查询接口
    @GetMapping("/search")
    public ResponseEntity<QueryResponse> search(@RequestParam String query) {
        try {
            // 1. 需求重写
            String rewrittenQuery = nl2SqlService.rewrite(query);
            
            if (SMALL_TALK_REJECT.equals(rewrittenQuery)) {
                return ResponseEntity.ok(QueryResponse.reject("请输入数据查询相关的问题"));
            }
            
            // 2. 生成SQL
            String sql = nl2SqlService.nl2sql(rewrittenQuery);
            
            // 3. 执行查询
            QueryResult result = sqlExecutionService.executeQuery(sql, 100);
            
            // 4. 格式化结果
            String formattedResult = resultFormatterService.formatToMarkdown(result);
            
            return ResponseEntity.ok(QueryResponse.success(formattedResult, sql));
            
        } catch (Exception e) {
            logger.error("Query processing failed", e);
            return ResponseEntity.ok(QueryResponse.error(e.getMessage()));
        }
    }
    
    // 流式查询接口
    @GetMapping("/stream/search")
    public SseEmitter streamSearch(@RequestParam String query) {
        SseEmitter emitter = new SseEmitter(30000L);
        
        CompletableFuture.runAsync(() -> {
            try {
                // 发送处理状态
                emitter.send("🔄 正在理解查询需求...");
                String rewrittenQuery = nl2SqlService.rewrite(query);
                
                emitter.send("🔍 正在分析数据库结构...");
                String sql = nl2SqlService.nl2sql(rewrittenQuery);
                
                emitter.send("⚡ 正在执行SQL查询...");
                QueryResult result = sqlExecutionService.executeQuery(sql, 100);
                
                emitter.send("📊 正在格式化结果...");
                String formattedResult = resultFormatterService.formatToMarkdown(result);
                
                // 发送最终结果
                emitter.send(formattedResult);
                emitter.complete();
                
            } catch (Exception e) {
                try {
                    emitter.send("❌ 查询处理失败：" + e.getMessage());
                    emitter.complete();
                } catch (IOException ioException) {
                    emitter.completeWithError(ioException);
                }
            }
        });
        
        return emitter;
    }
}
```

### 7.2 响应模型

```java
@Data
public class QueryResponse {
    private boolean success;
    private String message;
    private String sql;
    private Object data;
    private String errorCode;
    private long timestamp;
    
    public static QueryResponse success(String result, String sql) {
        QueryResponse response = new QueryResponse();
        response.setSuccess(true);
        response.setMessage(result);
        response.setSql(sql);
        response.setTimestamp(System.currentTimeMillis());
        return response;
    }
    
    public static QueryResponse error(String errorMessage) {
        QueryResponse response = new QueryResponse();
        response.setSuccess(false);
        response.setMessage(errorMessage);
        response.setTimestamp(System.currentTimeMillis());
        return response;
    }
    
    public static QueryResponse reject(String rejectMessage) {
        QueryResponse response = new QueryResponse();
        response.setSuccess(false);
        response.setMessage(rejectMessage);
        response.setErrorCode("INTENT_REJECTED");
        response.setTimestamp(System.currentTimeMillis());
        return response;
    }
}
```

## 8. 高级特性

### 8.1 多数据源支持

```java
@Configuration
public class MultiDataSourceConfig {
    
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://localhost:3306/primary_db")
            .build();
    }
    
    @Bean
    public DataSource analyticsDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://localhost:5432/analytics_db")
            .build();
    }
    
    @Bean
    public DataSourceRouter dataSourceRouter() {
        DataSourceRouter router = new DataSourceRouter();
        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put("primary", primaryDataSource());
        dataSources.put("analytics", analyticsDataSource());
        
        router.setTargetDataSources(dataSources);
        router.setDefaultTargetDataSource(primaryDataSource());
        return router;
    }
}
```

### 8.2 查询优化

```java
@Component
public class QueryOptimizer {
    
    public String optimizeQuery(String sql, SchemaDTO schema) {
        // 1. 添加索引提示
        sql = addIndexHints(sql, schema);
        
        // 2. 优化JOIN顺序
        sql = optimizeJoinOrder(sql, schema);
        
        // 3. 添加查询限制
        sql = addQueryLimits(sql);
        
        return sql;
    }
    
    private String addIndexHints(String sql, SchemaDTO schema) {
        // 分析WHERE条件，建议使用索引
        for (TableSchema table : schema.getTables()) {
            for (IndexInfo index : table.getIndexes()) {
                if (sql.contains(index.getColumnName())) {
                    // 添加索引提示
                    sql = sql.replace(
                        table.getTableName(),
                        table.getTableName() + " USE INDEX(" + index.getIndexName() + ")"
                    );
                }
            }
        }
        return sql;
    }
}
```

### 8.3 权限控制

```java
@Component
public class QueryPermissionService {
    
    public boolean hasPermission(String userId, String sql, SchemaDTO schema) {
        // 1. 检查用户对表的访问权限
        for (TableSchema table : schema.getTables()) {
            if (!hasTablePermission(userId, table.getTableName())) {
                return false;
            }
        }
        
        // 2. 检查字段级权限
        List<String> columns = extractColumns(sql);
        for (String column : columns) {
            if (!hasColumnPermission(userId, column)) {
                return false;
            }
        }
        
        // 3. 检查数据行级权限
        return hasRowLevelPermission(userId, sql);
    }
    
    public String applyDataMasking(String sql, String userId) {
        // 对敏感字段应用数据脱敏
        if (isSensitiveQuery(sql)) {
            return applySensitiveDataMasking(sql, userId);
        }
        return sql;
    }
}
```

## 9. 监控与运维

### 9.1 查询统计

```java
@Component
public class QueryMetrics {
    
    private final Counter queryCounter;
    private final Timer queryTimer;
    private final Gauge activeQueries;
    
    public QueryMetrics(MeterRegistry meterRegistry) {
        this.queryCounter = Counter.builder("nl2sql.queries.total")
            .description("Total NL2SQL queries")
            .register(meterRegistry);
            
        this.queryTimer = Timer.builder("nl2sql.query.duration")
            .description("Query processing duration")
            .register(meterRegistry);
            
        this.activeQueries = Gauge.builder("nl2sql.queries.active")
            .description("Active queries count")
            .register(meterRegistry, this, QueryMetrics::getActiveQueryCount);
    }
    
    public void recordQuery(String queryType, Duration duration, boolean success) {
        queryCounter.increment(
            Tags.of(
                "type", queryType,
                "status", success ? "success" : "failure"
            )
        );
        
        queryTimer.record(duration, 
            Tags.of("type", queryType)
        );
    }
}
```

### 9.2 异常监控

```java
@Component
public class Nl2SqlHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // 检查数据库连接
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            
            // 检查向量存储
            vectorStore.similaritySearch(SearchRequest.query("test").withTopK(1));
            
            // 检查AI服务
            aiService.call("测试连接");
            
            return Health.up()
                .withDetail("database", "connected")
                .withDetail("vectorStore", "available")
                .withDetail("aiService", "available")
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

### 核心能力借鉴

1. **自然语言理解**: 结合业务上下文的智能query理解
2. **向量检索**: 使用embedding技术进行schema匹配
3. **提示工程**: 专业化的提示词模板体系
4. **错误恢复**: 智能的SQL错误检测和修复机制
5. **安全控制**: 企业级的权限和安全检查

### Sen-AiFlow集成设计

```java
// Sen-AiFlow中的NL2SQL服务设计
@SenAiFlowComponent("nl2sql-service")
public class ReactiveNl2SqlService {
    
    public Mono<String> rewriteQuery(String naturalLanguage);
    public Flux<SchemaMatch> retrieveSchema(String query);
    public Mono<String> generateSql(String query, List<SchemaMatch> schemas);
    public Mono<QueryResult> executeQuery(String sql);
    public Mono<String> formatResult(QueryResult result);
    
    // 响应式流水线
    public Flux<ProcessingStep> processQueryPipeline(String naturalLanguage) {
        return rewriteQuery(naturalLanguage)
            .flatMapMany(query -> retrieveSchema(query))
            .collectList()
            .flatMap(schemas -> generateSql(naturalLanguage, schemas))
            .flatMap(this::executeQuery)
            .flatMapMany(result -> formatResult(result).flux());
    }
}
```

Spring AI Alibaba的NL2SQL模块展示了企业级自然语言查询系统的完整解决方案，其智能理解、向量检索、错误恢复等机制为Sen-AiFlow的数据查询能力提供了重要参考，特别是在提示工程和错误处理方面的最佳实践。 