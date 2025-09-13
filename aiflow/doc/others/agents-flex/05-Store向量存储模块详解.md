# Agents-Flex - Store向量存储模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Agents-Flex的Store模块提供了强大的向量存储能力，支持多种主流向量数据库，是构建RAG（检索增强生成）系统的核心基础设施。该模块通过统一的抽象接口，屏蔽了不同向量数据库的实现差异，让开发者可以轻松切换存储后端。

### 核心特性

- **多数据库支持**: 支持Milvus、Qdrant、Redis、Elasticsearch、Chroma等主流向量数据库
- **统一抽象**: 提供一致的API接口，屏蔽底层实现差异
- **云服务集成**: 支持阿里云、腾讯云等云厂商的向量数据库服务
- **索引优化**: 支持多种索引算法和优化策略
- **批量操作**: 高效的批量存储和检索操作
- **元数据管理**: 丰富的元数据存储和筛选能力
- **相似性搜索**: 多种相似性度量和搜索策略

## 2. 核心架构

### 向量存储架构

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Vector Document   │───▶│   Vector Store       │───▶│   Similarity Search │
│   (向量文档)        │    │   (向量存储)         │    │   (相似性搜索)      │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          ▼                            ▼                           ▼
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Embedding Vector  │    │   Index Manager      │    │   Search Result     │
│   (嵌入向量)        │    │   (索引管理)         │    │   (搜索结果)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          └────────────────────────────┼───────────────────────────┘
                                       ▼
                           ┌──────────────────────┐
                           │   Metadata Filter    │
                           │   (元数据过滤)       │
                           └──────────────────────┘
```

### 数据库支持矩阵

```
VectorStore (统一接口)
├── CloudVectorStores (云服务)
│   ├── AliyunVectorStore (阿里云向量检索服务)
│   ├── QCloudVectorStore (腾讯云向量数据库)
│   └── VectorexDBVectorStore (VectorexDB)
├── OpenSourceVectorStores (开源数据库)
│   ├── MilvusVectorStore (Milvus)
│   ├── QdrantVectorStore (Qdrant)
│   ├── ChromaVectorStore (Chroma)
│   └── ElasticsearchVectorStore (Elasticsearch)
├── InMemoryVectorStores (内存存储)
│   ├── RedisVectorStore (Redis)
│   └── PgVectorStore (PostgreSQL + pgvector)
└── HybridVectorStores (混合存储)
    └── OpenSearchVectorStore (OpenSearch)
```

## 3. 核心实现分析

### 3.1 VectorStore统一接口

```java
public interface VectorStore {
    
    /**
     * 添加向量文档
     * @param documents 向量文档列表
     */
    void add(List<VectorDocument> documents);
    
    /**
     * 删除向量文档
     * @param ids 文档ID列表
     * @return 删除的文档数量
     */
    long delete(List<String> ids);
    
    /**
     * 更新向量文档
     * @param documents 要更新的文档列表
     */
    void update(List<VectorDocument> documents);
    
    /**
     * 相似性搜索
     * @param query 查询向量
     * @param topK 返回的Top-K结果数量
     * @return 搜索结果列表
     */
    List<VectorDocument> similaritySearch(float[] query, int topK);
    
    /**
     * 带过滤条件的相似性搜索
     * @param query 查询向量
     * @param topK 返回的Top-K结果数量
     * @param filter 过滤条件
     * @return 搜索结果列表
     */
    List<VectorDocument> similaritySearch(float[] query, int topK, VectorFilter filter);
    
    /**
     * 范围搜索
     * @param query 查询向量
     * @param threshold 相似度阈值
     * @return 搜索结果列表
     */
    List<VectorDocument> rangeSearch(float[] query, double threshold);
    
    /**
     * 获取向量存储统计信息
     * @return 统计信息
     */
    VectorStoreStats getStats();
}
```

**设计特点**:
- **统一接口**: 所有向量数据库都实现相同的接口
- **批量操作**: 支持批量增删改查操作
- **灵活查询**: 支持多种搜索模式
- **元数据过滤**: 基于元数据的条件过滤

### 3.2 VectorDocument数据模型

```java
public class VectorDocument {
    private String id;                          // 文档唯一标识
    private float[] vector;                     // 向量数据
    private String content;                     // 原始内容
    private Map<String, Object> metadata;      // 元数据
    private Double score;                       // 相似度分数
    
    public VectorDocument(String id, float[] vector, String content) {
        this.id = id;
        this.vector = vector;
        this.content = content;
        this.metadata = new HashMap<>();
    }
    
    // 添加元数据
    public VectorDocument addMetadata(String key, Object value) {
        this.metadata.put(key, value);
        return this;
    }
    
    // 获取元数据
    public <T> T getMetadata(String key, Class<T> type) {
        Object value = metadata.get(key);
        return value != null ? type.cast(value) : null;
    }
    
    // 计算与另一个向量的相似度
    public double similarity(VectorDocument other, SimilarityFunction function) {
        return function.calculate(this.vector, other.vector);
    }
    
    // 文档指纹（用于去重）
    public String getFingerprint() {
        return DigestUtils.md5Hex(content + vector.toString() + id);
    }
}
```

### 3.3 Milvus向量存储实现

```java
public class MilvusVectorStore implements VectorStore {
    
    private final MilvusServiceClient client;
    private final String collectionName;
    private final int dimension;
    private final IndexType indexType;
    private final MetricType metricType;
    
    public MilvusVectorStore(MilvusConfig config) {
        this.client = new MilvusServiceClient(
            ConnectParam.newBuilder()
                .withHost(config.getHost())
                .withPort(config.getPort())
                .withAuthorization(config.getUsername(), config.getPassword())
                .build()
        );
        
        this.collectionName = config.getCollectionName();
        this.dimension = config.getDimension();
        this.indexType = config.getIndexType();
        this.metricType = config.getMetricType();
        
        // 初始化集合
        initializeCollection();
    }
    
    @Override
    public void add(List<VectorDocument> documents) {
        if (documents.isEmpty()) {
            return;
        }
        
        // 构建插入数据
        List<InsertParam.Field> fields = new ArrayList<>();
        
        // ID字段
        List<String> ids = documents.stream()
            .map(VectorDocument::getId)
            .collect(Collectors.toList());
        fields.add(new InsertParam.Field("id", ids));
        
        // 向量字段
        List<List<Float>> vectors = documents.stream()
            .map(doc -> {
                float[] vector = doc.getVector();
                return Arrays.stream(vector)
                    .boxed()
                    .collect(Collectors.toList());
            })
            .collect(Collectors.toList());
        fields.add(new InsertParam.Field("vector", vectors));
        
        // 内容字段
        List<String> contents = documents.stream()
            .map(VectorDocument::getContent)
            .collect(Collectors.toList());
        fields.add(new InsertParam.Field("content", contents));
        
        // 元数据字段
        addMetadataFields(fields, documents);
        
        // 执行插入
        InsertParam insertParam = InsertParam.newBuilder()
            .withCollectionName(collectionName)
            .withFields(fields)
            .build();
            
        R<MutationResult> response = client.insert(insertParam);
        
        if (response.getStatus() != R.Status.Success.getCode()) {
            throw new VectorStoreException("Failed to insert documents: " + response.getMessage());
        }
        
        logger.info("Successfully inserted {} documents into Milvus", documents.size());
    }
    
    @Override
    public List<VectorDocument> similaritySearch(float[] query, int topK, VectorFilter filter) {
        // 构建搜索参数
        SearchParam.Builder searchBuilder = SearchParam.newBuilder()
            .withCollectionName(collectionName)
            .withMetricType(metricType)
            .withTopK(topK)
            .withVectors(Collections.singletonList(Arrays.asList(
                ArrayUtils.toObject(query)
            )))
            .withVectorFieldName("vector");
        
        // 添加过滤条件
        if (filter != null) {
            String filterExpression = buildFilterExpression(filter);
            searchBuilder.withExpr(filterExpression);
        }
        
        // 设置搜索参数
        Map<String, Object> searchParams = new HashMap<>();
        searchParams.put("nprobe", 10); // IVFFLAT索引参数
        searchBuilder.withParams(JsonUtils.toJson(searchParams));
        
        // 执行搜索
        R<SearchResults> response = client.search(searchBuilder.build());
        
        if (response.getStatus() != R.Status.Success.getCode()) {
            throw new VectorStoreException("Search failed: " + response.getMessage());
        }
        
        // 解析搜索结果
        return parseSearchResults(response.getData());
    }
    
    private void initializeCollection() {
        // 检查集合是否存在
        R<Boolean> hasResponse = client.hasCollection(
            HasCollectionParam.newBuilder()
                .withCollectionName(collectionName)
                .build()
        );
        
        if (hasResponse.getData()) {
            logger.info("Collection {} already exists", collectionName);
            return;
        }
        
        // 创建集合
        List<FieldType> fields = Arrays.asList(
            // ID字段
            FieldType.newBuilder()
                .withName("id")
                .withDataType(DataType.VarChar)
                .withMaxLength(255)
                .withPrimaryKey(true)
                .build(),
            
            // 向量字段
            FieldType.newBuilder()
                .withName("vector")
                .withDataType(DataType.FloatVector)
                .withDimension(dimension)
                .build(),
            
            // 内容字段
            FieldType.newBuilder()
                .withName("content")
                .withDataType(DataType.VarChar)
                .withMaxLength(65535)
                .build()
        );
        
        CreateCollectionParam createParam = CreateCollectionParam.newBuilder()
            .withCollectionName(collectionName)
            .withDescription("Vector store collection for Agents-Flex")
            .withFieldTypes(fields)
            .build();
            
        R<RpcStatus> response = client.createCollection(createParam);
        
        if (response.getStatus() != R.Status.Success.getCode()) {
            throw new VectorStoreException("Failed to create collection: " + response.getMessage());
        }
        
        // 创建索引
        createIndex();
        
        // 加载集合到内存
        loadCollection();
        
        logger.info("Successfully created and loaded collection {}", collectionName);
    }
    
    private void createIndex() {
        // 构建索引参数
        Map<String, Object> indexParams = new HashMap<>();
        indexParams.put("M", 16);
        indexParams.put("efConstruction", 256);
        
        CreateIndexParam indexParam = CreateIndexParam.newBuilder()
            .withCollectionName(collectionName)
            .withFieldName("vector")
            .withIndexType(indexType)
            .withMetricType(metricType)
            .withExtraParam(JsonUtils.toJson(indexParams))
            .build();
            
        R<RpcStatus> response = client.createIndex(indexParam);
        
        if (response.getStatus() != R.Status.Success.getCode()) {
            throw new VectorStoreException("Failed to create index: " + response.getMessage());
        }
    }
    
    private String buildFilterExpression(VectorFilter filter) {
        StringBuilder expression = new StringBuilder();
        
        for (VectorFilter.Condition condition : filter.getConditions()) {
            if (expression.length() > 0) {
                expression.append(" and ");
            }
            
            String field = condition.getField();
            Object value = condition.getValue();
            VectorFilter.Operator operator = condition.getOperator();
            
            switch (operator) {
                case EQUAL:
                    expression.append(String.format("%s == '%s'", field, value));
                    break;
                case NOT_EQUAL:
                    expression.append(String.format("%s != '%s'", field, value));
                    break;
                case IN:
                    List<Object> values = (List<Object>) value;
                    String valueList = values.stream()
                        .map(v -> "'" + v + "'")
                        .collect(Collectors.joining(", "));
                    expression.append(String.format("%s in [%s]", field, valueList));
                    break;
                case GREATER_THAN:
                    expression.append(String.format("%s > %s", field, value));
                    break;
                case LESS_THAN:
                    expression.append(String.format("%s < %s", field, value));
                    break;
            }
        }
        
        return expression.toString();
    }
}
```

### 3.4 Redis向量存储实现

```java
public class RedisVectorStore implements VectorStore {
    
    private final JedisPool jedisPool;
    private final String indexName;
    private final int dimension;
    private final DistanceMetric metric;
    
    public RedisVectorStore(RedisConfig config) {
        this.jedisPool = new JedisPool(
            new JedisPoolConfig(),
            config.getHost(),
            config.getPort(),
            config.getTimeout(),
            config.getPassword()
        );
        
        this.indexName = config.getIndexName();
        this.dimension = config.getDimension();
        this.metric = config.getMetric();
        
        // 初始化索引
        initializeIndex();
    }
    
    @Override
    public void add(List<VectorDocument> documents) {
        try (Jedis jedis = jedisPool.getResource()) {
            Pipeline pipeline = jedis.pipelined();
            
            for (VectorDocument document : documents) {
                String key = getDocumentKey(document.getId());
                
                // 存储文档数据
                Map<String, String> documentData = new HashMap<>();
                documentData.put("content", document.getContent());
                documentData.put("vector", encodeVector(document.getVector()));
                
                // 添加元数据
                document.getMetadata().forEach((k, v) -> 
                    documentData.put("meta_" + k, v.toString()));
                
                pipeline.hset(key, documentData);
            }
            
            pipeline.sync();
            
            logger.info("Successfully added {} documents to Redis", documents.size());
        }
    }
    
    @Override
    public List<VectorDocument> similaritySearch(float[] query, int topK, VectorFilter filter) {
        try (Jedis jedis = jedisPool.getResource()) {
            // 构建查询
            Query queryBuilder = new Query("*");
            
            // 添加向量搜索
            queryBuilder.returnFields("content", "vector", "__score");
            queryBuilder.setSortBy("__score", true);
            queryBuilder.limit(0, topK);
            
            // 添加KNN搜索
            byte[] queryVector = encodeVectorBytes(query);
            queryBuilder.addFilter(
                new Query.VectorFilter("vector", queryVector)
                    .setSimilarityFunction(convertMetric(metric))
            );
            
            // 添加元数据过滤
            if (filter != null) {
                addFilterConditions(queryBuilder, filter);
            }
            
            // 执行搜索
            SearchResult result = jedis.ftSearch(indexName, queryBuilder);
            
            // 解析结果
            List<VectorDocument> documents = new ArrayList<>();
            for (Document doc : result.getDocuments()) {
                VectorDocument vectorDoc = parseDocument(doc);
                documents.add(vectorDoc);
            }
            
            return documents;
        }
    }
    
    private void initializeIndex() {
        try (Jedis jedis = jedisPool.getResource()) {
            // 检查索引是否存在
            try {
                jedis.ftInfo(indexName);
                logger.info("Index {} already exists", indexName);
                return;
            } catch (Exception e) {
                // 索引不存在，需要创建
            }
            
            // 创建索引
            Schema schema = new Schema()
                .addTextField("content", 1.0)
                .addVectorField("vector", 
                    Schema.VectorField.VectorAlgo.HNSW,
                    Map.of(
                        "TYPE", "FLOAT32",
                        "DIM", dimension,
                        "DISTANCE_METRIC", metric.name()
                    )
                );
            
            IndexDefinition definition = new IndexDefinition(IndexDefinition.Type.HASH)
                .setPrefixes(getKeyPrefix());
                
            jedis.ftCreate(indexName, IndexOptions.defaultOptions().setDefinition(definition), schema);
            
            logger.info("Successfully created Redis vector index {}", indexName);
        }
    }
    
    private String encodeVector(float[] vector) {
        return Arrays.stream(vector)
            .mapToObj(String::valueOf)
            .collect(Collectors.joining(","));
    }
    
    private float[] decodeVector(String vectorStr) {
        return Arrays.stream(vectorStr.split(","))
            .mapToDouble(Double::parseDouble)
            .collect(() -> new float[0], 
                    (list, value) -> {
                        float[] newArray = Arrays.copyOf(list, list.length + 1);
                        newArray[list.length] = (float) value;
                        return newArray;
                    },
                    (list1, list2) -> {
                        float[] combined = Arrays.copyOf(list1, list1.length + list2.length);
                        System.arraycopy(list2, 0, combined, list1.length, list2.length);
                        return combined;
                    });
    }
}
```

## 4. 相似性度量与搜索策略

### 4.1 相似性函数

```java
public enum SimilarityFunction {
    
    COSINE("cosine") {
        @Override
        public double calculate(float[] vector1, float[] vector2) {
            return cosineSimilarity(vector1, vector2);
        }
    },
    
    EUCLIDEAN("euclidean") {
        @Override
        public double calculate(float[] vector1, float[] vector2) {
            return 1.0 / (1.0 + euclideanDistance(vector1, vector2));
        }
    },
    
    DOT_PRODUCT("dot_product") {
        @Override
        public double calculate(float[] vector1, float[] vector2) {
            return dotProduct(vector1, vector2);
        }
    },
    
    MANHATTAN("manhattan") {
        @Override
        public double calculate(float[] vector1, float[] vector2) {
            return 1.0 / (1.0 + manhattanDistance(vector1, vector2));
        }
    };
    
    private final String name;
    
    SimilarityFunction(String name) {
        this.name = name;
    }
    
    public abstract double calculate(float[] vector1, float[] vector2);
    
    private static double cosineSimilarity(float[] vector1, float[] vector2) {
        double dotProduct = 0.0;
        double norm1 = 0.0;
        double norm2 = 0.0;
        
        for (int i = 0; i < vector1.length; i++) {
            dotProduct += vector1[i] * vector2[i];
            norm1 += vector1[i] * vector1[i];
            norm2 += vector2[i] * vector2[i];
        }
        
        return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
    
    private static double euclideanDistance(float[] vector1, float[] vector2) {
        double sum = 0.0;
        for (int i = 0; i < vector1.length; i++) {
            double diff = vector1[i] - vector2[i];
            sum += diff * diff;
        }
        return Math.sqrt(sum);
    }
    
    private static double dotProduct(float[] vector1, float[] vector2) {
        double sum = 0.0;
        for (int i = 0; i < vector1.length; i++) {
            sum += vector1[i] * vector2[i];
        }
        return sum;
    }
    
    private static double manhattanDistance(float[] vector1, float[] vector2) {
        double sum = 0.0;
        for (int i = 0; i < vector1.length; i++) {
            sum += Math.abs(vector1[i] - vector2[i]);
        }
        return sum;
    }
}
```

### 4.2 搜索策略

```java
public class SearchStrategy {
    
    /**
     * 精确搜索 - 返回最相似的结果
     */
    public static class ExactSearch implements SearchStrategy {
        private final int topK;
        private final double threshold;
        
        public ExactSearch(int topK, double threshold) {
            this.topK = topK;
            this.threshold = threshold;
        }
        
        @Override
        public List<VectorDocument> search(VectorStore store, float[] query, VectorFilter filter) {
            List<VectorDocument> results = store.similaritySearch(query, topK, filter);
            
            // 过滤低于阈值的结果
            return results.stream()
                .filter(doc -> doc.getScore() >= threshold)
                .collect(Collectors.toList());
        }
    }
    
    /**
     * 混合搜索 - 结合多种相似性度量
     */
    public static class HybridSearch implements SearchStrategy {
        private final Map<SimilarityFunction, Double> weights;
        private final int topK;
        
        public HybridSearch(Map<SimilarityFunction, Double> weights, int topK) {
            this.weights = weights;
            this.topK = topK;
        }
        
        @Override
        public List<VectorDocument> search(VectorStore store, float[] query, VectorFilter filter) {
            Map<String, Double> combinedScores = new HashMap<>();
            
            // 对每种相似性度量进行搜索
            for (Map.Entry<SimilarityFunction, Double> entry : weights.entrySet()) {
                SimilarityFunction function = entry.getKey();
                double weight = entry.getValue();
                
                // 使用特定相似性度量搜索
                List<VectorDocument> results = searchWithFunction(store, query, function, filter);
                
                // 合并分数
                for (VectorDocument doc : results) {
                    String docId = doc.getId();
                    double weightedScore = doc.getScore() * weight;
                    combinedScores.merge(docId, weightedScore, Double::sum);
                }
            }
            
            // 重新获取文档并排序
            return combinedScores.entrySet().stream()
                .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                .limit(topK)
                .map(entry -> store.getDocument(entry.getKey()))
                .filter(Objects::nonNull)
                .peek(doc -> doc.setScore(combinedScores.get(doc.getId())))
                .collect(Collectors.toList());
        }
    }
    
    /**
     * 多阶段搜索 - 粗筛 + 精排
     */
    public static class MultiStageSearch implements SearchStrategy {
        private final int coarseTopK;
        private final int fineTopK;
        private final SimilarityFunction coarseFunction;
        private final SimilarityFunction fineFunction;
        
        public MultiStageSearch(int coarseTopK, int fineTopK, 
                              SimilarityFunction coarseFunction, 
                              SimilarityFunction fineFunction) {
            this.coarseTopK = coarseTopK;
            this.fineTopK = fineTopK;
            this.coarseFunction = coarseFunction;
            this.fineFunction = fineFunction;
        }
        
        @Override
        public List<VectorDocument> search(VectorStore store, float[] query, VectorFilter filter) {
            // 第一阶段：粗筛
            List<VectorDocument> coarseResults = searchWithFunction(
                store, query, coarseFunction, filter, coarseTopK);
            
            // 第二阶段：精排
            List<VectorDocument> refinedResults = coarseResults.stream()
                .peek(doc -> {
                    double fineScore = fineFunction.calculate(query, doc.getVector());
                    doc.setScore(fineScore);
                })
                .sorted((a, b) -> Double.compare(b.getScore(), a.getScore()))
                .limit(fineTopK)
                .collect(Collectors.toList());
            
            return refinedResults;
        }
    }
}
```

## 5. 元数据过滤系统

### 5.1 VectorFilter过滤器

```java
public class VectorFilter {
    
    private final List<Condition> conditions = new ArrayList<>();
    private LogicalOperator logicalOperator = LogicalOperator.AND;
    
    public static VectorFilter create() {
        return new VectorFilter();
    }
    
    public VectorFilter eq(String field, Object value) {
        conditions.add(new Condition(field, Operator.EQUAL, value));
        return this;
    }
    
    public VectorFilter ne(String field, Object value) {
        conditions.add(new Condition(field, Operator.NOT_EQUAL, value));
        return this;
    }
    
    public VectorFilter in(String field, List<Object> values) {
        conditions.add(new Condition(field, Operator.IN, values));
        return this;
    }
    
    public VectorFilter gt(String field, Object value) {
        conditions.add(new Condition(field, Operator.GREATER_THAN, value));
        return this;
    }
    
    public VectorFilter lt(String field, Object value) {
        conditions.add(new Condition(field, Operator.LESS_THAN, value));
        return this;
    }
    
    public VectorFilter range(String field, Object min, Object max) {
        conditions.add(new Condition(field, Operator.RANGE, Arrays.asList(min, max)));
        return this;
    }
    
    public VectorFilter exists(String field) {
        conditions.add(new Condition(field, Operator.EXISTS, null));
        return this;
    }
    
    public VectorFilter and() {
        this.logicalOperator = LogicalOperator.AND;
        return this;
    }
    
    public VectorFilter or() {
        this.logicalOperator = LogicalOperator.OR;
        return this;
    }
    
    // 内部类
    public static class Condition {
        private final String field;
        private final Operator operator;
        private final Object value;
        
        public Condition(String field, Operator operator, Object value) {
            this.field = field;
            this.operator = operator;
            this.value = value;
        }
        
        // getters...
    }
    
    public enum Operator {
        EQUAL, NOT_EQUAL, IN, NOT_IN,
        GREATER_THAN, LESS_THAN, GREATER_EQUAL, LESS_EQUAL,
        RANGE, EXISTS, NOT_EXISTS,
        LIKE, NOT_LIKE
    }
    
    public enum LogicalOperator {
        AND, OR, NOT
    }
}

// 使用示例
VectorFilter filter = VectorFilter.create()
    .eq("category", "technology")
    .gt("publish_date", "2023-01-01")
    .in("author", Arrays.asList("Alice", "Bob"))
    .and();
```

### 5.2 智能过滤优化

```java
public class FilterOptimizer {
    
    /**
     * 优化过滤条件的执行顺序
     */
    public VectorFilter optimize(VectorFilter filter, VectorStoreStats stats) {
        List<VectorFilter.Condition> conditions = filter.getConditions();
        
        // 根据选择性排序条件
        List<VectorFilter.Condition> optimizedConditions = conditions.stream()
            .sorted((c1, c2) -> {
                double selectivity1 = calculateSelectivity(c1, stats);
                double selectivity2 = calculateSelectivity(c2, stats);
                return Double.compare(selectivity1, selectivity2);
            })
            .collect(Collectors.toList());
        
        // 构建优化后的过滤器
        VectorFilter optimized = VectorFilter.create();
        optimizedConditions.forEach(condition -> 
            optimized.addCondition(condition));
        
        return optimized;
    }
    
    /**
     * 计算条件的选择性
     */
    private double calculateSelectivity(VectorFilter.Condition condition, VectorStoreStats stats) {
        String field = condition.getField();
        VectorFilter.Operator operator = condition.getOperator();
        
        // 根据统计信息计算选择性
        FieldStats fieldStats = stats.getFieldStats(field);
        if (fieldStats == null) {
            return 0.5; // 默认选择性
        }
        
        switch (operator) {
            case EQUAL:
                return 1.0 / fieldStats.getDistinctValues();
            case IN:
                List<Object> values = (List<Object>) condition.getValue();
                return (double) values.size() / fieldStats.getDistinctValues();
            case GREATER_THAN:
            case LESS_THAN:
                return 0.3; // 估算选择性
            case EXISTS:
                return 1.0 - fieldStats.getNullRatio();
            default:
                return 0.5;
        }
    }
}
```

## 6. 批量操作与性能优化

### 6.1 批量处理器

```java
public class BatchProcessor {
    
    private final VectorStore vectorStore;
    private final int batchSize;
    private final ExecutorService executorService;
    
    public BatchProcessor(VectorStore vectorStore, int batchSize) {
        this.vectorStore = vectorStore;
        this.batchSize = batchSize;
        this.executorService = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors()
        );
    }
    
    /**
     * 批量添加文档
     */
    public CompletableFuture<BatchResult> addBatch(List<VectorDocument> documents) {
        return CompletableFuture.supplyAsync(() -> {
            List<List<VectorDocument>> batches = partition(documents, batchSize);
            List<CompletableFuture<Void>> futures = new ArrayList<>();
            
            for (List<VectorDocument> batch : batches) {
                CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                    try {
                        vectorStore.add(batch);
                    } catch (Exception e) {
                        logger.error("Failed to add batch", e);
                        throw new RuntimeException(e);
                    }
                }, executorService);
                
                futures.add(future);
            }
            
            // 等待所有批次完成
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            
            return BatchResult.success(documents.size());
        });
    }
    
    /**
     * 并行搜索
     */
    public CompletableFuture<List<VectorDocument>> parallelSearch(
            List<float[]> queries, int topK, VectorFilter filter) {
        
        List<CompletableFuture<List<VectorDocument>>> futures = queries.stream()
            .map(query -> CompletableFuture.supplyAsync(() -> 
                vectorStore.similaritySearch(query, topK, filter), executorService))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .flatMap(future -> future.join().stream())
                .collect(Collectors.toList()));
    }
    
    private <T> List<List<T>> partition(List<T> list, int size) {
        List<List<T>> partitions = new ArrayList<>();
        for (int i = 0; i < list.size(); i += size) {
            partitions.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return partitions;
    }
}
```

### 6.2 连接池管理

```java
public class VectorStoreConnectionPool {
    
    private final Map<String, ObjectPool<VectorStore>> pools = new ConcurrentHashMap<>();
    
    public <T extends VectorStore> ObjectPool<T> getPool(String name, 
                                                        Supplier<T> factory,
                                                        PoolConfig config) {
        return (ObjectPool<T>) pools.computeIfAbsent(name, k -> 
            createPool(factory, config));
    }
    
    private <T extends VectorStore> ObjectPool<T> createPool(Supplier<T> factory, PoolConfig config) {
        GenericObjectPoolConfig<T> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(config.getMaxTotal());
        poolConfig.setMaxIdle(config.getMaxIdle());
        poolConfig.setMinIdle(config.getMinIdle());
        poolConfig.setMaxWaitMillis(config.getMaxWaitMillis());
        
        PooledObjectFactory<T> objectFactory = new BasePooledObjectFactory<T>() {
            @Override
            public T create() {
                return factory.get();
            }
            
            @Override
            public PooledObject<T> wrap(T obj) {
                return new DefaultPooledObject<>(obj);
            }
            
            @Override
            public boolean validateObject(PooledObject<T> p) {
                try {
                    VectorStoreStats stats = p.getObject().getStats();
                    return stats != null;
                } catch (Exception e) {
                    return false;
                }
            }
        };
        
        return new GenericObjectPool<>(objectFactory, poolConfig);
    }
}
```

## 7. 索引管理与优化

### 7.1 索引策略

```java
public class IndexManager {
    
    private final VectorStore vectorStore;
    private final IndexConfig config;
    
    public IndexManager(VectorStore vectorStore, IndexConfig config) {
        this.vectorStore = vectorStore;
        this.config = config;
    }
    
    /**
     * 创建索引
     */
    public void createIndex(String fieldName, IndexType indexType) {
        switch (indexType) {
            case HNSW:
                createHNSWIndex(fieldName);
                break;
            case IVF_FLAT:
                createIVFFlatIndex(fieldName);
                break;
            case IVF_PQ:
                createIVFPQIndex(fieldName);
                break;
            case ANNOY:
                createAnnoyIndex(fieldName);
                break;
        }
    }
    
    private void createHNSWIndex(String fieldName) {
        HNSWIndexParams params = HNSWIndexParams.builder()
            .M(config.getHnswM())
            .efConstruction(config.getHnswEfConstruction())
            .efSearch(config.getHnswEfSearch())
            .build();
            
        vectorStore.createIndex(fieldName, IndexType.HNSW, params);
        logger.info("Created HNSW index for field: {}", fieldName);
    }
    
    /**
     * 索引优化
     */
    public void optimizeIndex(String fieldName) {
        // 分析查询模式
        QueryPattern pattern = analyzeQueryPattern();
        
        // 根据查询模式调整索引参数
        if (pattern.isHighThroughput()) {
            // 高吞吐量场景，优化为快速检索
            optimizeForThroughput(fieldName);
        } else if (pattern.isHighAccuracy()) {
            // 高精度场景，优化为准确检索
            optimizeForAccuracy(fieldName);
        } else {
            // 均衡场景
            optimizeForBalance(fieldName);
        }
    }
    
    /**
     * 索引重建
     */
    public void rebuildIndex(String fieldName) {
        // 获取当前索引统计信息
        IndexStats stats = vectorStore.getIndexStats(fieldName);
        
        if (shouldRebuild(stats)) {
            logger.info("Rebuilding index for field: {}", fieldName);
            
            // 创建新索引
            String tempIndexName = fieldName + "_temp";
            vectorStore.createIndex(tempIndexName, stats.getIndexType(), stats.getParams());
            
            // 切换索引
            vectorStore.switchIndex(fieldName, tempIndexName);
            
            // 删除旧索引
            vectorStore.dropIndex(fieldName + "_old");
            
            logger.info("Index rebuild completed for field: {}", fieldName);
        }
    }
    
    private boolean shouldRebuild(IndexStats stats) {
        // 索引碎片率过高
        if (stats.getFragmentationRatio() > 0.3) {
            return true;
        }
        
        // 数据增长过多
        if (stats.getDataGrowthRatio() > 2.0) {
            return true;
        }
        
        // 查询性能下降
        if (stats.getAverageQueryTime() > stats.getBaselineQueryTime() * 1.5) {
            return true;
        }
        
        return false;
    }
}
```

### 7.2 索引监控

```java
public class IndexMonitor {
    
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private final MeterRegistry meterRegistry;
    
    public IndexMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void startMonitoring(VectorStore vectorStore) {
        scheduler.scheduleAtFixedRate(() -> {
            try {
                collectIndexMetrics(vectorStore);
            } catch (Exception e) {
                logger.error("Failed to collect index metrics", e);
            }
        }, 0, 60, TimeUnit.SECONDS);
    }
    
    private void collectIndexMetrics(VectorStore vectorStore) {
        VectorStoreStats stats = vectorStore.getStats();
        
        // 索引大小
        Gauge.builder("vector_store.index.size")
            .description("Index size in bytes")
            .register(meterRegistry, stats, VectorStoreStats::getIndexSize);
        
        // 文档数量
        Gauge.builder("vector_store.document.count")
            .description("Total document count")
            .register(meterRegistry, stats, VectorStoreStats::getDocumentCount);
        
        // 查询QPS
        Gauge.builder("vector_store.query.qps")
            .description("Queries per second")
            .register(meterRegistry, stats, VectorStoreStats::getQueryQPS);
        
        // 平均查询延迟
        Gauge.builder("vector_store.query.latency")
            .description("Average query latency")
            .register(meterRegistry, stats, VectorStoreStats::getAverageLatency);
        
        // 索引命中率
        Gauge.builder("vector_store.index.hit_rate")
            .description("Index hit rate")
            .register(meterRegistry, stats, VectorStoreStats::getIndexHitRate);
    }
}
```

## 8. 数据同步与一致性

### 8.1 数据同步器

```java
public class DataSynchronizer {
    
    private final VectorStore primaryStore;
    private final List<VectorStore> replicaStores;
    private final ExecutorService executorService;
    
    public DataSynchronizer(VectorStore primaryStore, List<VectorStore> replicaStores) {
        this.primaryStore = primaryStore;
        this.replicaStores = replicaStores;
        this.executorService = Executors.newFixedThreadPool(replicaStores.size());
    }
    
    /**
     * 同步添加文档
     */
    public CompletableFuture<Void> syncAdd(List<VectorDocument> documents) {
        // 先写主存储
        CompletableFuture<Void> primaryFuture = CompletableFuture.runAsync(() -> 
            primaryStore.add(documents));
        
        // 异步写副本存储
        List<CompletableFuture<Void>> replicaFutures = replicaStores.stream()
            .map(replica -> CompletableFuture.runAsync(() -> 
                replica.add(documents), executorService))
            .collect(Collectors.toList());
        
        // 等待主存储完成，副本存储异步执行
        return primaryFuture.thenCompose(v -> 
            CompletableFuture.allOf(replicaFutures.toArray(new CompletableFuture[0])));
    }
    
    /**
     * 数据一致性检查
     */
    public CompletableFuture<ConsistencyReport> checkConsistency() {
        return CompletableFuture.supplyAsync(() -> {
            VectorStoreStats primaryStats = primaryStore.getStats();
            ConsistencyReport report = new ConsistencyReport();
            
            for (VectorStore replica : replicaStores) {
                VectorStoreStats replicaStats = replica.getStats();
                
                ConsistencyCheck check = new ConsistencyCheck();
                check.setReplicaId(replica.getId());
                check.setDocumentCountDiff(
                    Math.abs(primaryStats.getDocumentCount() - replicaStats.getDocumentCount()));
                check.setConsistent(check.getDocumentCountDiff() == 0);
                
                report.addCheck(check);
            }
            
            return report;
        });
    }
    
    /**
     * 数据修复
     */
    public CompletableFuture<Void> repairInconsistency(String replicaId) {
        return CompletableFuture.runAsync(() -> {
            VectorStore replica = findReplica(replicaId);
            if (replica == null) {
                throw new IllegalArgumentException("Replica not found: " + replicaId);
            }
            
            // 获取主存储的所有文档
            List<VectorDocument> primaryDocuments = primaryStore.getAllDocuments();
            
            // 清空副本并重新加载
            replica.clear();
            replica.add(primaryDocuments);
            
            logger.info("Repaired inconsistency for replica: {}", replicaId);
        });
    }
}
```

## 9. 高级特性

### 9.1 分片支持

```java
public class ShardedVectorStore implements VectorStore {
    
    private final List<VectorStore> shards;
    private final ShardingStrategy shardingStrategy;
    private final ConsistentHashRouter router;
    
    public ShardedVectorStore(List<VectorStore> shards, ShardingStrategy strategy) {
        this.shards = shards;
        this.shardingStrategy = strategy;
        this.router = new ConsistentHashRouter(shards);
    }
    
    @Override
    public void add(List<VectorDocument> documents) {
        // 按分片策略分组文档
        Map<VectorStore, List<VectorDocument>> shardGroups = documents.stream()
            .collect(Collectors.groupingBy(doc -> 
                router.route(doc.getId())));
        
        // 并行写入各分片
        List<CompletableFuture<Void>> futures = shardGroups.entrySet().stream()
            .map(entry -> CompletableFuture.runAsync(() -> 
                entry.getKey().add(entry.getValue())))
            .collect(Collectors.toList());
        
        // 等待所有分片写入完成
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    }
    
    @Override
    public List<VectorDocument> similaritySearch(float[] query, int topK, VectorFilter filter) {
        // 并行搜索所有分片
        List<CompletableFuture<List<VectorDocument>>> futures = shards.stream()
            .map(shard -> CompletableFuture.supplyAsync(() -> 
                shard.similaritySearch(query, topK, filter)))
            .collect(Collectors.toList());
        
        // 合并所有分片的结果
        List<VectorDocument> allResults = futures.stream()
            .flatMap(future -> future.join().stream())
            .collect(Collectors.toList());
        
        // 重新排序并返回TopK
        return allResults.stream()
            .sorted((a, b) -> Double.compare(b.getScore(), a.getScore()))
            .limit(topK)
            .collect(Collectors.toList());
    }
}
```

### 9.2 缓存层

```java
public class CachedVectorStore implements VectorStore {
    
    private final VectorStore delegate;
    private final Cache<String, VectorDocument> documentCache;
    private final Cache<String, List<VectorDocument>> searchCache;
    
    public CachedVectorStore(VectorStore delegate, CacheConfig config) {
        this.delegate = delegate;
        
        this.documentCache = Caffeine.newBuilder()
            .maximumSize(config.getDocumentCacheSize())
            .expireAfterWrite(config.getDocumentCacheExpiration())
            .recordStats()
            .build();
            
        this.searchCache = Caffeine.newBuilder()
            .maximumSize(config.getSearchCacheSize())
            .expireAfterWrite(config.getSearchCacheExpiration())
            .recordStats()
            .build();
    }
    
    @Override
    public List<VectorDocument> similaritySearch(float[] query, int topK, VectorFilter filter) {
        String cacheKey = buildSearchCacheKey(query, topK, filter);
        
        return searchCache.get(cacheKey, key -> 
            delegate.similaritySearch(query, topK, filter));
    }
    
    @Override
    public void add(List<VectorDocument> documents) {
        delegate.add(documents);
        
        // 更新文档缓存
        documents.forEach(doc -> 
            documentCache.put(doc.getId(), doc));
        
        // 清空搜索缓存（数据已变更）
        searchCache.invalidateAll();
    }
    
    private String buildSearchCacheKey(float[] query, int topK, VectorFilter filter) {
        return DigestUtils.md5Hex(
            Arrays.toString(query) + topK + (filter != null ? filter.toString() : ""));
    }
}
```

## 10. 对Sen-AiFlow的启发

### 核心设计理念

1. **统一抽象**: 屏蔽不同向量数据库的实现差异
2. **性能优化**: 批量操作、连接池、缓存机制
3. **灵活搜索**: 多种相似性度量和搜索策略
4. **元数据丰富**: 强大的过滤和查询能力
5. **企业级特性**: 分片、同步、监控等高级功能

### Sen-AiFlow响应式向量存储设计

```java
// Sen-AiFlow中的响应式向量存储设计
@SenAiFlowComponent("reactive-vector-store")
public interface ReactiveVectorStore {
    
    // 响应式文档操作
    Mono<Void> add(Flux<VectorDocument> documents);
    Flux<VectorDocument> similaritySearch(Mono<float[]> query, int topK);
    Mono<Long> delete(Flux<String> ids);
    
    // 流式搜索
    Flux<VectorDocument> searchStream(Mono<float[]> query, SearchOptions options);
    
    // 批量操作
    Flux<BatchResult> addBatch(Flux<List<VectorDocument>> batches);
    
    // 健康检查
    Mono<StoreHealth> checkHealth();
    
    // 指标监控
    Flux<StoreMetrics> getMetrics();
}
```

Agents-Flex的Store模块展示了向量存储系统的完整解决方案，其统一抽象、性能优化和企业级特性为Sen-AiFlow的向量存储能力提供了重要参考，特别是在保持高性能的同时支持多种数据库后端方面的设计经验。 