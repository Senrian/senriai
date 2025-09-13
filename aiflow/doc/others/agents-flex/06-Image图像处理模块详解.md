# Agents-Flex - Image图像处理模块详解

**作者**: senrian  
**日期**: 2024年12月

## 1. 模块概述

Agents-Flex的Image模块提供了全面的AI图像处理能力，支持多种主流的图像生成和处理服务。该模块通过统一的抽象接口，集成了OpenAI DALL·E、阿里云通义万相、腾讯云、Stability AI等多个图像AI服务，为开发者提供了一站式的图像AI解决方案。

### 核心特性

- **多服务集成**: 支持OpenAI、阿里云、腾讯云、Stability AI等主流图像AI服务
- **统一抽象**: 提供一致的API接口，屏蔽不同服务商的差异
- **多种操作**: 支持文本生成图像、图像编辑、图像变换、风格转换等
- **高质量输出**: 支持多种分辨率和图像质量选项
- **批量处理**: 高效的批量图像生成和处理
- **异步处理**: 支持异步操作和进度跟踪
- **缓存机制**: 智能的结果缓存和重用

## 2. 核心架构

### 图像处理架构

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Image Request     │───▶│   Image Provider     │───▶│   Image Response    │
│   (图像请求)        │    │   (图像提供商)       │    │   (图像响应)        │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          ▼                            ▼                           ▼
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│   Request Builder   │    │   Service Adapter    │    │   Result Processor  │
│   (请求构建器)      │    │   (服务适配器)       │    │   (结果处理器)      │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
          │                            │                           │
          └────────────────────────────┼───────────────────────────┘
                                       ▼
                           ┌──────────────────────┐
                           │   Cache Manager      │
                           │   (缓存管理器)       │
                           └──────────────────────┘
```

### 提供商支持矩阵

```
ImageProvider (统一接口)
├── OpenAI
│   ├── OpenAIImageProvider (DALL·E 2/3)
│   └── OpenAIImageEditProvider (图像编辑)
├── Alibaba
│   ├── QwenImageProvider (通义万相)
│   └── GiteeImageProvider (Gitee AI)
├── Tencent
│   └── TencentImageProvider (腾讯云图像生成)
├── Stability
│   └── StabilityImageProvider (Stable Diffusion)
├── Volcengine
│   └── VolcengineImageProvider (火山引擎)
└── SiliconFlow
    └── SiliconFlowImageProvider (硅基流动)
```

## 3. 核心实现分析

### 3.1 ImageProvider统一接口

```java
public interface ImageProvider {
    
    /**
     * 根据文本提示生成图像
     * @param prompt 文本提示
     * @param options 生成选项
     * @return 生成的图像结果
     */
    ImageResponse generate(String prompt, ImageOptions options);
    
    /**
     * 编辑已有图像
     * @param originalImage 原始图像
     * @param prompt 编辑提示
     * @param mask 编辑遮罩（可选）
     * @param options 编辑选项
     * @return 编辑后的图像
     */
    ImageResponse edit(BufferedImage originalImage, String prompt, 
                      BufferedImage mask, ImageOptions options);
    
    /**
     * 创建图像变体
     * @param originalImage 原始图像
     * @param options 变体选项
     * @return 图像变体
     */
    ImageResponse createVariation(BufferedImage originalImage, ImageOptions options);
    
    /**
     * 批量生成图像
     * @param prompts 文本提示列表
     * @param options 生成选项
     * @return 批量生成结果
     */
    List<ImageResponse> batchGenerate(List<String> prompts, ImageOptions options);
    
    /**
     * 异步生成图像
     * @param prompt 文本提示
     * @param options 生成选项
     * @param callback 结果回调
     */
    void generateAsync(String prompt, ImageOptions options, ImageCallback callback);
    
    /**
     * 获取提供商信息
     * @return 提供商信息
     */
    ProviderInfo getProviderInfo();
}
```

### 3.2 ImageOptions配置类

```java
public class ImageOptions {
    
    private String model = "dall-e-2";              // 模型名称
    private ImageSize size = ImageSize.SIZE_1024;   // 图像尺寸
    private ImageQuality quality = ImageQuality.STANDARD; // 图像质量
    private ImageStyle style = ImageStyle.VIVID;    // 图像风格
    private int n = 1;                              // 生成图像数量
    private String responseFormat = "url";          // 响应格式
    private String user;                            // 用户标识
    private Integer seed;                           // 随机种子
    private Double guidanceScale = 7.5;             // 引导比例
    private Integer steps = 30;                     // 推理步数
    private String sampler = "ddim";                // 采样器
    private Map<String, Object> extras = new HashMap<>(); // 扩展参数
    
    // Builder模式
    public static ImageOptions defaults() {
        return new ImageOptions();
    }
    
    public ImageOptions model(String model) {
        this.model = model;
        return this;
    }
    
    public ImageOptions size(ImageSize size) {
        this.size = size;
        return this;
    }
    
    public ImageOptions quality(ImageQuality quality) {
        this.quality = quality;
        return this;
    }
    
    public ImageOptions style(ImageStyle style) {
        this.style = style;
        return this;
    }
    
    public ImageOptions count(int n) {
        this.n = n;
        return this;
    }
    
    public ImageOptions seed(Integer seed) {
        this.seed = seed;
        return this;
    }
    
    public ImageOptions guidanceScale(Double guidanceScale) {
        this.guidanceScale = guidanceScale;
        return this;
    }
    
    public ImageOptions steps(Integer steps) {
        this.steps = steps;
        return this;
    }
    
    // 枚举定义
    public enum ImageSize {
        SIZE_256("256x256"),
        SIZE_512("512x512"), 
        SIZE_1024("1024x1024"),
        SIZE_1792_1024("1792x1024"),
        SIZE_1024_1792("1024x1792");
        
        private final String value;
        
        ImageSize(String value) {
            this.value = value;
        }
        
        public String getValue() {
            return value;
        }
    }
    
    public enum ImageQuality {
        STANDARD("standard"),
        HD("hd");
        
        private final String value;
        
        ImageQuality(String value) {
            this.value = value;
        }
    }
    
    public enum ImageStyle {
        VIVID("vivid"),
        NATURAL("natural");
        
        private final String value;
        
        ImageStyle(String value) {
            this.value = value;
        }
    }
}
```

### 3.3 OpenAI图像提供商实现

```java
public class OpenAIImageProvider implements ImageProvider {
    
    private final OpenAIClient client;
    private final String apiKey;
    private final String baseUrl;
    private final ObjectMapper objectMapper;
    
    public OpenAIImageProvider(OpenAIConfig config) {
        this.apiKey = config.getApiKey();
        this.baseUrl = config.getBaseUrl();
        this.objectMapper = new ObjectMapper();
        
        this.client = OpenAIClient.builder()
            .apiKey(apiKey)
            .baseUrl(baseUrl)
            .timeout(Duration.ofSeconds(config.getTimeout()))
            .build();
    }
    
    @Override
    public ImageResponse generate(String prompt, ImageOptions options) {
        try {
            // 构建请求
            CreateImageRequest request = CreateImageRequest.builder()
                .prompt(prompt)
                .model(options.getModel())
                .n(options.getN())
                .size(options.getSize().getValue())
                .quality(options.getQuality().getValue())
                .style(options.getStyle().getValue())
                .responseFormat(options.getResponseFormat())
                .user(options.getUser())
                .build();
            
            // 调用API
            CreateImageResponse response = client.createImage(request);
            
            // 处理响应
            return processImageResponse(response, prompt, options);
            
        } catch (Exception e) {
            logger.error("Failed to generate image with OpenAI", e);
            throw new ImageGenerationException("OpenAI image generation failed", e);
        }
    }
    
    @Override
    public ImageResponse edit(BufferedImage originalImage, String prompt, 
                             BufferedImage mask, ImageOptions options) {
        try {
            // 将图像转换为临时文件
            File imageFile = convertToTempFile(originalImage, "png");
            File maskFile = mask != null ? convertToTempFile(mask, "png") : null;
            
            // 构建编辑请求
            CreateImageEditRequest.Builder requestBuilder = CreateImageEditRequest.builder()
                .image(imageFile)
                .prompt(prompt)
                .n(options.getN())
                .size(options.getSize().getValue())
                .responseFormat(options.getResponseFormat())
                .user(options.getUser());
            
            if (maskFile != null) {
                requestBuilder.mask(maskFile);
            }
            
            CreateImageEditRequest request = requestBuilder.build();
            
            // 调用API
            CreateImageResponse response = client.createImageEdit(request);
            
            // 清理临时文件
            cleanupTempFiles(imageFile, maskFile);
            
            // 处理响应
            return processImageResponse(response, prompt, options);
            
        } catch (Exception e) {
            logger.error("Failed to edit image with OpenAI", e);
            throw new ImageEditException("OpenAI image edit failed", e);
        }
    }
    
    @Override
    public ImageResponse createVariation(BufferedImage originalImage, ImageOptions options) {
        try {
            // 将图像转换为临时文件
            File imageFile = convertToTempFile(originalImage, "png");
            
            // 构建变体请求
            CreateImageVariationRequest request = CreateImageVariationRequest.builder()
                .image(imageFile)
                .n(options.getN())
                .size(options.getSize().getValue())
                .responseFormat(options.getResponseFormat())
                .user(options.getUser())
                .build();
            
            // 调用API
            CreateImageResponse response = client.createImageVariation(request);
            
            // 清理临时文件
            imageFile.delete();
            
            // 处理响应
            return processImageResponse(response, "variation", options);
            
        } catch (Exception e) {
            logger.error("Failed to create image variation with OpenAI", e);
            throw new ImageVariationException("OpenAI image variation failed", e);
        }
    }
    
    @Override
    public void generateAsync(String prompt, ImageOptions options, ImageCallback callback) {
        CompletableFuture.supplyAsync(() -> {
            try {
                return generate(prompt, options);
            } catch (Exception e) {
                callback.onError(e);
                return null;
            }
        }).thenAccept(result -> {
            if (result != null) {
                callback.onSuccess(result);
            }
        });
    }
    
    private ImageResponse processImageResponse(CreateImageResponse response, 
                                             String prompt, ImageOptions options) {
        List<GeneratedImage> images = new ArrayList<>();
        
        for (ImageData imageData : response.getData()) {
            GeneratedImage image = GeneratedImage.builder()
                .url(imageData.getUrl())
                .b64Json(imageData.getB64Json())
                .revisedPrompt(imageData.getRevisedPrompt())
                .build();
            
            images.add(image);
        }
        
        return ImageResponse.builder()
            .images(images)
            .prompt(prompt)
            .options(options)
            .provider("openai")
            .created(response.getCreated())
            .build();
    }
    
    private File convertToTempFile(BufferedImage image, String format) throws IOException {
        File tempFile = File.createTempFile("image_", "." + format);
        ImageIO.write(image, format, tempFile);
        return tempFile;
    }
    
    private void cleanupTempFiles(File... files) {
        for (File file : files) {
            if (file != null && file.exists()) {
                file.delete();
            }
        }
    }
    
    @Override
    public ProviderInfo getProviderInfo() {
        return ProviderInfo.builder()
            .name("OpenAI")
            .version("v1")
            .supportedModels(Arrays.asList("dall-e-2", "dall-e-3"))
            .supportedOperations(Arrays.asList("generate", "edit", "variation"))
            .maxImageSize(ImageSize.SIZE_1792_1024)
            .supportsBatch(true)
            .supportsAsync(true)
            .build();
    }
}
```

### 3.4 阿里云通义万相提供商

```java
public class QwenImageProvider implements ImageProvider {
    
    private final DashScopeClient client;
    private final String apiKey;
    private final String model;
    
    public QwenImageProvider(QwenImageConfig config) {
        this.apiKey = config.getApiKey();
        this.model = config.getModel();
        
        this.client = DashScopeClient.builder()
            .apiKey(apiKey)
            .build();
    }
    
    @Override
    public ImageResponse generate(String prompt, ImageOptions options) {
        try {
            // 构建请求参数
            Map<String, Object> parameters = new HashMap<>();
            parameters.put("style", convertStyle(options.getStyle()));
            parameters.put("size", convertSize(options.getSize()));
            parameters.put("n", options.getN());
            
            if (options.getSeed() != null) {
                parameters.put("seed", options.getSeed());
            }
            
            // 构建请求
            GenerationRequest request = GenerationRequest.builder()
                .model(model)
                .input(GenerationInput.builder()
                    .prompt(prompt)
                    .build())
                .parameters(GenerationParameters.builder()
                    .putAll(parameters)
                    .build())
                .build();
            
            // 调用API
            GenerationResponse response = client.call(request);
            
            // 处理响应
            return processQwenResponse(response, prompt, options);
            
        } catch (Exception e) {
            logger.error("Failed to generate image with Qwen", e);
            throw new ImageGenerationException("Qwen image generation failed", e);
        }
    }
    
    @Override
    public List<ImageResponse> batchGenerate(List<String> prompts, ImageOptions options) {
        return prompts.parallelStream()
            .map(prompt -> generate(prompt, options))
            .collect(Collectors.toList());
    }
    
    private ImageResponse processQwenResponse(GenerationResponse response, 
                                            String prompt, ImageOptions options) {
        List<GeneratedImage> images = new ArrayList<>();
        
        if (response.getOutput() != null && response.getOutput().getResults() != null) {
            for (GenerationResult result : response.getOutput().getResults()) {
                GeneratedImage image = GeneratedImage.builder()
                    .url(result.getUrl())
                    .build();
                
                images.add(image);
            }
        }
        
        return ImageResponse.builder()
            .images(images)
            .prompt(prompt)
            .options(options)
            .provider("qwen")
            .requestId(response.getRequestId())
            .usage(convertQwenUsage(response.getUsage()))
            .build();
    }
    
    private String convertStyle(ImageOptions.ImageStyle style) {
        switch (style) {
            case VIVID:
                return "<photography>";
            case NATURAL:
                return "<auto>";
            default:
                return "<auto>";
        }
    }
    
    private String convertSize(ImageOptions.ImageSize size) {
        switch (size) {
            case SIZE_512:
                return "512*512";
            case SIZE_1024:
                return "1024*1024";
            case SIZE_1792_1024:
                return "1792*1024";
            case SIZE_1024_1792:
                return "1024*1792";
            default:
                return "1024*1024";
        }
    }
    
    @Override
    public ProviderInfo getProviderInfo() {
        return ProviderInfo.builder()
            .name("Qwen")
            .version("v1")
            .supportedModels(Arrays.asList("wanx-v1"))
            .supportedOperations(Arrays.asList("generate"))
            .maxImageSize(ImageSize.SIZE_1792_1024)
            .supportsBatch(true)
            .supportsAsync(false)
            .build();
    }
}
```

## 4. 图像处理工具类

### 4.1 图像工具类

```java
public class ImageUtils {
    
    /**
     * 从URL加载图像
     */
    public static BufferedImage loadImageFromUrl(String url) throws IOException {
        try (InputStream inputStream = new URL(url).openStream()) {
            return ImageIO.read(inputStream);
        }
    }
    
    /**
     * 从Base64字符串加载图像
     */
    public static BufferedImage loadImageFromBase64(String base64) throws IOException {
        byte[] imageBytes = Base64.getDecoder().decode(base64);
        try (ByteArrayInputStream inputStream = new ByteArrayInputStream(imageBytes)) {
            return ImageIO.read(inputStream);
        }
    }
    
    /**
     * 将图像转换为Base64字符串
     */
    public static String imageToBase64(BufferedImage image, String format) throws IOException {
        try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
            ImageIO.write(image, format, outputStream);
            byte[] imageBytes = outputStream.toByteArray();
            return Base64.getEncoder().encodeToString(imageBytes);
        }
    }
    
    /**
     * 调整图像尺寸
     */
    public static BufferedImage resizeImage(BufferedImage original, int width, int height) {
        Image scaledImage = original.getScaledInstance(width, height, Image.SCALE_SMOOTH);
        BufferedImage resized = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
        
        Graphics2D g2d = resized.createGraphics();
        g2d.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BILINEAR);
        g2d.drawImage(scaledImage, 0, 0, null);
        g2d.dispose();
        
        return resized;
    }
    
    /**
     * 裁剪图像
     */
    public static BufferedImage cropImage(BufferedImage original, int x, int y, int width, int height) {
        return original.getSubimage(x, y, width, height);
    }
    
    /**
     * 创建圆形遮罩
     */
    public static BufferedImage createCircularMask(int width, int height, int centerX, int centerY, int radius) {
        BufferedImage mask = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g2d = mask.createGraphics();
        g2d.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
        
        // 设置透明背景
        g2d.setColor(new Color(0, 0, 0, 0));
        g2d.fillRect(0, 0, width, height);
        
        // 绘制白色圆形
        g2d.setColor(Color.WHITE);
        g2d.fillOval(centerX - radius, centerY - radius, radius * 2, radius * 2);
        
        g2d.dispose();
        return mask;
    }
    
    /**
     * 创建矩形遮罩
     */
    public static BufferedImage createRectangularMask(int width, int height, int x, int y, int rectWidth, int rectHeight) {
        BufferedImage mask = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g2d = mask.createGraphics();
        
        // 设置透明背景
        g2d.setColor(new Color(0, 0, 0, 0));
        g2d.fillRect(0, 0, width, height);
        
        // 绘制白色矩形
        g2d.setColor(Color.WHITE);
        g2d.fillRect(x, y, rectWidth, rectHeight);
        
        g2d.dispose();
        return mask;
    }
    
    /**
     * 检测图像格式
     */
    public static String detectImageFormat(byte[] imageData) throws IOException {
        try (ByteArrayInputStream inputStream = new ByteArrayInputStream(imageData)) {
            Iterator<ImageReader> readers = ImageIO.getImageReaders(inputStream);
            if (readers.hasNext()) {
                ImageReader reader = readers.next();
                return reader.getFormatName().toLowerCase();
            }
        }
        throw new IOException("Unable to detect image format");
    }
    
    /**
     * 计算图像相似度
     */
    public static double calculateImageSimilarity(BufferedImage image1, BufferedImage image2) {
        if (image1.getWidth() != image2.getWidth() || image1.getHeight() != image2.getHeight()) {
            // 调整到相同尺寸
            int width = Math.min(image1.getWidth(), image2.getWidth());
            int height = Math.min(image1.getHeight(), image2.getHeight());
            image1 = resizeImage(image1, width, height);
            image2 = resizeImage(image2, width, height);
        }
        
        long totalPixels = image1.getWidth() * image1.getHeight();
        long differentPixels = 0;
        
        for (int x = 0; x < image1.getWidth(); x++) {
            for (int y = 0; y < image1.getHeight(); y++) {
                if (image1.getRGB(x, y) != image2.getRGB(x, y)) {
                    differentPixels++;
                }
            }
        }
        
        return 1.0 - (double) differentPixels / totalPixels;
    }
}
```

### 4.2 图像验证器

```java
public class ImageValidator {
    
    private static final int MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
    private static final Set<String> SUPPORTED_FORMATS = Set.of("png", "jpg", "jpeg", "webp");
    private static final int MIN_DIMENSION = 64;
    private static final int MAX_DIMENSION = 4096;
    
    /**
     * 验证图像文件
     */
    public static ValidationResult validateImage(File imageFile) {
        ValidationResult result = new ValidationResult();
        
        try {
            // 检查文件大小
            if (imageFile.length() > MAX_FILE_SIZE) {
                result.addError("File size exceeds maximum allowed size: " + MAX_FILE_SIZE + " bytes");
            }
            
            // 检查文件类型
            String format = ImageUtils.detectImageFormat(Files.readAllBytes(imageFile.toPath()));
            if (!SUPPORTED_FORMATS.contains(format)) {
                result.addError("Unsupported image format: " + format);
            }
            
            // 检查图像尺寸
            BufferedImage image = ImageIO.read(imageFile);
            if (image != null) {
                int width = image.getWidth();
                int height = image.getHeight();
                
                if (width < MIN_DIMENSION || height < MIN_DIMENSION) {
                    result.addError(String.format("Image dimensions too small: %dx%d (minimum: %dx%d)", 
                                                 width, height, MIN_DIMENSION, MIN_DIMENSION));
                }
                
                if (width > MAX_DIMENSION || height > MAX_DIMENSION) {
                    result.addError(String.format("Image dimensions too large: %dx%d (maximum: %dx%d)", 
                                                 width, height, MAX_DIMENSION, MAX_DIMENSION));
                }
                
                // 检查宽高比
                double aspectRatio = (double) width / height;
                if (aspectRatio > 4.0 || aspectRatio < 0.25) {
                    result.addWarning("Image aspect ratio is extreme: " + aspectRatio);
                }
            } else {
                result.addError("Unable to read image data");
            }
            
        } catch (Exception e) {
            result.addError("Error validating image: " + e.getMessage());
        }
        
        return result;
    }
    
    /**
     * 验证提示文本
     */
    public static ValidationResult validatePrompt(String prompt) {
        ValidationResult result = new ValidationResult();
        
        if (prompt == null || prompt.trim().isEmpty()) {
            result.addError("Prompt cannot be empty");
            return result;
        }
        
        String trimmedPrompt = prompt.trim();
        
        // 检查长度
        if (trimmedPrompt.length() > 1000) {
            result.addError("Prompt too long: " + trimmedPrompt.length() + " characters (maximum: 1000)");
        }
        
        if (trimmedPrompt.length() < 3) {
            result.addError("Prompt too short: " + trimmedPrompt.length() + " characters (minimum: 3)");
        }
        
        // 检查不当内容
        List<String> inappropriateKeywords = Arrays.asList(
            "nude", "naked", "explicit", "violence", "blood", "gore"
        );
        
        String lowerPrompt = trimmedPrompt.toLowerCase();
        for (String keyword : inappropriateKeywords) {
            if (lowerPrompt.contains(keyword)) {
                result.addWarning("Prompt contains potentially inappropriate content: " + keyword);
            }
        }
        
        return result;
    }
    
    public static class ValidationResult {
        private final List<String> errors = new ArrayList<>();
        private final List<String> warnings = new ArrayList<>();
        
        public void addError(String error) {
            errors.add(error);
        }
        
        public void addWarning(String warning) {
            warnings.add(warning);
        }
        
        public boolean isValid() {
            return errors.isEmpty();
        }
        
        public boolean hasWarnings() {
            return !warnings.isEmpty();
        }
        
        public List<String> getErrors() {
            return errors;
        }
        
        public List<String> getWarnings() {
            return warnings;
        }
    }
}
```

## 5. 高级特性

### 5.1 批量处理器

```java
public class BatchImageProcessor {
    
    private final ImageProvider provider;
    private final ExecutorService executorService;
    private final int batchSize;
    private final int maxConcurrency;
    
    public BatchImageProcessor(ImageProvider provider, BatchConfig config) {
        this.provider = provider;
        this.batchSize = config.getBatchSize();
        this.maxConcurrency = config.getMaxConcurrency();
        
        this.executorService = new ThreadPoolExecutor(
            config.getCorePoolSize(),
            config.getMaxPoolSize(),
            config.getKeepAliveTime(),
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(config.getQueueCapacity()),
            new ThreadFactoryBuilder().setNameFormat("batch-image-processor-%d").build()
        );
    }
    
    /**
     * 批量生成图像
     */
    public CompletableFuture<List<ImageResponse>> batchGenerate(
            List<String> prompts, ImageOptions options) {
        
        // 分组处理
        List<List<String>> batches = partition(prompts, batchSize);
        
        List<CompletableFuture<List<ImageResponse>>> futures = batches.stream()
            .map(batch -> CompletableFuture.supplyAsync(() -> 
                processBatch(batch, options), executorService))
            .collect(Collectors.toList());
        
        // 合并所有批次结果
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .flatMap(future -> future.join().stream())
                .collect(Collectors.toList()));
    }
    
    /**
     * 批量编辑图像
     */
    public CompletableFuture<List<ImageResponse>> batchEdit(
            List<BatchEditRequest> requests, ImageOptions options) {
        
        List<CompletableFuture<ImageResponse>> futures = requests.stream()
            .map(request -> CompletableFuture.supplyAsync(() -> 
                provider.edit(request.getOriginalImage(), request.getPrompt(), 
                             request.getMask(), options), executorService))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }
    
    private List<ImageResponse> processBatch(List<String> prompts, ImageOptions options) {
        List<ImageResponse> results = new ArrayList<>();
        
        for (String prompt : prompts) {
            try {
                ImageResponse response = provider.generate(prompt, options);
                results.add(response);
                
                // 添加延迟以避免API限流
                Thread.sleep(100);
                
            } catch (Exception e) {
                logger.error("Failed to generate image for prompt: {}", prompt, e);
                
                // 创建错误响应
                ImageResponse errorResponse = ImageResponse.builder()
                    .prompt(prompt)
                    .error(e.getMessage())
                    .build();
                results.add(errorResponse);
            }
        }
        
        return results;
    }
    
    private <T> List<List<T>> partition(List<T> list, int size) {
        List<List<T>> partitions = new ArrayList<>();
        for (int i = 0; i < list.size(); i += size) {
            partitions.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return partitions;
    }
    
    public void shutdown() {
        executorService.shutdown();
        try {
            if (!executorService.awaitTermination(60, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
        } catch (InterruptedException e) {
            executorService.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### 5.2 结果缓存

```java
public class ImageCacheManager {
    
    private final Cache<String, ImageResponse> responseCache;
    private final Cache<String, BufferedImage> imageCache;
    private final Path cacheDirectory;
    
    public ImageCacheManager(CacheConfig config) {
        this.cacheDirectory = Paths.get(config.getCacheDirectory());
        
        // 创建缓存目录
        try {
            Files.createDirectories(cacheDirectory);
        } catch (IOException e) {
            throw new RuntimeException("Failed to create cache directory", e);
        }
        
        // 响应缓存
        this.responseCache = Caffeine.newBuilder()
            .maximumSize(config.getMaxResponseCacheSize())
            .expireAfterWrite(config.getResponseCacheExpiration())
            .recordStats()
            .build();
        
        // 图像缓存
        this.imageCache = Caffeine.newBuilder()
            .maximumSize(config.getMaxImageCacheSize())
            .expireAfterWrite(config.getImageCacheExpiration())
            .removalListener(this::onImageEvicted)
            .recordStats()
            .build();
    }
    
    /**
     * 获取缓存的响应
     */
    public ImageResponse getCachedResponse(String prompt, ImageOptions options) {
        String cacheKey = buildCacheKey(prompt, options);
        return responseCache.getIfPresent(cacheKey);
    }
    
    /**
     * 缓存响应
     */
    public void cacheResponse(String prompt, ImageOptions options, ImageResponse response) {
        String cacheKey = buildCacheKey(prompt, options);
        responseCache.put(cacheKey, response);
        
        // 同时缓存图像到磁盘
        for (GeneratedImage image : response.getImages()) {
            cacheImageToDisk(image);
        }
    }
    
    /**
     * 获取缓存的图像
     */
    public BufferedImage getCachedImage(String imageId) {
        BufferedImage cached = imageCache.getIfPresent(imageId);
        if (cached != null) {
            return cached;
        }
        
        // 尝试从磁盘加载
        return loadImageFromDisk(imageId);
    }
    
    /**
     * 缓存图像
     */
    public void cacheImage(String imageId, BufferedImage image) {
        imageCache.put(imageId, image);
        saveImageToDisk(imageId, image);
    }
    
    private String buildCacheKey(String prompt, ImageOptions options) {
        return DigestUtils.md5Hex(prompt + options.toString());
    }
    
    private void cacheImageToDisk(GeneratedImage image) {
        try {
            if (image.getUrl() != null) {
                BufferedImage bufferedImage = ImageUtils.loadImageFromUrl(image.getUrl());
                String imageId = DigestUtils.md5Hex(image.getUrl());
                saveImageToDisk(imageId, bufferedImage);
            }
        } catch (Exception e) {
            logger.warn("Failed to cache image to disk", e);
        }
    }
    
    private void saveImageToDisk(String imageId, BufferedImage image) {
        try {
            Path imagePath = cacheDirectory.resolve(imageId + ".png");
            ImageIO.write(image, "png", imagePath.toFile());
        } catch (IOException e) {
            logger.warn("Failed to save image to disk: {}", imageId, e);
        }
    }
    
    private BufferedImage loadImageFromDisk(String imageId) {
        try {
            Path imagePath = cacheDirectory.resolve(imageId + ".png");
            if (Files.exists(imagePath)) {
                BufferedImage image = ImageIO.read(imagePath.toFile());
                if (image != null) {
                    imageCache.put(imageId, image);
                    return image;
                }
            }
        } catch (IOException e) {
            logger.warn("Failed to load image from disk: {}", imageId, e);
        }
        return null;
    }
    
    private void onImageEvicted(String imageId, BufferedImage image, RemovalCause cause) {
        // 可以选择保留磁盘文件或删除
        if (cause == RemovalCause.SIZE) {
            // 如果是因为大小限制被驱逐，保留磁盘文件
            logger.debug("Image evicted from memory cache due to size limit: {}", imageId);
        }
    }
    
    /**
     * 清理过期的磁盘缓存
     */
    @Scheduled(fixedRate = 3600000) // 每小时执行一次
    public void cleanupExpiredDiskCache() {
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(cacheDirectory, "*.png")) {
            long cutoffTime = System.currentTimeMillis() - Duration.ofDays(7).toMillis();
            
            for (Path path : stream) {
                BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
                if (attrs.creationTime().toMillis() < cutoffTime) {
                    Files.delete(path);
                    logger.debug("Deleted expired cache file: {}", path.getFileName());
                }
            }
        } catch (IOException e) {
            logger.error("Failed to cleanup expired disk cache", e);
        }
    }
    
    /**
     * 获取缓存统计信息
     */
    public CacheStats getCacheStats() {
        return CacheStats.builder()
            .responseCacheStats(responseCache.stats())
            .imageCacheStats(imageCache.stats())
            .diskCacheSize(calculateDiskCacheSize())
            .build();
    }
    
    private long calculateDiskCacheSize() {
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(cacheDirectory)) {
            long totalSize = 0;
            for (Path path : stream) {
                totalSize += Files.size(path);
            }
            return totalSize;
        } catch (IOException e) {
            logger.warn("Failed to calculate disk cache size", e);
            return 0;
        }
    }
}
```

### 5.3 图像质量评估

```java
public class ImageQualityAssessor {
    
    /**
     * 评估图像质量
     */
    public QualityScore assessQuality(BufferedImage image) {
        QualityScore score = new QualityScore();
        
        // 分辨率评分
        score.setResolutionScore(assessResolution(image));
        
        // 清晰度评分
        score.setSharpnessScore(assessSharpness(image));
        
        // 颜色饱和度评分
        score.setSaturationScore(assessSaturation(image));
        
        // 对比度评分
        score.setContrastScore(assessContrast(image));
        
        // 综合评分
        double overallScore = (score.getResolutionScore() * 0.2 +
                              score.getSharpnessScore() * 0.3 +
                              score.getSaturationScore() * 0.25 +
                              score.getContrastScore() * 0.25);
        
        score.setOverallScore(overallScore);
        
        return score;
    }
    
    private double assessResolution(BufferedImage image) {
        int totalPixels = image.getWidth() * image.getHeight();
        
        if (totalPixels >= 1024 * 1024) {
            return 1.0; // 高分辨率
        } else if (totalPixels >= 512 * 512) {
            return 0.8; // 中等分辨率
        } else if (totalPixels >= 256 * 256) {
            return 0.6; // 较低分辨率
        } else {
            return 0.3; // 低分辨率
        }
    }
    
    private double assessSharpness(BufferedImage image) {
        // 使用Sobel算子计算图像梯度
        double[][] sobelX = {{-1, 0, 1}, {-2, 0, 2}, {-1, 0, 1}};
        double[][] sobelY = {{-1, -2, -1}, {0, 0, 0}, {1, 2, 1}};
        
        double totalGradient = 0;
        int pixelCount = 0;
        
        for (int x = 1; x < image.getWidth() - 1; x++) {
            for (int y = 1; y < image.getHeight() - 1; y++) {
                double gx = 0, gy = 0;
                
                for (int i = -1; i <= 1; i++) {
                    for (int j = -1; j <= 1; j++) {
                        int pixel = image.getRGB(x + i, y + j);
                        int gray = (int) (0.299 * ((pixel >> 16) & 0xFF) +
                                         0.587 * ((pixel >> 8) & 0xFF) +
                                         0.114 * (pixel & 0xFF));
                        
                        gx += gray * sobelX[i + 1][j + 1];
                        gy += gray * sobelY[i + 1][j + 1];
                    }
                }
                
                double gradient = Math.sqrt(gx * gx + gy * gy);
                totalGradient += gradient;
                pixelCount++;
            }
        }
        
        double averageGradient = totalGradient / pixelCount;
        
        // 归一化到0-1范围
        return Math.min(1.0, averageGradient / 100.0);
    }
    
    private double assessSaturation(BufferedImage image) {
        double totalSaturation = 0;
        int pixelCount = image.getWidth() * image.getHeight();
        
        for (int x = 0; x < image.getWidth(); x++) {
            for (int y = 0; y < image.getHeight(); y++) {
                int pixel = image.getRGB(x, y);
                int r = (pixel >> 16) & 0xFF;
                int g = (pixel >> 8) & 0xFF;
                int b = pixel & 0xFF;
                
                float[] hsb = Color.RGBtoHSB(r, g, b, null);
                totalSaturation += hsb[1]; // 饱和度分量
            }
        }
        
        return totalSaturation / pixelCount;
    }
    
    private double assessContrast(BufferedImage image) {
        int[] histogram = new int[256];
        int pixelCount = image.getWidth() * image.getHeight();
        
        // 构建灰度直方图
        for (int x = 0; x < image.getWidth(); x++) {
            for (int y = 0; y < image.getHeight(); y++) {
                int pixel = image.getRGB(x, y);
                int gray = (int) (0.299 * ((pixel >> 16) & 0xFF) +
                                 0.587 * ((pixel >> 8) & 0xFF) +
                                 0.114 * (pixel & 0xFF));
                histogram[gray]++;
            }
        }
        
        // 计算标准差作为对比度指标
        double mean = 0;
        for (int i = 0; i < 256; i++) {
            mean += i * histogram[i] / (double) pixelCount;
        }
        
        double variance = 0;
        for (int i = 0; i < 256; i++) {
            double freq = histogram[i] / (double) pixelCount;
            variance += freq * Math.pow(i - mean, 2);
        }
        
        double standardDeviation = Math.sqrt(variance);
        
        // 归一化到0-1范围
        return Math.min(1.0, standardDeviation / 64.0);
    }
    
    public static class QualityScore {
        private double resolutionScore;
        private double sharpnessScore;
        private double saturationScore;
        private double contrastScore;
        private double overallScore;
        
        // getters and setters...
    }
}
```

## 6. 监控与指标

### 6.1 图像生成指标

```java
@Component
public class ImageMetrics {
    
    private final MeterRegistry meterRegistry;
    
    // 计数器
    private final Counter generationCounter;
    private final Counter editCounter;
    private final Counter variationCounter;
    private final Counter errorCounter;
    
    // 计时器
    private final Timer generationTimer;
    private final Timer downloadTimer;
    
    // 分布统计
    private final DistributionSummary imageSizeDistribution;
    private final DistributionSummary qualityScoreDistribution;
    
    public ImageMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.generationCounter = Counter.builder("image.generation.total")
            .description("Total image generations")
            .register(meterRegistry);
            
        this.editCounter = Counter.builder("image.edit.total")
            .description("Total image edits")
            .register(meterRegistry);
            
        this.variationCounter = Counter.builder("image.variation.total")
            .description("Total image variations")
            .register(meterRegistry);
            
        this.errorCounter = Counter.builder("image.error.total")
            .description("Total image processing errors")
            .register(meterRegistry);
            
        this.generationTimer = Timer.builder("image.generation.duration")
            .description("Image generation duration")
            .register(meterRegistry);
            
        this.downloadTimer = Timer.builder("image.download.duration")
            .description("Image download duration")
            .register(meterRegistry);
            
        this.imageSizeDistribution = DistributionSummary.builder("image.size.bytes")
            .description("Generated image size distribution")
            .register(meterRegistry);
            
        this.qualityScoreDistribution = DistributionSummary.builder("image.quality.score")
            .description("Image quality score distribution")
            .register(meterRegistry);
    }
    
    public void recordGeneration(String provider, String model, Duration duration, boolean success) {
        generationCounter.increment(
            Tags.of(
                "provider", provider,
                "model", model,
                "status", success ? "success" : "failure"
            )
        );
        
        if (success) {
            generationTimer.record(duration, Tags.of("provider", provider, "model", model));
        } else {
            errorCounter.increment(Tags.of("provider", provider, "operation", "generation"));
        }
    }
    
    public void recordImageDownload(String provider, long sizeBytes, Duration duration) {
        downloadTimer.record(duration, Tags.of("provider", provider));
        imageSizeDistribution.record(sizeBytes);
    }
    
    public void recordQualityScore(String provider, double score) {
        qualityScoreDistribution.record(score, Tags.of("provider", provider));
    }
    
    public void recordBatchOperation(String operation, int batchSize, int successCount, Duration totalDuration) {
        Counter.builder("image.batch.total")
            .tag("operation", operation)
            .register(meterRegistry)
            .increment();
            
        Gauge.builder("image.batch.success_rate")
            .tag("operation", operation)
            .register(meterRegistry, this, metrics -> (double) successCount / batchSize);
            
        Timer.builder("image.batch.duration")
            .tag("operation", operation)
            .register(meterRegistry)
            .record(totalDuration);
    }
}
```

## 7. 对Sen-AiFlow的启发

### 核心设计理念

1. **统一抽象**: 屏蔽不同图像AI服务的差异
2. **多模式支持**: 生成、编辑、变体等多种操作模式
3. **批量优化**: 高效的批量处理能力
4. **质量控制**: 图像验证和质量评估
5. **缓存策略**: 智能的结果缓存和复用

### Sen-AiFlow响应式图像处理设计

```java
// Sen-AiFlow中的响应式图像处理设计
@SenAiFlowComponent("reactive-image-processor")
public interface ReactiveImageProcessor {
    
    // 响应式图像生成
    Mono<ImageResponse> generate(Mono<String> prompt, ImageOptions options);
    
    // 流式批量生成
    Flux<ImageResponse> generateBatch(Flux<String> prompts, ImageOptions options);
    
    // 响应式图像编辑
    Mono<ImageResponse> edit(Mono<BufferedImage> image, Mono<String> prompt, ImageOptions options);
    
    // 图像质量评估
    Mono<QualityScore> assessQuality(Mono<BufferedImage> image);
    
    // 缓存管理
    Mono<Void> cacheImage(String key, BufferedImage image);
    Mono<BufferedImage> getCachedImage(String key);
    
    // 健康监控
    Flux<ImageProviderHealth> monitorProviders();
}
```

Agents-Flex的Image模块展示了AI图像处理系统的完整解决方案，其多服务集成、统一抽象和质量控制等特性为Sen-AiFlow的图像处理能力提供了重要参考，特别是在保持API一致性的同时支持多种图像AI服务方面的设计经验。 