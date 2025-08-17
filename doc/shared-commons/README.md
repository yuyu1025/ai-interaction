# Shared Commons - 公共组件库

## 组件职责

提供跨服务的公共工具类、数据结构、常量定义、异常处理等基础组件。

## 核心模块

### 1. 公共数据结构
- 会话相关实体
- AI服务请求/响应模型
- 网络协议数据结构
- 配置相关实体

### 2. 工具类
- Protobuf序列化工具
- 加密/解密工具
- 时间处理工具
- 网络工具
- MyBatis-Plus工具类

### 3. 常量与配置
- 系统常量定义
- 错误码定义
- 配置键名定义
- 默认值管理

### 4. 异常处理
- 业务异常定义
- 统一异常处理器
- 错误响应格式
- 重试机制

## 技术架构

```
┌──────────────────────────────────┐
│        应用服务层         │
│ ┌──────────────────────────────┐ │
│ │ Gateway + Signaling + Media │ │
│ │ + AI + Session + Edge + Mon │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
                      │
                      ▼ (depends on)
    ┌──────────────────────────────────┐
    │         Shared Commons        │
    │ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
    │ │  Models  │ │ Utils   │ │Constants│ │
    │ └──────────┘ └──────────┘ └──────────┘ │
    │ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
    │ │Exception│ │Validator│ │ Config  │ │
    │ └──────────┘ └──────────┘ └──────────┘ │
    └──────────────────────────────────┘
```

## 核心实体定义

### 会话相关
```java
// 会话信息
@Data
@Builder
public class SessionInfo {
    private String sessionId;
    private String userId;
    private SessionStatus status;
    private Instant createdAt;
    private Instant lastActivityAt;
    private SessionConfiguration configuration;
    private ResourceAllocation resourceAllocation;
}

// 会话状态
public enum SessionStatus {
    INITIALIZING,
    ACTIVE, 
    IDLE,
    SUSPENDED,
    TERMINATED,
    FAILED
}

// 资源分配
@Data
@Builder
public class ResourceAllocation {
    private String sessionId;
    private ServiceNode mediaServiceNode;
    private ServiceNode aiServiceNode;
    private ServiceNode signalingServiceNode;
    private Instant allocatedAt;
}
```

### AI服务模型
```java
// ASR结果
@Data
@Builder
public class TranscriptionResult {
    private String text;
    private double confidence;
    private String language;
    private Instant timestamp;
    private List<Word> words;
}

// LLM响应
@Data
@Builder
public class LLMResponse {
    private String text;
    private String sessionId;
    private Instant timestamp;
    private Map<String, Object> metadata;
}

// TTS音频结果
@Data
@Builder
public class AudioResult {
    private byte[] audioData;
    private String format; // wav, mp3, etc.
    private int sampleRate;
    private Duration duration;
    private Map<String, Object> metadata;
}
```

### 网络协议
```java
// WebRTC信令消息
@Data
@Builder
public class SignalingMessage {
    private String sessionId;
    private MessageType type;
    private String payload;
    private Instant timestamp;
    private String sequenceId;
}

// 网络统计信息
@Data
@Builder
public class NetworkStats {
    private double packetLoss;
    private int jitterMs;
    private int roundTripTimeMs;
    private int bandwidth;
    private NetworkQuality quality;
}
```

## 工具类

### JSON序列化工具
```java
public class JsonUtils {
    
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
        .registerModule(new JavaTimeModule())
        .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    
    public static <T> String toJson(T object) {
        try {
            return OBJECT_MAPPER.writeValueAsString(object);
        } catch (JsonProcessingException e) {
            throw new SerializationException("Failed to serialize object to JSON", e);
        }
    }
    
    public static <T> T fromJson(String json, Class<T> clazz) {
        try {
            return OBJECT_MAPPER.readValue(json, clazz);
        } catch (JsonProcessingException e) {
            throw new DeserializationException("Failed to deserialize JSON to object", e);
        }
    }
    
    public static <T> T fromJson(String json, TypeReference<T> typeRef) {
        try {
            return OBJECT_MAPPER.readValue(json, typeRef);
        } catch (JsonProcessingException e) {
            throw new DeserializationException("Failed to deserialize JSON to object", e);
        }
    }
}
```

### 加密工具
```java
public class EncryptionUtils {
    
    private static final String AES_ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 16;
    
    public static String encrypt(String plainText, String secretKey) {
        try {
            SecretKeySpec keySpec = new SecretKeySpec(secretKey.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
            
            byte[] iv = new byte[GCM_IV_LENGTH];
            new SecureRandom().nextBytes(iv);
            GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
            
            cipher.init(Cipher.ENCRYPT_MODE, keySpec, parameterSpec);
            byte[] encryptedData = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));
            
            // 合并IV和加密数据
            byte[] encryptedWithIv = new byte[GCM_IV_LENGTH + encryptedData.length];
            System.arraycopy(iv, 0, encryptedWithIv, 0, GCM_IV_LENGTH);
            System.arraycopy(encryptedData, 0, encryptedWithIv, GCM_IV_LENGTH, encryptedData.length);
            
            return Base64.getEncoder().encodeToString(encryptedWithIv);
        } catch (Exception e) {
            throw new EncryptionException("Failed to encrypt data", e);
        }
    }
    
    public static String decrypt(String encryptedText, String secretKey) {
        try {
            byte[] decodedData = Base64.getDecoder().decode(encryptedText);
            
            // 提取IV和加密数据
            byte[] iv = new byte[GCM_IV_LENGTH];
            System.arraycopy(decodedData, 0, iv, 0, GCM_IV_LENGTH);
            
            byte[] encryptedData = new byte[decodedData.length - GCM_IV_LENGTH];
            System.arraycopy(decodedData, GCM_IV_LENGTH, encryptedData, 0, encryptedData.length);
            
            SecretKeySpec keySpec = new SecretKeySpec(secretKey.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
            GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
            
            cipher.init(Cipher.DECRYPT_MODE, keySpec, parameterSpec);
            byte[] decryptedData = cipher.doFinal(encryptedData);
            
            return new String(decryptedData, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new DecryptionException("Failed to decrypt data", e);
        }
    }
}
```

### 时间处理工具
```java
public class TimeUtils {
    
    public static final ZoneId UTC = ZoneId.of("UTC");
    public static final ZoneId BEIJING = ZoneId.of("Asia/Shanghai");
    
    public static Instant now() {
        return Instant.now();
    }
    
    public static String formatISO8601(Instant instant) {
        return instant.atZone(UTC).format(DateTimeFormatter.ISO_INSTANT);
    }
    
    public static Instant parseISO8601(String dateString) {
        return Instant.from(DateTimeFormatter.ISO_INSTANT.parse(dateString));
    }
    
    public static long toEpochMilli(Instant instant) {
        return instant.toEpochMilli();
    }
    
    public static Instant fromEpochMilli(long epochMilli) {
        return Instant.ofEpochMilli(epochMilli);
    }
    
    public static Duration between(Instant start, Instant end) {
        return Duration.between(start, end);
    }
    
    public static boolean isExpired(Instant expiry) {
        return Instant.now().isAfter(expiry);
    }
}
```

## 常量定义

```java
public class SystemConstants {
    
    // 系统级常量
    public static final String SYSTEM_NAME = "ai-interaction";
    public static final String API_VERSION = "v1";
    public static final String DEFAULT_CHARSET = "UTF-8";
    
    // 会话相关
    public static final int DEFAULT_SESSION_TIMEOUT_MINUTES = 30;
    public static final int MAX_SESSION_IDLE_MINUTES = 10;
    public static final String SESSION_ID_PREFIX = "sess_";
    
    // 音频相关
    public static final int DEFAULT_SAMPLE_RATE = 16000;
    public static final String DEFAULT_AUDIO_FORMAT = "wav";
    public static final int MAX_AUDIO_DURATION_SECONDS = 60;
    
    // AI服务相关
    public static final int DEFAULT_ASR_TIMEOUT_MS = 3000;
    public static final int DEFAULT_LLM_TIMEOUT_MS = 10000;
    public static final int DEFAULT_TTS_TIMEOUT_MS = 5000;
    public static final int MAX_LLM_TOKENS = 500;
}

public class ErrorCodes {
    
    // 通用错误码
    public static final String SUCCESS = "0000";
    public static final String SYSTEM_ERROR = "9999";
    public static final String INVALID_PARAMETER = "1001";
    public static final String UNAUTHORIZED = "1002";
    public static final String FORBIDDEN = "1003";
    public static final String NOT_FOUND = "1004";
    
    // 会话相关错误码
    public static final String SESSION_NOT_FOUND = "2001";
    public static final String SESSION_EXPIRED = "2002";
    public static final String SESSION_LIMIT_EXCEEDED = "2003";
    
    // AI服务错误码
    public static final String ASR_SERVICE_ERROR = "3001";
    public static final String LLM_SERVICE_ERROR = "3002";
    public static final String TTS_SERVICE_ERROR = "3003";
    public static final String AI_SERVICE_TIMEOUT = "3004";
    
    // 媒体相关错误码
    public static final String MEDIA_CONNECTION_FAILED = "4001";
    public static final String AUDIO_PROCESSING_ERROR = "4002";
    public static final String NETWORK_QUALITY_POOR = "4003";
}
```

## 异常处理

### 自定义异常
```java
// 基础业务异常
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;
    private final Object[] args;
    
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.args = new Object[0];
    }
    
    public BusinessException(String errorCode, String message, Object... args) {
        super(message);
        this.errorCode = errorCode;
        this.args = args;
    }
    
    public String getErrorCode() {
        return errorCode;
    }
    
    public Object[] getArgs() {
        return args;
    }
}

// 会话相关异常
public class SessionNotFoundException extends BusinessException {
    public SessionNotFoundException(String sessionId) {
        super(ErrorCodes.SESSION_NOT_FOUND, "Session not found: {0}", sessionId);
    }
}

public class SessionExpiredException extends BusinessException {
    public SessionExpiredException(String sessionId) {
        super(ErrorCodes.SESSION_EXPIRED, "Session expired: {0}", sessionId);
    }
}

// AI服务相关异常
public class AIServiceException extends BusinessException {
    public AIServiceException(String serviceType, String message) {
        super("3001", "AI service error [{0}]: {1}", serviceType, message);
    }
}
```

### 全局异常处理器
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.warn("Business exception: {}", e.getMessage());
        
        ErrorResponse errorResponse = ErrorResponse.builder()
            .code(e.getErrorCode())
            .message(e.getMessage())
            .timestamp(TimeUtils.now())
            .build();
            
        return ResponseEntity.badRequest().body(errorResponse);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnknownException(Exception e) {
        log.error("Unknown exception occurred", e);
        
        ErrorResponse errorResponse = ErrorResponse.builder()
            .code(ErrorCodes.SYSTEM_ERROR)
            .message("Internal server error")
            .timestamp(TimeUtils.now())
            .build();
            
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}

@Data
@Builder
public class ErrorResponse {
    private String code;
    private String message;
    private Instant timestamp;
    private Map<String, Object> details;
}
```

## 验证工具

```java
public class ValidationUtils {
    
    public static void notNull(Object object, String fieldName) {
        if (object == null) {
            throw new IllegalArgumentException(fieldName + " cannot be null");
        }
    }
    
    public static void notEmpty(String str, String fieldName) {
        if (str == null || str.trim().isEmpty()) {
            throw new IllegalArgumentException(fieldName + " cannot be empty");
        }
    }
    
    public static void notEmpty(Collection<?> collection, String fieldName) {
        if (collection == null || collection.isEmpty()) {
            throw new IllegalArgumentException(fieldName + " cannot be empty");
        }
    }
    
    public static void isTrue(boolean expression, String message) {
        if (!expression) {
            throw new IllegalArgumentException(message);
        }
    }
    
    public static void inRange(int value, int min, int max, String fieldName) {
        if (value < min || value > max) {
            throw new IllegalArgumentException(
                String.format("%s must be between %d and %d, but was %d", 
                fieldName, min, max, value));
        }
    }
    
    public static boolean isValidSessionId(String sessionId) {
        return sessionId != null && sessionId.startsWith(SystemConstants.SESSION_ID_PREFIX);
    }
    
    public static boolean isValidEmail(String email) {
        return email != null && email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
}
```

## MyBatis-Plus基础类

### 通用基础实体
```java
@Data
public abstract class BaseEntity {
    
    @TableField(value = "create_time", fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(value = "update_time", fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
    
    @TableLogic
    @TableField("deleted")
    private Boolean deleted;
}
```

### 自动填充处理器
```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "deleted", Boolean.class, false);
    }
    
    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

### JSON类型处理器
```java
@Component
public class JsonTypeHandler<T> extends BaseTypeHandler<T> {
    
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final Class<T> type;
    
    public JsonTypeHandler(Class<T> type) {
        this.type = type;
    }
    
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        try {
            ps.setString(i, objectMapper.writeValueAsString(parameter));
        } catch (JsonProcessingException e) {
            throw new SQLException("Failed to serialize object to JSON", e);
        }
    }
    
    @Override
    public T getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String json = rs.getString(columnName);
        return parseJson(json);
    }
    
    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String json = rs.getString(columnIndex);
        return parseJson(json);
    }
    
    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String json = cs.getString(columnIndex);
        return parseJson(json);
    }
    
    private T parseJson(String json) {
        if (json == null || json.trim().isEmpty()) {
            return null;
        }
        try {
            return objectMapper.readValue(json, type);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to deserialize JSON to object", e);
        }
    }
}
```

## Maven配置

```xml
<project>
    <groupId>com.ai.interaction</groupId>
    <artifactId>shared-commons</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <dependencies>
        <!-- MyBatis-Plus -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
        </dependency>
        
        <!-- JSON处理 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>
        
        <!-- Protocol Buffers -->
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
        </dependency>
        
        <!-- 数据验证 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- 工具类 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>
        
        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

---

*技术方案版本: v1.0*