# Agents-Flex - Document文档处理模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Agents-Flex的Document Parser模块提供了强大的文档处理能力，支持多种文档格式的解析和内容提取。该模块是构建RAG（检索增强生成）系统的重要基础，能够将各种格式的文档转换为结构化的Document对象，便于后续的向量化存储和检索。

### 核心特性

- **多格式支持**: 支持PDF、Word、Excel、PowerPoint等主流文档格式
- **分页处理**: 支持按页、按章节等方式拆分大文档
- **元数据提取**: 自动提取文档元数据（作者、创建时间、页码等）
- **内容清洗**: 智能清理格式符号，提取纯文本内容
- **扩展机制**: 插件化架构，易于扩展新的文档格式
- **并行处理**: 支持批量文档的并行处理
- **异常处理**: 完善的异常处理和错误恢复机制

## 2. 核心架构

### 文档处理架构

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Document Source   │───▶│   Parser Registry    │───▶│   Document Parser   │
│   (文档源)          │    │   (解析器注册表)     │    │   (文档解析器)      │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          ▼                            ▼                           ▼
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Content Cleaner   │    │   Metadata Extractor │    │   Document Splitter │
│   (内容清洗)        │    │   (元数据提取)       │    │   (文档分割)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          └────────────────────────────┼───────────────────────────┘
                                       ▼
                           ┌──────────────────────┐
                           │   Document Object    │
                           │   (文档对象)         │
                           └──────────────────────┘
```

### 解析器类型体系

```
DocumentParser (解析器接口)
├── PdfBoxDocumentParser (PDF解析器 - 基于Apache PDFBox)
├── PoiDocumentParser (Office文档解析器 - 基于Apache POI)
├── OmniParseDocumentParser (多格式解析器 - 基于OmniParse)
├── PlainTextParser (纯文本解析器)
├── MarkdownParser (Markdown解析器)  
└── CustomDocumentParser (自定义解析器)
```

## 3. 核心实现分析

### 3.1 Document核心对象

```java
public class Document {
    private String content;                          // 文档内容
    private Map<String, Object> metadata;           // 元数据
    private List<DocumentChunk> chunks;             // 文档分块
    private DocumentSource source;                  // 文档来源
    
    public Document(String content) {
        this.content = content;
        this.metadata = new HashMap<>();
        this.chunks = new ArrayList<>();
    }
    
    // 添加元数据
    public void addMetadata(String key, Object value) {
        metadata.put(key, value);
    }
    
    // 获取元数据
    public <T> T getMetadata(String key, Class<T> clazz) {
        Object value = metadata.get(key);
        return value != null ? clazz.cast(value) : null;
    }
    
    // 分割文档
    public List<Document> split(DocumentSplitter splitter) {
        return splitter.split(this);
    }
    
    // 清洗内容
    public Document clean(ContentCleaner cleaner) {
        String cleanedContent = cleaner.clean(this.content);
        Document cleanedDoc = new Document(cleanedContent);
        cleanedDoc.metadata.putAll(this.metadata);
        return cleanedDoc;
    }
    
    // 计算文档指纹
    public String getFingerprint() {
        return DigestUtils.md5Hex(content + metadata.toString());
    }
}
```

**设计特点**:
- **不可变性**: 内容修改通过返回新对象实现
- **元数据丰富**: 支持任意键值对元数据
- **链式操作**: 支持文档处理的链式调用
- **指纹机制**: 用于文档去重和缓存

### 3.2 DocumentParser接口设计

```java
public interface DocumentParser {
    
    /**
     * 解析文档
     * @param stream 文档输入流
     * @return 解析后的文档对象
     */
    Document parse(InputStream stream);
    
    /**
     * 批量解析（默认实现）
     * @param stream 文档输入流
     * @return 文档列表
     */
    default List<Document> parseMultiple(InputStream stream) {
        return Arrays.asList(parse(stream));
    }
    
    /**
     * 支持的文件类型
     * @return 文件扩展名列表
     */
    default List<String> getSupportedExtensions() {
        return Collections.emptyList();
    }
    
    /**
     * 支持的MIME类型
     * @return MIME类型列表
     */
    default List<String> getSupportedMimeTypes() {
        return Collections.emptyList();
    }
    
    /**
     * 解析器优先级
     * @return 优先级数值，越大优先级越高
     */
    default int getPriority() {
        return 0;
    }
}
```

### 3.3 PDF文档解析器

```java
public class PdfBoxDocumentParser implements DocumentParser {
    
    private boolean extractMetadata = true;
    private boolean preserveFormatting = false;
    
    @Override
    public Document parse(InputStream stream) {
        try (PDDocument pdfDocument = PDDocument.load(stream)) {
            // 1. 提取文本内容
            PDFTextStripper stripper = new PDFTextStripper();
            if (preserveFormatting) {
                stripper.setSortByPosition(true);
                stripper.setLineSeparator("\n");
            }
            
            String text = stripper.getText(pdfDocument);
            
            // 2. 创建文档对象
            Document document = new Document(text);
            
            // 3. 提取元数据
            if (extractMetadata) {
                extractPdfMetadata(pdfDocument, document);
            }
            
            return document;
            
        } catch (IOException e) {
            throw new DocumentParseException("Failed to parse PDF document", e);
        }
    }
    
    /**
     * 按页解析PDF
     */
    public List<Document> parseWithPage(InputStream inputStream) {
        try (PDDocument pdfDocument = PDDocument.load(inputStream)) {
            List<Document> documents = new ArrayList<>();
            int pageCount = pdfDocument.getNumberOfPages();
            
            for (int pageNumber = 1; pageNumber <= pageCount; pageNumber++) {
                // 设置页面范围
                PDFTextStripper stripper = new PDFTextStripper();
                stripper.setStartPage(pageNumber);
                stripper.setEndPage(pageNumber);
                
                String content = stripper.getText(pdfDocument);
                
                // 创建页面文档
                Document pageDocument = new Document(content);
                pageDocument.addMetadata("pageNumber", pageNumber);
                pageDocument.addMetadata("totalPages", pageCount);
                pageDocument.addMetadata("documentType", "pdf_page");
                
                // 提取页面级元数据
                extractPageMetadata(pdfDocument, pageNumber, pageDocument);
                
                documents.add(pageDocument);
            }
            
            return documents;
            
        } catch (IOException e) {
            throw new DocumentParseException("Failed to parse PDF with pages", e);
        }
    }
    
    private void extractPdfMetadata(PDDocument pdfDocument, Document document) {
        PDDocumentInformation info = pdfDocument.getDocumentInformation();
        
        if (info.getTitle() != null) {
            document.addMetadata("title", info.getTitle());
        }
        if (info.getAuthor() != null) {
            document.addMetadata("author", info.getAuthor());
        }
        if (info.getSubject() != null) {
            document.addMetadata("subject", info.getSubject());
        }
        if (info.getKeywords() != null) {
            document.addMetadata("keywords", info.getKeywords());
        }
        if (info.getCreationDate() != null) {
            document.addMetadata("creationDate", info.getCreationDate().getTime());
        }
        if (info.getModificationDate() != null) {
            document.addMetadata("modificationDate", info.getModificationDate().getTime());
        }
        
        // 文档统计信息
        document.addMetadata("pageCount", pdfDocument.getNumberOfPages());
        document.addMetadata("documentType", "pdf");
        document.addMetadata("fileSize", estimateFileSize(pdfDocument));
    }
    
    @Override
    public List<String> getSupportedExtensions() {
        return Arrays.asList("pdf");
    }
    
    @Override
    public List<String> getSupportedMimeTypes() {
        return Arrays.asList("application/pdf");
    }
}
```

### 3.4 Office文档解析器

```java
public class PoiDocumentParser implements DocumentParser {
    
    @Override
    public Document parse(InputStream stream) {
        try (POITextExtractor extractor = ExtractorFactory.createExtractor(stream)) {
            // 1. 提取文本内容
            String text = extractor.getText();
            
            // 2. 创建文档对象
            Document document = new Document(text);
            
            // 3. 根据文档类型提取特定元数据
            extractOfficeMetadata(extractor, document);
            
            return document;
            
        } catch (IOException | InvalidFormatException e) {
            throw new DocumentParseException("Failed to parse Office document", e);
        }
    }
    
    /**
     * 解析Word文档并保持结构
     */
    public List<Document> parseWordWithStructure(InputStream stream) {
        try (XWPFDocument document = new XWPFDocument(stream)) {
            List<Document> sections = new ArrayList<>();
            
            // 按段落解析
            List<XWPFParagraph> paragraphs = document.getParagraphs();
            for (int i = 0; i < paragraphs.size(); i++) {
                XWPFParagraph paragraph = paragraphs.get(i);
                String text = paragraph.getText();
                
                if (StringUtils.isNotBlank(text)) {
                    Document section = new Document(text);
                    section.addMetadata("paragraphIndex", i);
                    section.addMetadata("documentType", "word_paragraph");
                    
                    // 提取段落样式信息
                    if (paragraph.getStyle() != null) {
                        section.addMetadata("style", paragraph.getStyle());
                    }
                    
                    sections.add(section);
                }
            }
            
            // 处理表格
            List<XWPFTable> tables = document.getTables();
            for (int i = 0; i < tables.size(); i++) {
                XWPFTable table = tables.get(i);
                String tableText = extractTableText(table);
                
                Document tableDoc = new Document(tableText);
                tableDoc.addMetadata("tableIndex", i);
                tableDoc.addMetadata("documentType", "word_table");
                
                sections.add(tableDoc);
            }
            
            return sections;
            
        } catch (IOException e) {
            throw new DocumentParseException("Failed to parse Word document with structure", e);
        }
    }
    
    private void extractOfficeMetadata(POITextExtractor extractor, Document document) {
        try {
            // 获取文档属性
            POIDocument poiDocument = extractor.getDocument();
            SummaryInformation summaryInfo = poiDocument.getSummaryInformation();
            DocumentSummaryInformation docSummaryInfo = poiDocument.getDocumentSummaryInformation();
            
            if (summaryInfo != null) {
                if (summaryInfo.getTitle() != null) {
                    document.addMetadata("title", summaryInfo.getTitle());
                }
                if (summaryInfo.getAuthor() != null) {
                    document.addMetadata("author", summaryInfo.getAuthor());
                }
                if (summaryInfo.getSubject() != null) {
                    document.addMetadata("subject", summaryInfo.getSubject());
                }
                if (summaryInfo.getKeywords() != null) {
                    document.addMetadata("keywords", summaryInfo.getKeywords());
                }
                if (summaryInfo.getCreateDateTime() != null) {
                    document.addMetadata("creationDate", summaryInfo.getCreateDateTime().getTime());
                }
                if (summaryInfo.getLastSaveDateTime() != null) {
                    document.addMetadata("modificationDate", summaryInfo.getLastSaveDateTime().getTime());
                }
            }
            
            if (docSummaryInfo != null) {
                if (docSummaryInfo.getCompany() != null) {
                    document.addMetadata("company", docSummaryInfo.getCompany());
                }
                if (docSummaryInfo.getCategory() != null) {
                    document.addMetadata("category", docSummaryInfo.getCategory());
                }
            }
            
            // 确定文档类型
            String documentType = determineOfficeDocumentType(extractor);
            document.addMetadata("documentType", documentType);
            
        } catch (Exception e) {
            // 元数据提取失败不应该影响文档解析
            logger.warn("Failed to extract metadata from Office document", e);
        }
    }
    
    private String determineOfficeDocumentType(POITextExtractor extractor) {
        if (extractor instanceof XWPFWordExtractor) {
            return "word";
        } else if (extractor instanceof XSSFExcelExtractor) {
            return "excel";
        } else if (extractor instanceof XSLFPowerPointExtractor) {
            return "powerpoint";
        } else {
            return "office";
        }
    }
    
    @Override
    public List<String> getSupportedExtensions() {
        return Arrays.asList("doc", "docx", "xls", "xlsx", "ppt", "pptx");
    }
    
    @Override
    public List<String> getSupportedMimeTypes() {
        return Arrays.asList(
            "application/msword",
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            "application/vnd.ms-excel",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            "application/vnd.ms-powerpoint",
            "application/vnd.openxmlformats-officedocument.presentationml.presentation"
        );
    }
}
```

## 4. 文档分割与分块

### 4.1 DocumentSplitter接口

```java
public interface DocumentSplitter {
    
    /**
     * 分割文档
     * @param document 原始文档
     * @return 分割后的文档列表
     */
    List<Document> split(Document document);
    
    /**
     * 分割配置
     * @return 分割配置
     */
    default SplitConfig getConfig() {
        return SplitConfig.defaults();
    }
}

public class SplitConfig {
    private int maxChunkSize = 1000;        // 最大分块大小
    private int chunkOverlap = 200;         // 分块重叠大小
    private String splitPattern = "\n\n";   // 分割模式
    private boolean preserveMetadata = true; // 保持元数据
    
    public static SplitConfig defaults() {
        return new SplitConfig();
    }
    
    public static SplitConfig forRAG() {
        return new SplitConfig()
            .setMaxChunkSize(512)
            .setChunkOverlap(50)
            .setSplitPattern("。|！|？|\n\n");
    }
}
```

### 4.2 智能分块器

```java
public class SemanticDocumentSplitter implements DocumentSplitter {
    
    private final SplitConfig config;
    private final SentenceTokenizer sentenceTokenizer;
    
    public SemanticDocumentSplitter(SplitConfig config) {
        this.config = config;
        this.sentenceTokenizer = new SentenceTokenizer();
    }
    
    @Override
    public List<Document> split(Document document) {
        List<Document> chunks = new ArrayList<>();
        String content = document.getContent();
        
        // 1. 按句子分割
        List<String> sentences = sentenceTokenizer.tokenize(content);
        
        // 2. 构建语义块
        List<String> semanticChunks = buildSemanticChunks(sentences);
        
        // 3. 创建文档块
        for (int i = 0; i < semanticChunks.size(); i++) {
            String chunkContent = semanticChunks.get(i);
            
            Document chunk = new Document(chunkContent);
            
            // 复制元数据
            if (config.isPreserveMetadata()) {
                chunk.getMetadata().putAll(document.getMetadata());
            }
            
            // 添加分块元数据
            chunk.addMetadata("chunkIndex", i);
            chunk.addMetadata("totalChunks", semanticChunks.size());
            chunk.addMetadata("chunkSize", chunkContent.length());
            chunk.addMetadata("parentDocumentId", document.getFingerprint());
            
            chunks.add(chunk);
        }
        
        return chunks;
    }
    
    private List<String> buildSemanticChunks(List<String> sentences) {
        List<String> chunks = new ArrayList<>();
        StringBuilder currentChunk = new StringBuilder();
        
        for (String sentence : sentences) {
            // 检查是否超过最大块大小
            if (currentChunk.length() + sentence.length() > config.getMaxChunkSize()) {
                if (currentChunk.length() > 0) {
                    chunks.add(currentChunk.toString().trim());
                    
                    // 处理重叠
                    currentChunk = new StringBuilder();
                    if (config.getChunkOverlap() > 0) {
                        String overlap = getOverlapText(chunks.get(chunks.size() - 1), config.getChunkOverlap());
                        currentChunk.append(overlap);
                    }
                }
            }
            
            currentChunk.append(sentence).append(" ");
        }
        
        // 添加最后一个块
        if (currentChunk.length() > 0) {
            chunks.add(currentChunk.toString().trim());
        }
        
        return chunks;
    }
    
    private String getOverlapText(String text, int overlapSize) {
        if (text.length() <= overlapSize) {
            return text;
        }
        
        // 从句子边界开始重叠
        String suffix = text.substring(text.length() - overlapSize);
        int lastSentenceEnd = Math.max(
            suffix.lastIndexOf("。"),
            Math.max(suffix.lastIndexOf("！"), suffix.lastIndexOf("？"))
        );
        
        if (lastSentenceEnd > 0) {
            return suffix.substring(lastSentenceEnd + 1);
        }
        
        return suffix;
    }
}
```

### 4.3 结构化分割器

```java
public class StructuredDocumentSplitter implements DocumentSplitter {
    
    @Override
    public List<Document> split(Document document) {
        String content = document.getContent();
        List<Document> sections = new ArrayList<>();
        
        // 1. 识别文档结构
        DocumentStructure structure = analyzeStructure(content);
        
        // 2. 按结构分割
        for (Section section : structure.getSections()) {
            Document sectionDoc = new Document(section.getContent());
            
            // 复制原文档元数据
            sectionDoc.getMetadata().putAll(document.getMetadata());
            
            // 添加结构元数据
            sectionDoc.addMetadata("sectionType", section.getType());
            sectionDoc.addMetadata("sectionLevel", section.getLevel());
            sectionDoc.addMetadata("sectionTitle", section.getTitle());
            sectionDoc.addMetadata("sectionIndex", section.getIndex());
            
            sections.add(sectionDoc);
        }
        
        return sections;
    }
    
    private DocumentStructure analyzeStructure(String content) {
        DocumentStructure structure = new DocumentStructure();
        String[] lines = content.split("\n");
        
        Section currentSection = null;
        StringBuilder sectionContent = new StringBuilder();
        int sectionIndex = 0;
        
        for (String line : lines) {
            // 检测标题模式
            SectionType sectionType = detectSectionType(line);
            
            if (sectionType != SectionType.CONTENT) {
                // 保存前一个段落
                if (currentSection != null) {
                    currentSection.setContent(sectionContent.toString().trim());
                    structure.addSection(currentSection);
                }
                
                // 开始新段落
                currentSection = new Section();
                currentSection.setType(sectionType);
                currentSection.setTitle(extractTitle(line));
                currentSection.setLevel(detectLevel(line));
                currentSection.setIndex(sectionIndex++);
                
                sectionContent = new StringBuilder();
            } else {
                // 添加到当前段落
                sectionContent.append(line).append("\n");
            }
        }
        
        // 添加最后一个段落
        if (currentSection != null) {
            currentSection.setContent(sectionContent.toString().trim());
            structure.addSection(currentSection);
        }
        
        return structure;
    }
    
    private SectionType detectSectionType(String line) {
        line = line.trim();
        
        // 检测Markdown标题
        if (line.startsWith("#")) {
            return SectionType.HEADING;
        }
        
        // 检测编号标题
        if (line.matches("^\\d+\\.\\s+.+")) {
            return SectionType.NUMBERED_HEADING;
        }
        
        // 检测列表项
        if (line.startsWith("- ") || line.startsWith("* ") || line.matches("^\\d+\\.\\s+.+")) {
            return SectionType.LIST_ITEM;
        }
        
        return SectionType.CONTENT;
    }
}
```

## 5. 内容清洗与优化

### 5.1 ContentCleaner接口

```java
public interface ContentCleaner {
    
    /**
     * 清洗文档内容
     * @param content 原始内容
     * @return 清洗后的内容
     */
    String clean(String content);
    
    /**
     * 清洗配置
     * @return 清洗配置
     */
    default CleanConfig getConfig() {
        return CleanConfig.defaults();
    }
}

public class CleanConfig {
    private boolean removeExtraWhitespace = true;     // 移除多余空白
    private boolean removeSpecialChars = false;       // 移除特殊字符
    private boolean normalizeUnicode = true;          // Unicode标准化
    private boolean removeEmptyLines = true;          // 移除空行
    private List<String> preservePatterns = new ArrayList<>(); // 保留模式
    private List<String> removePatterns = new ArrayList<>();   // 移除模式
}
```

### 5.2 智能内容清洗器

```java
public class IntelligentContentCleaner implements ContentCleaner {
    
    private final CleanConfig config;
    private final Pattern emailPattern = Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b");
    private final Pattern urlPattern = Pattern.compile("https?://[\\w\\-._~:/?#[\\]@!$&'()*+,;=%]+");
    private final Pattern phonePattern = Pattern.compile("\\b\\d{3}-\\d{3}-\\d{4}\\b|\\b\\d{10}\\b");
    
    public IntelligentContentCleaner(CleanConfig config) {
        this.config = config;
    }
    
    @Override
    public String clean(String content) {
        if (content == null || content.trim().isEmpty()) {
            return "";
        }
        
        String cleaned = content;
        
        // 1. Unicode标准化
        if (config.isNormalizeUnicode()) {
            cleaned = Normalizer.normalize(cleaned, Normalizer.Form.NFC);
        }
        
        // 2. 移除不可见字符
        cleaned = removeInvisibleChars(cleaned);
        
        // 3. 处理换行符
        cleaned = normalizeLineBreaks(cleaned);
        
        // 4. 移除多余空白
        if (config.isRemoveExtraWhitespace()) {
            cleaned = removeExtraWhitespace(cleaned);
        }
        
        // 5. 移除空行
        if (config.isRemoveEmptyLines()) {
            cleaned = removeEmptyLines(cleaned);
        }
        
        // 6. 应用自定义清洗规则
        cleaned = applyCustomRules(cleaned);
        
        // 7. 保护敏感信息
        cleaned = protectSensitiveInfo(cleaned);
        
        return cleaned.trim();
    }
    
    private String removeInvisibleChars(String text) {
        // 移除零宽字符和控制字符
        return text.replaceAll("[\\p{C}&&[^\n\r\t]]", "");
    }
    
    private String normalizeLineBreaks(String text) {
        // 统一换行符
        return text.replaceAll("\\r\\n|\\r", "\n");
    }
    
    private String removeExtraWhitespace(String text) {
        // 移除多余的空格和制表符
        text = text.replaceAll("[ \t]+", " ");
        // 移除行首行尾空白
        text = text.replaceAll("(?m)^[ \t]+|[ \t]+$", "");
        return text;
    }
    
    private String removeEmptyLines(String text) {
        // 移除空行，但保留单个换行符
        return text.replaceAll("(?m)^[ \t]*\n", "\n")
                  .replaceAll("\n{3,}", "\n\n");
    }
    
    private String applyCustomRules(String text) {
        // 应用移除模式
        for (String pattern : config.getRemovePatterns()) {
            text = text.replaceAll(pattern, "");
        }
        
        return text;
    }
    
    private String protectSensitiveInfo(String text) {
        // 脱敏邮箱地址
        text = emailPattern.matcher(text).replaceAll("***@***.***");
        
        // 脱敏电话号码
        text = phonePattern.matcher(text).replaceAll("***-***-****");
        
        // 保留URL的域名部分
        text = urlPattern.matcher(text).replaceAll(match -> {
            try {
                URL url = new URL(match.group());
                return "https://" + url.getHost() + "/***";
            } catch (Exception e) {
                return "https://***";
            }
        });
        
        return text;
    }
}
```

## 6. 文档解析器注册与管理

### 6.1 解析器注册表

```java
public class ParserRegistry {
    
    private final Map<String, DocumentParser> extensionParsers = new ConcurrentHashMap<>();
    private final Map<String, DocumentParser> mimeTypeParsers = new ConcurrentHashMap<>();
    private final List<DocumentParser> orderedParsers = new ArrayList<>();
    
    public void registerParser(DocumentParser parser) {
        // 按扩展名注册
        for (String extension : parser.getSupportedExtensions()) {
            extensionParsers.put(extension.toLowerCase(), parser);
        }
        
        // 按MIME类型注册
        for (String mimeType : parser.getSupportedMimeTypes()) {
            mimeTypeParsers.put(mimeType.toLowerCase(), parser);
        }
        
        // 按优先级排序
        orderedParsers.add(parser);
        orderedParsers.sort((p1, p2) -> Integer.compare(p2.getPriority(), p1.getPriority()));
        
        logger.info("Registered parser: {} for extensions: {} and mime types: {}",
                   parser.getClass().getSimpleName(),
                   parser.getSupportedExtensions(),
                   parser.getSupportedMimeTypes());
    }
    
    public DocumentParser findParser(String filename) {
        String extension = getFileExtension(filename);
        return extensionParsers.get(extension.toLowerCase());
    }
    
    public DocumentParser findParserByMimeType(String mimeType) {
        return mimeTypeParsers.get(mimeType.toLowerCase());
    }
    
    public DocumentParser findBestParser(String filename, String mimeType) {
        // 优先使用MIME类型匹配
        if (mimeType != null) {
            DocumentParser parser = findParserByMimeType(mimeType);
            if (parser != null) {
                return parser;
            }
        }
        
        // 其次使用文件扩展名匹配
        if (filename != null) {
            DocumentParser parser = findParser(filename);
            if (parser != null) {
                return parser;
            }
        }
        
        // 最后尝试所有解析器
        for (DocumentParser parser : orderedParsers) {
            // 这里可以添加更复杂的匹配逻辑
            if (canHandle(parser, filename, mimeType)) {
                return parser;
            }
        }
        
        return null;
    }
    
    private boolean canHandle(DocumentParser parser, String filename, String mimeType) {
        // 可以在这里添加更智能的判断逻辑
        // 比如读取文件头、分析文件内容等
        return false;
    }
}
```

### 6.2 文档解析服务

```java
@Service
public class DocumentParsingService {
    
    private final ParserRegistry parserRegistry;
    private final ExecutorService executorService;
    private final ContentCleaner contentCleaner;
    private final DocumentSplitter documentSplitter;
    
    public DocumentParsingService() {
        this.parserRegistry = new ParserRegistry();
        this.executorService = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors()
        );
        this.contentCleaner = new IntelligentContentCleaner(CleanConfig.defaults());
        this.documentSplitter = new SemanticDocumentSplitter(SplitConfig.forRAG());
        
        // 注册默认解析器
        registerDefaultParsers();
    }
    
    /**
     * 解析单个文档
     */
    public Document parseDocument(InputStream inputStream, String filename, String mimeType) {
        DocumentParser parser = parserRegistry.findBestParser(filename, mimeType);
        if (parser == null) {
            throw new UnsupportedDocumentException("No parser found for file: " + filename);
        }
        
        try {
            // 1. 解析文档
            Document document = parser.parse(inputStream);
            
            // 2. 添加解析元数据
            document.addMetadata("originalFilename", filename);
            document.addMetadata("mimeType", mimeType);
            document.addMetadata("parsedBy", parser.getClass().getSimpleName());
            document.addMetadata("parseTime", System.currentTimeMillis());
            
            // 3. 清洗内容
            Document cleanedDocument = document.clean(contentCleaner);
            
            return cleanedDocument;
            
        } catch (Exception e) {
            throw new DocumentParseException("Failed to parse document: " + filename, e);
        }
    }
    
    /**
     * 批量解析文档
     */
    public CompletableFuture<List<Document>> parseDocuments(List<DocumentSource> sources) {
        List<CompletableFuture<Document>> futures = sources.stream()
            .map(source -> CompletableFuture.supplyAsync(() -> {
                try (InputStream stream = source.getInputStream()) {
                    return parseDocument(stream, source.getFilename(), source.getMimeType());
                } catch (Exception e) {
                    logger.error("Failed to parse document: {}", source.getFilename(), e);
                    return null;
                }
            }, executorService))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .filter(Objects::nonNull)
                .collect(Collectors.toList()));
    }
    
    /**
     * 解析并分割文档
     */
    public List<Document> parseAndSplit(InputStream inputStream, String filename, String mimeType) {
        Document document = parseDocument(inputStream, filename, mimeType);
        return documentSplitter.split(document);
    }
    
    private void registerDefaultParsers() {
        parserRegistry.registerParser(new PdfBoxDocumentParser());
        parserRegistry.registerParser(new PoiDocumentParser());
        parserRegistry.registerParser(new PlainTextParser());
        parserRegistry.registerParser(new MarkdownParser());
    }
}
```

## 7. 性能优化与缓存

### 7.1 文档缓存机制

```java
public class DocumentCache {
    
    private final Cache<String, Document> documentCache;
    private final Cache<String, List<Document>> chunkCache;
    
    public DocumentCache() {
        this.documentCache = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofHours(2))
            .recordStats()
            .build();
            
        this.chunkCache = Caffeine.newBuilder()
            .maximumSize(5000)
            .expireAfterWrite(Duration.ofHours(1))
            .recordStats()
            .build();
    }
    
    public Document getDocument(String key, Supplier<Document> supplier) {
        return documentCache.get(key, k -> supplier.get());
    }
    
    public List<Document> getChunks(String key, Supplier<List<Document>> supplier) {
        return chunkCache.get(key, k -> supplier.get());
    }
    
    public void invalidate(String key) {
        documentCache.invalidate(key);
        chunkCache.invalidate(key);
    }
    
    public CacheStats getDocumentCacheStats() {
        return documentCache.stats();
    }
    
    public CacheStats getChunkCacheStats() {
        return chunkCache.stats();
    }
}
```

### 7.2 异步处理优化

```java
public class AsyncDocumentProcessor {
    
    private final ExecutorService processingExecutor;
    private final CompletionService<ProcessingResult> completionService;
    
    public AsyncDocumentProcessor() {
        this.processingExecutor = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors()
        );
        this.completionService = new ExecutorCompletionService<>(processingExecutor);
    }
    
    public CompletableFuture<List<Document>> processDocumentsAsync(List<DocumentSource> sources) {
        return CompletableFuture.supplyAsync(() -> {
            List<Future<ProcessingResult>> futures = new ArrayList<>();
            
            // 提交所有任务
            for (DocumentSource source : sources) {
                futures.add(completionService.submit(() -> processDocument(source)));
            }
            
            // 收集结果
            List<Document> results = new ArrayList<>();
            for (int i = 0; i < futures.size(); i++) {
                try {
                    ProcessingResult result = completionService.take().get();
                    if (result.isSuccess()) {
                        results.addAll(result.getDocuments());
                    } else {
                        logger.warn("Document processing failed: {}", result.getError());
                    }
                } catch (Exception e) {
                    logger.error("Error processing document", e);
                }
            }
            
            return results;
        });
    }
    
    private ProcessingResult processDocument(DocumentSource source) {
        try {
            // 处理逻辑
            Document document = parseDocument(source);
            List<Document> chunks = splitDocument(document);
            
            return ProcessingResult.success(chunks);
        } catch (Exception e) {
            return ProcessingResult.failure(e);
        }
    }
}
```

## 8. 扩展与自定义

### 8.1 自定义解析器

```java
public class CustomMarkdownParser implements DocumentParser {
    
    private final MarkdownProcessor processor = new MarkdownProcessor();
    
    @Override
    public Document parse(InputStream stream) {
        try {
            String markdown = IOUtils.toString(stream, StandardCharsets.UTF_8);
            
            // 解析Markdown结构
            MarkdownDocument mdDoc = processor.parse(markdown);
            
            // 提取纯文本
            String plainText = mdDoc.toPlainText();
            
            Document document = new Document(plainText);
            
            // 提取Markdown特有的元数据
            extractMarkdownMetadata(mdDoc, document);
            
            return document;
            
        } catch (IOException e) {
            throw new DocumentParseException("Failed to parse Markdown document", e);
        }
    }
    
    private void extractMarkdownMetadata(MarkdownDocument mdDoc, Document document) {
        // 提取前言（Front Matter）
        if (mdDoc.hasFrontMatter()) {
            Map<String, Object> frontMatter = mdDoc.getFrontMatter();
            frontMatter.forEach(document::addMetadata);
        }
        
        // 提取标题结构
        List<String> headings = mdDoc.getHeadings();
        document.addMetadata("headings", headings);
        
        // 提取链接
        List<String> links = mdDoc.getLinks();
        document.addMetadata("links", links);
        
        // 提取图片
        List<String> images = mdDoc.getImages();
        document.addMetadata("images", images);
        
        document.addMetadata("documentType", "markdown");
    }
    
    @Override
    public List<String> getSupportedExtensions() {
        return Arrays.asList("md", "markdown");
    }
    
    @Override
    public List<String> getSupportedMimeTypes() {
        return Arrays.asList("text/markdown", "text/x-markdown");
    }
}
```

### 8.2 插件化扩展

```java
// SPI接口定义
public interface DocumentParserPlugin {
    String getName();
    String getVersion();
    List<DocumentParser> getParsers();
    void initialize(PluginContext context);
    void destroy();
}

// 插件管理器
public class PluginManager {
    
    private final Map<String, DocumentParserPlugin> plugins = new HashMap<>();
    private final ServiceLoader<DocumentParserPlugin> loader = 
        ServiceLoader.load(DocumentParserPlugin.class);
    
    public void loadPlugins() {
        for (DocumentParserPlugin plugin : loader) {
            try {
                plugin.initialize(new PluginContext());
                plugins.put(plugin.getName(), plugin);
                
                // 注册解析器
                for (DocumentParser parser : plugin.getParsers()) {
                    parserRegistry.registerParser(parser);
                }
                
                logger.info("Loaded plugin: {} v{}", plugin.getName(), plugin.getVersion());
            } catch (Exception e) {
                logger.error("Failed to load plugin: {}", plugin.getName(), e);
            }
        }
    }
}
```

## 9. 监控与诊断

### 9.1 解析指标

```java
@Component
public class DocumentParsingMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter parseCounter;
    private final Timer parseTimer;
    private final Gauge cacheSizeGauge;
    
    public DocumentParsingMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.parseCounter = Counter.builder("document.parse.total")
            .description("Total document parse attempts")
            .register(meterRegistry);
            
        this.parseTimer = Timer.builder("document.parse.duration")
            .description("Document parsing duration")
            .register(meterRegistry);
            
        this.cacheSizeGauge = Gauge.builder("document.cache.size")
            .description("Document cache size")
            .register(meterRegistry, this, DocumentParsingMetrics::getCacheSize);
    }
    
    public void recordParse(String documentType, String parserType, Duration duration, boolean success) {
        parseCounter.increment(
            Tags.of(
                "document_type", documentType,
                "parser_type", parserType,
                "status", success ? "success" : "failure"
            )
        );
        
        parseTimer.record(duration, 
            Tags.of(
                "document_type", documentType,
                "parser_type", parserType
            )
        );
    }
    
    private double getCacheSize() {
        return documentCache.estimatedSize();
    }
}
```

## 10. 对Sen-AiFlow的启发

### 核心设计理念

1. **插件化架构**: 支持多种文档格式的可扩展解析
2. **智能分块**: 基于语义的文档分割策略
3. **元数据丰富**: 保留文档结构和属性信息
4. **异步处理**: 高效的批量文档处理能力
5. **缓存优化**: 智能的解析结果缓存机制

### Sen-AiFlow文档处理设计

```java
// Sen-AiFlow中的响应式文档处理设计
@SenAiFlowComponent("reactive-document-processor")
public class ReactiveDocumentProcessor {
    
    public Flux<Document> parseDocument(Mono<DocumentSource> source) {
        return source
            .flatMapMany(this::detectParser)
            .flatMap(this::parseWithParser)
            .flatMap(this::cleanContent)
            .flatMapMany(this::splitDocument)
            .doOnNext(this::enrichMetadata);
    }
    
    public Flux<Document> parseBatch(Flux<DocumentSource> sources) {
        return sources
            .parallel()
            .flatMap(this::parseDocument)
            .sequential()
            .onErrorContinue((error, item) -> 
                logger.warn("Failed to parse document: {}", item, error));
    }
    
    private Mono<DocumentParser> detectParser(DocumentSource source) {
        return Mono.fromCallable(() -> 
            parserRegistry.findBestParser(source.getFilename(), source.getMimeType()));
    }
}
```

Agents-Flex的Document处理模块展示了企业级文档处理系统的完整解决方案，其插件化架构、智能分块和异步处理等特性为Sen-AiFlow的文档处理能力提供了重要参考，特别是在保持高性能的同时支持多种文档格式方面的设计经验。 