# AI Orchestrator - AI服务编排引擎

## 组件职责

负责AI服务的统一调度、流水线编排、服务适配、结果聚合等，实现ASR→LLM→TTS的完整AI交互链路。

## 核心功能

### 1. AI服务抽象层
- 统一的AI服务接口定义
- 多厂商AI服务适配器 (OpenAI、百度、阿里云等)
- 服务能力发现与注册
- 动态服务选择与负载均衡

### 2. 服务编排引擎
- 基于规则的服务编排
- 支持同步/异步调用模式
- 分支条件与循环控制
- 错误处理与重试机制

### 3. 实时流处理
- 流式ASR识别处理
- 实时语音活动检测 (VAD)
- 分段语音处理与缓冲
- 低延迟TTS流式合成

### 4. 上下文管理
- 对话历史维护
- 语义上下文理解
- 个性化配置管理
- 会话状态持久化

## 技术架构

### 技术栈选型
```
- Spring Boot + WebFlux (异步响应式)
- Spring Cloud OpenFeign (服务调用)
- RxJava (响应式流处理)
- Redis (上下文缓存)
- Apache Kafka (事件流)
- Drools (规则引擎)
```

### 架构设计
```
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│  Media Service  │───►│   AI Orchestrator    │───►│   AI Services   │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
      Audio Stream              │                     ┌──────────────┐
                                │                     │ ASR Service  │
                                ▼                     │ LLM Service  │
                        ┌──────────────┐              │ TTS Service  │
                        │ Orchestration│              │ VAD Service  │
                        │ Engine       │              └──────────────┘
                        │ ┌──────────┐ │
                        │ │Rule-based│ │
                        │ │Workflow  │ │
                        │ └──────────┘ │
                        │ ┌──────────┐ │
                        │ │Context   │ │
                        │ │Manager   │ │
                        │ └──────────┘ │
                        └──────────────┘
                                │
                                ▼
                        ┌──────────────┐
                        │Redis Cluster │
                        │- Contexts    │
                        │- Sessions    │
                        │- Cache       │
                        └──────────────┘
```

### 核心组件

#### 1. AI Service Registry
```java
@Service
public class AIServiceRegistry {
    
    private final Map<ServiceType, List<AIServiceAdapter>> serviceAdapters = new ConcurrentHashMap<>();
    
    @PostConstruct
    public void initializeAdapters() {
        // ASR服务适配器
        registerAdapter(ServiceType.ASR, new BaiduASRAdapter());
        registerAdapter(ServiceType.ASR, new AliCloudASRAdapter());
        
        // LLM服务适配器
        registerAdapter(ServiceType.LLM, new OpenAIAdapter());
        registerAdapter(ServiceType.LLM, new QianWenAdapter());
        
        // TTS服务适配器
        registerAdapter(ServiceType.TTS, new XunFeiTTSAdapter());
        registerAdapter(ServiceType.TTS, new BaiduTTSAdapter());
    }
    
    public AIServiceAdapter selectAdapter(ServiceType type, SelectionStrategy strategy) {
        List<AIServiceAdapter> adapters = serviceAdapters.get(type);
        return strategy.select(adapters);
    }
}
```

#### 2. Orchestration Engine
```java
@Component
public class OrchestrationEngine {
    
    public Mono<AIResponse> processAudioInput(String sessionId, AudioChunk audioChunk) {
        return Mono.fromCallable(() -> audioChunk)
            .flatMap(audio -> vadService.detectVoiceActivity(sessionId, audio))
            .filter(VoiceActivityResult::hasVoice)
            .flatMap(vadResult -> asrService.transcribe(sessionId, vadResult.getAudio()))
            .flatMap(transcription -> contextManager.enrichContext(sessionId, transcription))
            .flatMap(enrichedInput -> llmService.generateResponse(sessionId, enrichedInput))
            .flatMap(llmResponse -> ttsService.synthesize(sessionId, llmResponse))
            .flatMap(audioResponse -> mediaService.sendAudioToClient(sessionId, audioResponse));
    }
    
    @EventListener
    public void handleVoiceEndEvent(VoiceEndEvent event) {
        // 用户停止说话时触发处理流程
        processAudioInput(event.getSessionId(), event.getAudioBuffer())
            .subscribe(
                response -> log.info("AI response generated for session: {}", event.getSessionId()),
                error -> log.error("AI processing failed for session: {}", event.getSessionId(), error)
            );
    }
}
```

#### 3. Context Manager
```java
@Service
public class ContextManager {
    
    public Mono<EnrichedContext> enrichContext(String sessionId, TranscriptionResult transcription) {
        return getDialogHistory(sessionId)
            .map(history -> {
                // 更新对话历史
                history.addUserInput(transcription.getText());
                
                // 构建上下文
                return EnrichedContext.builder()
                    .sessionId(sessionId)
                    .currentInput(transcription.getText())
                    .dialogHistory(history.getRecentMessages(10)) // 保留最近10轮对话
                    .userProfile(getUserProfile(sessionId))
                    .timestamp(Instant.now())
                    .build();
            })
            .flatMap(context -> cacheContext(sessionId, context));
    }
    
    private Mono<DialogHistory> getDialogHistory(String sessionId) {
        return redisTemplate.opsForValue()
            .get("dialog:history:" + sessionId)
            .cast(DialogHistory.class)
            .switchIfEmpty(Mono.just(new DialogHistory()));
    }
}
```

## AI服务适配器

### 1. ASR服务适配器
```java
public interface ASRAdapter extends AIServiceAdapter {
    Mono<TranscriptionResult> transcribe(String sessionId, AudioChunk audio);
    Flux<PartialTranscriptionResult> transcribeStreaming(String sessionId, Flux<AudioChunk> audioStream);
}

@Component
public class BaiduASRAdapter implements ASRAdapter {
    
    @Override
    public Mono<TranscriptionResult> transcribe(String sessionId, AudioChunk audio) {
        return webClient.post()
            .uri("/speech/v1/asr")
            .header("Authorization", "Bearer " + getAccessToken())
            .bodyValue(buildASRRequest(audio))
            .retrieve()
            .bodyToMono(BaiduASRResponse.class)
            .map(this::convertToTranscriptionResult)
            .doOnError(error -> log.error("ASR transcription failed for session: {}", sessionId, error));
    }
    
    private TranscriptionResult convertToTranscriptionResult(BaiduASRResponse response) {
        return TranscriptionResult.builder()
            .text(response.getResult())
            .confidence(response.getConfidence())
            .language(response.getLanguage())
            .timestamp(Instant.now())
            .build();
    }
}
```

### 2. LLM服务适配器
```java
public interface LLMAdapter extends AIServiceAdapter {
    Mono<LLMResponse> generateResponse(String sessionId, EnrichedContext context);
    Flux<String> generateStreamingResponse(String sessionId, EnrichedContext context);
}

@Component
public class OpenAIAdapter implements LLMAdapter {
    
    @Override
    public Mono<LLMResponse> generateResponse(String sessionId, EnrichedContext context) {
        OpenAIRequest request = buildOpenAIRequest(context);
        
        return webClient.post()
            .uri("/v1/chat/completions")
            .header("Authorization", "Bearer " + apiKey)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(OpenAIResponse.class)
            .map(this::convertToLLMResponse)
            .timeout(Duration.ofSeconds(10)) // 10秒超时
            .retry(2); // 重试2次
    }
    
    private OpenAIRequest buildOpenAIRequest(EnrichedContext context) {
        List<Message> messages = new ArrayList<>();
        
        // 添加系统提示
        messages.add(Message.system("你是一个有用的AI助手，请简洁友好地回答用户问题。"));
        
        // 添加历史对话
        context.getDialogHistory().forEach(dialog -> {
            messages.add(Message.user(dialog.getUserInput()));
            messages.add(Message.assistant(dialog.getAiResponse()));
        });
        
        // 添加当前输入
        messages.add(Message.user(context.getCurrentInput()));
        
        return OpenAIRequest.builder()
            .model("gpt-3.5-turbo")
            .messages(messages)
            .maxTokens(200)
            .temperature(0.7)
            .build();
    }
}
```

### 3. TTS服务适配器
```java
public interface TTSAdapter extends AIServiceAdapter {
    Mono<AudioResult> synthesize(String sessionId, LLMResponse text);
    Flux<AudioChunk> synthesizeStreaming(String sessionId, String text);
}

@Component
public class XunFeiTTSAdapter implements TTSAdapter {
    
    @Override
    public Mono<AudioResult> synthesize(String sessionId, LLMResponse llmResponse) {
        TTSRequest request = TTSRequest.builder()
            .text(llmResponse.getText())
            .voice("xiaoyan") // 晓燕声音
            .speed(5) // 语速
            .volume(50) // 音量
            .audioFormat("wav")
            .sampleRate(16000)
            .build();
            
        return webClient.post()
            .uri("/v1/tts")
            .header("Authorization", buildAuthHeader())
            .bodyValue(request)
            .retrieve()
            .bodyToMono(byte[].class)
            .map(audioData -> AudioResult.builder()
                .audioData(audioData)
                .format("wav")
                .sampleRate(16000)
                .duration(calculateDuration(audioData))
                .build());
    }
}
```

## 编排规则引擎

### 规则定义
```java
@Component
public class OrchestrationRules {
    
    @EventListener
    @ConditionalOnProperty("ai.interruption.enabled")
    public void handleUserInterruption(UserInterruptionEvent event) {
        // 用户打断规则
        String sessionId = event.getSessionId();
        
        // 停止当前TTS播放
        ttsService.stopSynthesis(sessionId);
        
        // 清空输出缓冲
        mediaService.clearAudioBuffer(sessionId);
        
        // 重新开始ASR监听
        asrService.startListening(sessionId);
    }
    
    @EventListener
    public void handleLongSilence(SilenceDetectedEvent event) {
        if (event.getSilenceDuration() > Duration.ofSeconds(3)) {
            // 长时间静音：主动询问
            LLMResponse prompt = LLMResponse.builder()
                .text("还有什么我可以帮助您的吗？")
                .build();
                
            ttsService.synthesize(event.getSessionId(), prompt)
                .subscribe();
        }
    }
}
```

### 工作流配置
```yaml
ai:
  orchestration:
    workflows:
      default:
        steps:
          - name: vad
            service: voice-activity-detection
            timeout: 5000ms
          - name: asr  
            service: speech-to-text
            timeout: 3000ms
            retry: 2
          - name: llm
            service: large-language-model
            timeout: 10000ms
            retry: 1
          - name: tts
            service: text-to-speech
            timeout: 5000ms
            retry: 2
        
        rules:
          - condition: "asr.confidence < 0.7"
            action: "request_repeat"
          - condition: "llm.response.length > 500"
            action: "split_response"
```

## 性能优化

### 1. 并行处理
```java
@Service
public class ParallelOrchestrationService {
    
    public Mono<AIResponse> processWithParallelOptimization(String sessionId, AudioChunk audio) {
        // ASR和情感分析并行处理
        Mono<TranscriptionResult> asrMono = asrService.transcribe(sessionId, audio);
        Mono<EmotionResult> emotionMono = emotionService.analyze(sessionId, audio);
        
        return Mono.zip(asrMono, emotionMono)
            .flatMap(tuple -> {
                TranscriptionResult transcription = tuple.getT1();
                EmotionResult emotion = tuple.getT2();
                
                // 结合文本和情感信息生成上下文
                EnrichedContext context = buildContextWithEmotion(sessionId, transcription, emotion);
                
                return llmService.generateResponse(sessionId, context);
            })
            .flatMap(llmResponse -> ttsService.synthesize(sessionId, llmResponse));
    }
}
```

### 2. 缓存策略
```java
@Service
public class CacheOptimizedOrchestrator {
    
    @Cacheable(value = "llm-responses", key = "#context.hash()")
    public Mono<LLMResponse> generateResponseWithCache(String sessionId, EnrichedContext context) {
        // 对相似问题使用缓存响应
        return llmService.generateResponse(sessionId, context);
    }
    
    @Cacheable(value = "tts-audio", key = "#text.hash() + '_' + #voice")
    public Mono<AudioResult> synthesizeWithCache(String text, String voice) {
        // 对相同文本和声音组合使用缓存音频
        return ttsService.synthesize(text, voice);
    }
}
```

## 监控与指标

### 关键指标
```java
@Component
public class OrchestrationMetrics {
    
    @EventListener
    public void onOrchestrationComplete(OrchestrationCompleteEvent event) {
        Duration totalLatency = event.getTotalLatency();
        Map<String, Duration> stepLatencies = event.getStepLatencies();
        
        // 总体延迟指标
        meterRegistry.timer("ai.orchestration.total_latency").record(totalLatency);
        
        // 各步骤延迟指标
        stepLatencies.forEach((step, latency) -> {
            meterRegistry.timer("ai.orchestration.step_latency", Tags.of("step", step)).record(latency);
        });
        
        // 成功率指标
        meterRegistry.counter("ai.orchestration.success").increment();
    }
    
    @EventListener
    public void onOrchestrationError(OrchestrationErrorEvent event) {
        meterRegistry.counter("ai.orchestration.error", 
            Tags.of("step", event.getFailedStep(), "error", event.getErrorType())).increment();
    }
}
```

### 性能目标
- AI处理总延迟: P99 < 1000ms
- ASR延迟: P95 < 300ms  
- LLM延迟: P95 < 800ms
- TTS延迟: P95 < 500ms
- 成功率: > 99%

## 接口设计

### REST API
```http
# 编排配置管理
POST   /api/orchestration/workflows
GET    /api/orchestration/workflows
PUT    /api/orchestration/workflows/{id}
DELETE /api/orchestration/workflows/{id}

# 会话管理
POST   /api/orchestration/sessions
GET    /api/orchestration/sessions/{sessionId}
DELETE /api/orchestration/sessions/{sessionId}

# 上下文管理
GET    /api/orchestration/sessions/{sessionId}/context
PUT    /api/orchestration/sessions/{sessionId}/context

# 监控指标
GET    /api/orchestration/metrics
GET    /api/orchestration/health
```

### 事件API
```java
public interface OrchestrationEventListener {
    
    // 音频输入事件
    void onAudioInputReceived(AudioInputEvent event);
    
    // VAD检测事件  
    void onVoiceActivityDetected(VoiceActivityEvent event);
    
    // ASR结果事件
    void onTranscriptionCompleted(TranscriptionEvent event);
    
    // LLM响应事件
    void onLLMResponseGenerated(LLMResponseEvent event);
    
    // TTS合成事件
    void onTTSSynthesisCompleted(TTSEvent event);
    
    // 错误事件
    void onOrchestrationError(OrchestrationErrorEvent event);
}
```

## 部署方案

### 容器化部署
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-orchestrator
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ai-orchestrator
  template:
    spec:
      containers:
      - name: ai-orchestrator
        image: ai-interaction/ai-orchestrator:latest
        ports:
        - containerPort: 8083
        env:
        - name: REDIS_CLUSTER_NODES
          value: "redis1:6379,redis2:6379,redis3:6379"
        - name: KAFKA_BROKERS
          value: "kafka1:9092,kafka2:9092"
        resources:
          requests:
            memory: "1Gi"
            cpu: "1000m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
```

## 测试策略

### 单元测试
```java
@Test
public void testOrchestrationFlow() {
    // Mock各个AI服务
    // 验证编排逻辑正确性
    // 测试错误处理机制
}
```

### 集成测试
```java
@Test  
public void testEndToEndOrchestration() {
    // 端到端AI交互测试
    // 验证各服务适配器集成
    // 测试并发处理能力
}
```

### 性能测试
```java
@Test
public void testOrchestrationLatency() {
    // 延迟分布统计
    // 并发处理压力测试
    // 资源使用率监控
}
```

---

*技术方案版本: v1.0*
*最后更新: 2024-12-XX*