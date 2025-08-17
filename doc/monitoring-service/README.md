# Monitoring Service - 可观测性服务

## 组件职责

提供全链路分布式追踪、指标收集、日志聚合、告警通知等可观测性能力。

## 核心功能

### 1. 分布式追踪
- OpenTelemetry集成
- 全链路 trace追踪
- 性能癃颈定位
- 调用链可视化

### 2. 指标收集
- Prometheus指标采集
- 媒体质量指标
- 系统性能指标
- AI服务指标

### 3. 日志管理
- 结构化日志输出
- 日志聚合与分析
- 实时日志流
- 日志检索与查询

### 4. 告警系统
- 实时告警规则
- 多渠道通知
- 告警级联与升级
- 静默期管理

## 技术架构

```
┌──────────────────────────────────┐
│         应用服务集群          │
│ ┌──────────────────────────────┐ │
│ │ Gateway + Signaling + Media │ │
│ │ + AI + Session + Edge      │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
             │        │        │
             ▼        ▼        ▼
       ┌────────┐ ┌────────┐ ┌────────┐
       │ Traces │ │Metrics│ │  Logs  │
       └────────┘ └────────┘ └────────┘
             │        │        │
             ▼        ▼        ▼
    ┌──────────────────────────────────┐
    │        监控服务集群        │
    │ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
    │ │ Jaeger  │ │Prometheus│ │ ELK Stack│ │
    │ └──────────┘ └──────────┘ └──────────┘ │
    └──────────────────────────────────┘
                         │
                         ▼
                ┌──────────────────┐
                │   Grafana 仪表盘   │
                │ + AlertManager │
                └──────────────────┘
```

## OpenTelemetry集成

```java
@Configuration
public class TracingConfiguration {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        return OpenTelemetrySdk.builder()
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .addSpanProcessor(BatchSpanProcessor.builder(
                        JaegerGrpcSpanExporter.builder()
                            .setEndpoint("http://jaeger:14250")
                            .build()
                    ).build())
                    .setResource(Resource.getDefault()
                        .merge(Resource.create(
                            Attributes.of(ResourceAttributes.SERVICE_NAME, "ai-interaction")
                        ))
                    )
                    .build()
            )
            .buildAndRegisterGlobal();
    }
}

@Component
public class TracingInterceptor {
    
    @EventListener
    public void onAudioProcessingStart(AudioProcessingEvent event) {
        Span span = tracer.spanBuilder("audio.processing")
            .setAttribute("session.id", event.getSessionId())
            .setAttribute("audio.duration", event.getDuration())
            .startSpan();
            
        // 将span存储到上下文中
        Context.current().with(span).makeCurrent();
    }
}
```

## 指标体系

### 媒体质量指标
```
- media_audio_jitter_ms (P99音频抖动)
- media_audio_packet_loss_rate (丢包率)
- media_audio_rtt_ms (往返时延)
- media_codec_bitrate_kbps (编码码率)
```

### AI服务指标
```
- ai_asr_latency_ms (ASR延迟)
- ai_llm_latency_ms (LLM延迟)
- ai_tts_latency_ms (TTS延迟)
- ai_orchestration_success_rate (成功率)
```

### 系统指标
```
- system_cpu_usage_percent (CPU使用率)
- system_memory_usage_bytes (内存使用)
- system_gc_duration_ms (GC延迟)
- system_active_connections (活跃连接数)
```

## 告警规则

```yaml
groups:
  - name: ai-interaction.rules
    rules:
      - alert: HighAudioLatency
        expr: media_audio_jitter_ms{quantile="0.99"} > 300
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "音频延迟过高"
          
      - alert: AIServiceDown
        expr: up{job="ai-orchestrator"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "AI服务不可用"
```

## 性能目标

- 指标采集延迟: < 10s
- 日志处理吐吐: 100000+ 条/秒
- 告警响应时间: < 30s
- 数据保留期: 30天

---

*技术方案版本: v1.0*