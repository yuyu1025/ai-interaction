# Media Service - 媒体处理服务

## 组件职责

基于Kurento媒体服务器，负责WebRTC媒体流处理、音频编解码、弱网对抗、音频预处理等核心媒体功能。

## 核心功能

### 1. WebRTC媒体传输
- 基于UDP的实时音频传输 (RTP/RTCP)
- SRTP加密音频流传输
- ICE连接建立与NAT穿越
- DTLS密钥交换与安全协商

### 2. 音频编解码
- OPUS音频编码 (2.5kbps-510kbps动态码率)
- 实时编码参数调整 (采样率、比特率、复杂度)
- 音频格式转换 (PCM、AAC、MP3等)
- 低延迟编解码优化

### 3. 音频预处理
- 回声消除 (AEC - Acoustic Echo Cancellation)
- 噪声抑制 (ANS - Acoustic Noise Suppression)
- 自动增益控制 (AGC - Automatic Gain Control)
- 语音活动检测 (VAD - Voice Activity Detection)

### 4. 弱网对抗机制
- 自适应抖动缓冲 (Adaptive Jitter Buffer)
- 前向纠错 (FEC - Forward Error Correction)
- 丢包隐藏 (PLC - Packet Loss Concealment)
- 选择性重传 (NACK - Negative Acknowledgment)
- 动态码率调整 (Bandwidth Adaptation)

## 技术架构

### 技术栈选型
```
- Kurento Media Server (媒体处理引擎)
- Kurento Java Client (服务端控制)
- Spring Boot (应用框架)
- Redis (媒体会话状态)
- FFmpeg (音频处理工具链)
- GStreamer (底层媒体管道)
```

### 架构设计
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Client WebRTC  │◄──►│   Media Service  │◄──►│   AI Services   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │
                                ▼
                        ┌──────────────────┐
                        │ Kurento Media    │
                        │ Server Cluster   │
                        │ ┌──────────────┐ │
                        │ │ WebRTC       │ │
                        │ │ Endpoint     │ │
                        │ └──────────────┘ │
                        │ ┌──────────────┐ │
                        │ │ Audio        │ │
                        │ │ Processing   │ │
                        │ └──────────────┘ │
                        └──────────────────┘
```

### 核心组件

#### 1. Media Pipeline Manager
```java
@Service
public class MediaPipelineManager {
    
    public MediaPipeline createPipeline(String sessionId) {
        MediaPipeline pipeline = kurentoClient.createMediaPipeline();
        
        // 创建WebRTC端点
        WebRtcEndpoint webRtcEndpoint = new WebRtcEndpoint.Builder(pipeline).build();
        
        // 创建音频处理元素
        AudioProcessor audioProcessor = new AudioProcessor.Builder(pipeline)
            .withEchoCancellation(true)
            .withNoiseReduction(true)
            .withAutomaticGainControl(true)
            .build();
            
        // 连接媒体管道
        webRtcEndpoint.connect(audioProcessor);
        
        return pipeline;
    }
}
```

#### 2. Audio Processing Pipeline
```java
@Component
public class AudioProcessingPipeline {
    
    public void setupAudioFilters(MediaPipeline pipeline, AudioConfig config) {
        // 回声消除滤波器
        EchoCancellationFilter echoFilter = new EchoCancellationFilter.Builder(pipeline)
            .withFilterLength(config.getEchoFilterLength())
            .withAdaptationRate(config.getEchoAdaptationRate())
            .build();
            
        // 噪声抑制滤波器  
        NoiseReductionFilter noiseFilter = new NoiseReductionFilter.Builder(pipeline)
            .withNoiseReductionLevel(config.getNoiseReductionLevel())
            .withSpectralSubtraction(true)
            .build();
            
        // 自动增益控制
        AutomaticGainControlFilter agcFilter = new AutomaticGainControlFilter.Builder(pipeline)
            .withTargetLevel(config.getTargetAudioLevel())
            .withMaxGain(config.getMaxAudioGain())
            .build();
    }
}
```

#### 3. Network Adaptation Manager
```java
@Service
public class NetworkAdaptationManager {
    
    public void adaptToNetworkCondition(String sessionId, NetworkStats stats) {
        if (stats.getPacketLoss() > 0.05) { // 丢包率 > 5%
            // 启用FEC
            enableForwardErrorCorrection(sessionId, stats.getPacketLoss());
            // 降低码率
            adjustBitrate(sessionId, stats.getBandwidth() * 0.8);
        }
        
        if (stats.getJitter() > 50) { // 抖动 > 50ms
            // 增大抖动缓冲
            adjustJitterBuffer(sessionId, stats.getJitter());
        }
    }
    
    private void enableForwardErrorCorrection(String sessionId, double lossRate) {
        FecConfig fecConfig = FecConfig.builder()
            .redPayloadType(96)
            .ulpfecPayloadType(97)
            .redRtxPayloadType(98)
            .build();
        kurentoClient.enableFec(sessionId, fecConfig);
    }
}
```

## 媒体处理流程

### 音频接收处理流程
```
Client Audio → WebRTC → Decrypt → Decode → Audio Processing → AI Service
                                   ↓
                                 OPUS     ↓
                                 Decoder  ↓
                                          ↓
                                     ┌────────┐
                                     │  AEC   │ (回声消除)
                                     │  ANS   │ (噪声抑制)  
                                     │  AGC   │ (自动增益)
                                     │  VAD   │ (语音检测)
                                     └────────┘
                                          ↓
                                     PCM Audio → ASR Service
```

### 音频发送处理流程  
```
TTS Service → PCM Audio → Audio Processing → Encode → Encrypt → WebRTC → Client
                              ↓               ↓        ↓
                         ┌────────┐      ┌──────┐  ┌─────┐
                         │Filters │      │OPUS  │  │SRTP │
                         │Volume  │      │Encode│  │     │
                         │EQ      │      │      │  │     │
                         └────────┘      └──────┘  └─────┘
```

## 弱网优化技术

### 1. 自适应抖动缓冲
```java
@Component
public class AdaptiveJitterBuffer {
    
    private int targetDelayMs = 120; // 目标延迟120ms
    private int maxDelayMs = 500;    // 最大延迟500ms
    
    public void adjustBufferSize(String sessionId, JitterStats stats) {
        int currentJitter = stats.getCurrentJitter();
        int packetLoss = stats.getPacketLossRate();
        
        // 基于网络状况动态调整
        if (currentJitter > 100 && packetLoss > 0.02) {
            // 高抖动高丢包：增大缓冲
            int newDelay = Math.min(targetDelayMs + currentJitter * 2, maxDelayMs);
            setJitterBufferDelay(sessionId, newDelay);
        } else if (currentJitter < 20 && packetLoss < 0.005) {
            // 低抖动低丢包：减小缓冲
            int newDelay = Math.max(targetDelayMs - 20, 60);
            setJitterBufferDelay(sessionId, newDelay);
        }
    }
}
```

### 2. 动态码率调整
```java
@Service
public class BitrateAdaptation {
    
    public void adaptBitrate(String sessionId, BandwidthEstimation estimation) {
        int availableBandwidth = estimation.getAvailableBandwidth();
        int currentBitrate = getCurrentBitrate(sessionId);
        
        // AIMD算法 (Additive Increase Multiplicative Decrease)
        if (estimation.isIncreasing()) {
            // 带宽充足：线性增加码率
            int newBitrate = Math.min(currentBitrate + 5000, 128000); // 最大128kbps
            setBitrate(sessionId, newBitrate);
        } else if (estimation.isDecreasing()) {
            // 带宽不足：快速降低码率
            int newBitrate = Math.max(currentBitrate * 0.8, 8000); // 最小8kbps
            setBitrate(sessionId, newBitrate);
        }
    }
}
```

### 3. 前向纠错实现
```java
@Component
public class ForwardErrorCorrection {
    
    public void configureFec(String sessionId, double lossRate) {
        if (lossRate > 0.10) {
            // 高丢包率：启用强FEC
            FecParameters params = FecParameters.builder()
                .redPayloadType(96)
                .ulpfecPayloadType(97) 
                .fecMaskType(FecMaskType.RANDOM)
                .protectionLevel(0.8) // 80%保护率
                .build();
        } else if (lossRate > 0.05) {
            // 中等丢包率：启用中等FEC
            FecParameters params = FecParameters.builder()
                .protectionLevel(0.5) // 50%保护率
                .build();
        }
        
        kurentoClient.configureFec(sessionId, params);
    }
}
```

## 音频质量优化

### 1. 回声消除配置
```java
public class EchoCancellationConfig {
    
    public void configureAEC(MediaPipeline pipeline, RoomAcoustics acoustics) {
        EchoCancellationFilter aecFilter = new EchoCancellationFilter.Builder(pipeline)
            .withFilterLength(acoustics.getReverbTime() * 48) // 根据混响时间调整
            .withAdaptationRate(0.5f) // 自适应速率
            .withNonlinearProcessing(true) // 非线性处理
            .withClockDriftCompensation(true) // 时钟漂移补偿
            .build();
            
        // 根据环境动态调整参数
        if (acoustics.isNoisy()) {
            aecFilter.setSuppressionLevel(SuppressionLevel.HIGH);
        }
    }
}
```

### 2. 噪声抑制算法
```java
@Component  
public class NoiseReduction {
    
    public void configureNoiseReduction(String sessionId, NoiseProfile profile) {
        NoiseReductionParams params = NoiseReductionParams.builder()
            .algorithm(NoiseReductionAlgorithm.SPECTRAL_SUBTRACTION)
            .noiseFloor(profile.getNoiseFloor()) 
            .aggressiveness(profile.getAggressiveness())
            .adaptiveMode(true) // 自适应模式
            .build();
            
        // 针对不同噪声类型优化
        switch (profile.getNoiseType()) {
            case STATIONARY -> params.setStationaryNoiseReduction(true);
            case TRANSIENT -> params.setTransientNoiseReduction(true); 
            case PERIODIC -> params.setPeriodicNoiseReduction(true);
        }
        
        mediaProcessor.setNoiseReduction(sessionId, params);
    }
}
```

## 性能监控

### 关键指标
```java
@Component
public class MediaMetrics {
    
    @EventListener
    public void onMediaStats(MediaStatsEvent event) {
        MediaStats stats = event.getStats();
        
        // 音频质量指标
        metricsRegistry.gauge("media.audio.jitter", stats.getJitterMs());
        metricsRegistry.gauge("media.audio.packet_loss", stats.getPacketLossRate());
        metricsRegistry.gauge("media.audio.rtt", stats.getRoundTripTimeMs());
        
        // 编码指标
        metricsRegistry.gauge("media.audio.bitrate", stats.getAudioBitrate());
        metricsRegistry.gauge("media.audio.frame_rate", stats.getAudioFrameRate());
        
        // 处理延迟
        metricsRegistry.timer("media.processing.latency", stats.getProcessingLatency());
    }
}
```

### 性能目标
- 端到端音频延迟: < 300ms (理想网络)
- 音频处理延迟: < 50ms
- 并发媒体流: 5000+ /节点
- 丢包恢复率: > 95% (15%丢包环境)

## 接口设计

### Kurento Client API
```java
public interface MediaServiceAPI {
    
    // 媒体管道管理
    CompletableFuture<String> createMediaPipeline(String sessionId);
    CompletableFuture<Void> releaseMediaPipeline(String sessionId);
    
    // WebRTC端点管理
    CompletableFuture<String> processOffer(String sessionId, String sdpOffer);
    CompletableFuture<Void> addIceCandidate(String sessionId, IceCandidate candidate);
    
    // 音频处理配置
    CompletableFuture<Void> configureAudioProcessing(String sessionId, AudioConfig config);
    CompletableFuture<Void> updateNetworkAdaptation(String sessionId, NetworkStats stats);
    
    // 状态查询
    CompletableFuture<MediaStats> getMediaStats(String sessionId);
    CompletableFuture<List<String>> getActiveSessions();
}
```

### REST管理API
```http
# 媒体会话管理
POST   /api/media/sessions
GET    /api/media/sessions/{sessionId}
DELETE /api/media/sessions/{sessionId}
PUT    /api/media/sessions/{sessionId}/config

# 统计信息  
GET    /api/media/sessions/{sessionId}/stats
GET    /api/media/stats/summary

# 健康检查
GET    /api/media/health
GET    /api/media/kurento/status
```

## 部署方案

### Kurento Media Server部署
```yaml
# kurento-docker-compose.yml
version: '3.8'
services:
  kurento:
    image: kurento/kurento-media-server:latest
    ports:
      - "8888:8888" # WebSocket端口
      - "5000-5100:5000-5100/udp" # RTP端口范围
    environment:
      - KMS_MIN_PORT=5000
      - KMS_MAX_PORT=5100
      - GST_DEBUG=3
    volumes:
      - ./kurento-logs:/var/log/kurento-media-server
    ulimits:
      core: -1
```

### 媒体服务部署
```yaml
apiVersion: apps/v1
kind: Deployment  
metadata:
  name: media-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: media-service
  template:
    spec:
      containers:
      - name: media-service
        image: ai-interaction/media-service:latest
        ports:
        - containerPort: 8082
        env:
        - name: KURENTO_WS_URI
          value: "ws://kurento:8888/kurento"
        resources:
          requests:
            memory: "1Gi"
            cpu: "1000m"
          limits:
            memory: "2Gi" 
            cpu: "2000m"
```

## 测试策略

### 音频质量测试
```java
@Test
public void audioQualityTest() {
    // PESQ音频质量评估
    // 不同码率下的音频质量对比
    // 噪声环境下的处理效果测试
}
```

### 弱网环境测试
```java
@Test
public void networkAdaptationTest() {
    // 模拟不同丢包率 (1%, 5%, 15%)
    // 模拟网络抖动 (10ms, 50ms, 100ms)
    // 验证自适应算法效果
}
```

### 性能压力测试
```java
@Test
public void concurrentMediaTest() {
    // 5000并发音频流测试
    // CPU/内存使用率监控
    // 延迟分布统计
}
```

---

*技术方案版本: v1.0*
*最后更新: 2024-12-XX*