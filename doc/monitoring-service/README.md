# Monitoring Module - 简化监控模块

## 组件职责

提供基础的健康检查、性能指标、日志记录等监控能力，使用开源组件降低成本。

## 核心功能

### 1. 健康检查
- Spring Boot Actuator集成
- 应用健康状态检查
- 依赖服务健康检查
- 自定义健康指标

### 2. 性能指标
- Micrometer指标收集
- JVM性能监控
- 应用业务指标
- HTTP请求统计

### 3. 日志管理
- Logback结构化日志
- 按模块分级日志
- 日志文件轮转
- 错误日志聚合

### 4. 可选监控
- Prometheus集成 (可选)
- Grafana仪表盘 (可选)
- 简单告警机制

## 技术架构

```
┌──────────────────────────────────┐
│         单体应用程序          │
│ ┌──────────────────────────────┐ │
│ │ Gateway + Signaling + Media │ │
│ │ + AI + Session + Monitor    │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
                │
                ▼
    ┌──────────────────────────────┐
    │       内置监控组件        │
    │ ┌──────────────────────────┐ │
    │ │ Spring Boot Actuator   │ │
    │ │ + Micrometer          │ │
    │ │ + Logback            │ │
    │ └──────────────────────────┘ │
    └──────────────────────────────┘
                │
                ▼
    ┌──────────────────────────────┐
    │      可选外部监控        │
    │ ┌──────────┐ ┌──────────┐ │
    │ │Prometheus│ │ Grafana  │ │
    │ │ (可选)   │ │ (可选)   │ │
    │ └──────────┘ └──────────┘ │
    └──────────────────────────────┘
```

## 技术实现

### Spring Boot Actuator配置
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      show-components: always
    metrics:
      enabled: true
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5,0.95,0.99
```

### 自定义健康检查
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final SessionMapper sessionMapper;
    
    @Override
    public Health health() {
        try {
            // 检查Redis连接
            redisTemplate.opsForValue().set("health:check", "ok", Duration.ofSeconds(10));
            
            // 检查数据库连接
            sessionMapper.selectCount(null);
            
            // 检查WebRTC媒体服务
            // ... kurento health check
            
            return Health.up()
                .withDetail("redis", "UP")
                .withDetail("database", "UP")
                .withDetail("kurento", "UP")
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### 自定义业务指标
```java
@Component
public class BusinessMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter sessionCreatedCounter;
    private final Timer audioProcessingTimer;
    private final Gauge activeSessions;
    
    public BusinessMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.sessionCreatedCounter = Counter.builder("sessions.created.total")
            .description("Total number of sessions created")
            .register(meterRegistry);
            
        this.audioProcessingTimer = Timer.builder("audio.processing.duration")
            .description("Audio processing duration")
            .register(meterRegistry);
            
        this.activeSessions = Gauge.builder("sessions.active.count")
            .description("Number of active sessions")
            .register(meterRegistry, this, BusinessMetrics::getActiveSessionCount);
    }
    
    public void recordSessionCreated() {
        sessionCreatedCounter.increment();
    }
    
    public void recordAudioProcessingTime(Duration duration) {
        audioProcessingTimer.record(duration);
    }
    
    private double getActiveSessionCount() {
        // 从Redis或数据库查询活跃会话数
        return sessionService.getActiveSessionCount();
    }
}
```

### 日志配置
```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="!prod">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
                <providers>
                    <timestamp/>
                    <logLevel/>
                    <loggerName/>
                    <message/>
                    <mdc/>
                    <stackTrace/>
                </providers>
            </encoder>
        </appender>
    </springProfile>
    
    <springProfile name="prod">
        <!-- 生产环境文件日志 -->
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/ai-interaction.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/ai-interaction.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <maxFileSize>100MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>3GB</totalSizeCap>
            </rollingPolicy>
            <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
                <providers>
                    <timestamp/>
                    <logLevel/>
                    <loggerName/>
                    <message/>
                    <mdc/>
                    <stackTrace/>
                </providers>
            </encoder>
        </appender>
    </springProfile>
    
    <!-- 按模块分级日志 -->
    <logger name="com.ai.interaction.session" level="INFO"/>
    <logger name="com.ai.interaction.media" level="WARN"/>
    <logger name="com.ai.interaction.orchestrator" level="INFO"/>
    <logger name="org.kurento" level="WARN"/>
    
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

## 关键指标定义

### 系统指标
```
- jvm.memory.used (JVM内存使用)
- jvm.gc.pause (GC暂停时间)
- system.cpu.usage (CPU使用率)
- disk.free (磁盘空间)
```

### 应用指标
```
- http.server.requests (HTTP请求统计)
- sessions.active.count (活跃会话数)
- sessions.created.total (创建会话总数)
- audio.processing.duration (音频处理耗时)
```

### 业务指标
```
- ai.asr.requests.total (ASR请求总数)
- ai.llm.requests.total (LLM请求总数)
- ai.tts.requests.total (TTS请求总数)
- webrtc.connections.active (WebRTC连接数)
```

## 可选Prometheus集成

### Docker Compose配置
```yaml
# docker-compose.monitoring.yml (可选)
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
      
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana

volumes:
  grafana-storage:
```

### Prometheus配置
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ai-interaction'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
    scrape_interval: 10s
```

## 简单告警

### 基于日志的告警
```java
@Component
public class SimpleAlerting {
    
    private static final Logger alertLogger = LoggerFactory.getLogger("ALERT");
    
    @EventListener
    public void onHighErrorRate(HighErrorRateEvent event) {
        alertLogger.error("ALERT: High error rate detected: {}%", event.getErrorRate());
        // 可以集成邮件、企业微信等通知
    }
    
    @EventListener  
    public void onSystemResourceLow(SystemResourceEvent event) {
        alertLogger.warn("ALERT: System resource low: {} at {}%", 
            event.getResourceType(), event.getUsagePercent());
    }
}
```

## 性能目标

- 监控数据采集延迟: < 10s
- 健康检查响应时间: < 1s
- 日志写入吞吐: 10000+ 条/秒
- 指标存储开销: < 5% CPU

## 部署方案

### 最小化部署
```bash
# 仅使用内置监控，无外部依赖
java -jar ai-interaction.jar
```

### 增强监控部署
```bash
# 启用Prometheus + Grafana
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
```

## 访问端点

```
# 健康检查
GET /actuator/health

# 应用信息
GET /actuator/info

# 指标数据
GET /actuator/metrics
GET /actuator/metrics/{metricName}

# Prometheus格式指标
GET /actuator/prometheus

# 日志级别管理
GET /actuator/loggers
POST /actuator/loggers/{logger}
```

---

*技术方案版本: v1.0*
*最后更新: 2024-12-XX*