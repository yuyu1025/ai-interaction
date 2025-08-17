# Signaling Service - WebRTC信令服务

## 组件职责

负责WebRTC信令协商、会话建立、状态同步等实时通信控制功能，是客户端与媒体服务间的桥梁。

## 核心功能

### 1. WebRTC信令协商
- SDP (Session Description Protocol) 协商与交换
- ICE (Interactive Connectivity Establishment) 候选收集与选择
- DTLS-SRTP密钥协商
- 媒体能力协商 (音频编码、采样率等)

### 2. WebSocket连接管理
- 基于WebSocket的全双工通信
- 连接保活与断线重连
- 消息可靠传输与确认
- 连接状态监控与告警

### 3. 协议优化
- Protocol Buffers序列化 (比JSON减少30-50%传输)
- 消息压缩与批量传输
- 优先级队列 (信令消息优先级调度)
- 自适应心跳间隔

### 4. 状态管理
- WebRTC连接状态跟踪
- 媒体流状态同步
- 会话元数据管理
- 异常状态检测与恢复

## 技术架构

### 技术栈选型
```
- Spring Boot + WebFlux (响应式编程)
- Spring WebSocket (信令通道)
- Protocol Buffers (消息序列化)
- Redis Cluster (状态存储)
- Kurento Java Client (媒体服务交互)
```

### 架构设计
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Client SDK    │◄──►│ Signaling Service│◄──►│  Media Service  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │
                                ▼
                        ┌──────────────┐
                        │Redis Cluster │
                        │- Session     │
                        │- Connection  │ 
                        │- Media State │
                        └──────────────┘
```

### 核心组件

#### 1. WebSocket Handler
```java
@Component
public class SignalingWebSocketHandler extends TextWebSocketHandler {
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        // 连接建立后的初始化逻辑
        sessionManager.registerSession(session);
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 信令消息处理路由
        SignalingMessage msg = protobufDecoder.decode(message.getPayload());
        messageRouter.route(session, msg);
    }
}
```

#### 2. SDP Negotiator
```java
@Service
public class SdpNegotiator {
    
    public SdpAnswer processOffer(String sessionId, SdpOffer offer) {
        // 1. 验证SDP格式与媒体能力
        // 2. 与Kurento媒体服务协商
        // 3. 生成SDP Answer
        // 4. 更新会话状态
    }
    
    public void addIceCandidate(String sessionId, IceCandidate candidate) {
        // ICE候选处理与选择
    }
}
```

#### 3. Message Router
```java
@Component
public class SignalingMessageRouter {
    
    public void route(WebSocketSession session, SignalingMessage message) {
        switch (message.getType()) {
            case SDP_OFFER -> sdpNegotiator.processOffer(session, message);
            case ICE_CANDIDATE -> sdpNegotiator.addIceCandidate(session, message);
            case MEDIA_STATE -> mediaStateHandler.updateState(session, message);
            // ... 其他消息类型
        }
    }
}
```

## 协议设计

### Protocol Buffers定义
```protobuf
syntax = "proto3";

message SignalingMessage {
    string session_id = 1;
    MessageType type = 2;
    string payload = 3;
    int64 timestamp = 4;
    string sequence_id = 5;
}

enum MessageType {
    SDP_OFFER = 0;
    SDP_ANSWER = 1;
    ICE_CANDIDATE = 2;
    MEDIA_STATE = 3;
    SESSION_CONTROL = 4;
}

message SdpOffer {
    string sdp = 1;
    MediaConstraints constraints = 2;
}

message IceCandidate {
    string candidate = 1;
    string sdp_mid = 2;
    int32 sdp_mline_index = 3;
}
```

### 信令交互流程
```
Client                Signaling Service           Media Service
  │                         │                         │
  ├─── WebSocket Connect ──►│                         │
  │◄── Connection OK ───────┤                         │
  │                         │                         │
  ├─── SDP Offer ──────────►│                         │
  │                         ├─── Create Endpoint ────►│
  │                         │◄── Endpoint Created ────┤
  │                         ├─── Process SDP ────────►│
  │                         │◄── SDP Answer ──────────┤
  │◄── SDP Answer ──────────┤                         │
  │                         │                         │
  ├─── ICE Candidates ─────►│                         │
  │                         ├─── Add ICE Candidates ►│
  │◄── ICE Candidates ──────┤◄── ICE Candidates ──────┤
  │                         │                         │
  ├─── Media Connected ────►│                         │
  │                         ├─── Start Media ───────►│
```

## 接口设计

### WebSocket API
```
# 连接建立
ws://host:port/signaling?sessionId={id}&token={jwt}

# 消息格式 (Protocol Buffers编码)
{
  "session_id": "uuid",
  "type": "SDP_OFFER",
  "payload": "base64_encoded_protobuf",
  "timestamp": 1640995200000,
  "sequence_id": "seq_001"
}
```

### REST管理API
```http
# 会话管理
GET    /api/sessions
GET    /api/sessions/{sessionId}
DELETE /api/sessions/{sessionId}

# 连接状态
GET    /api/connections
GET    /api/connections/{connectionId}/stats

# 系统状态
GET    /api/health
GET    /api/metrics
```

## 性能指标

### 目标性能
- 信令延迟: P99 < 100ms
- 并发连接: 10000+ WebSocket连接/节点
- 消息吞吐: 50000+ 消息/秒
- 连接建立成功率: > 99.5%

### 监控指标
```
- signaling_connections_total (连接总数)
- signaling_message_duration (消息处理延迟)
- signaling_negotiation_success_rate (协商成功率)
- signaling_websocket_errors (WebSocket错误数)
```

## 弱网优化

### 消息可靠性
```java
@Service
public class ReliableMessageService {
    
    // 消息确认机制
    public void sendWithAck(String sessionId, SignalingMessage message) {
        message.setSequenceId(generateSequenceId());
        pendingMessages.put(message.getSequenceId(), message);
        webSocketSession.send(message);
        
        // 超时重传
        scheduleRetransmission(message, RETRANSMIT_TIMEOUT);
    }
    
    // 重复消息检测
    public boolean isDuplicate(SignalingMessage message) {
        return processedMessages.contains(message.getSequenceId());
    }
}
```

### 自适应心跳
```java
@Service
public class AdaptiveHeartbeat {
    
    public void adjustHeartbeatInterval(String sessionId, NetworkQuality quality) {
        switch (quality) {
            case EXCELLENT -> setInterval(sessionId, 30000); // 30s
            case GOOD -> setInterval(sessionId, 15000);      // 15s
            case POOR -> setInterval(sessionId, 5000);       // 5s
        }
    }
}
```

## 安全机制

### 身份认证
- JWT Token验证
- WebSocket握手时认证
- 会话绑定验证
- 权限范围检查

### 传输安全
- WSS (WebSocket Secure) 强制加密
- 消息完整性校验
- 防重放攻击 (时间戳+序列号)
- 连接频率限制

### 数据保护
```java
@Component
public class SecurityFilter {
    
    public boolean validateMessage(SignalingMessage message) {
        // 1. 时间戳验证 (防重放)
        // 2. 消息大小限制
        // 3. 敏感字段过滤
        // 4. 格式验证
    }
}
```

## 状态管理

### 会话状态
```java
public class SessionState {
    private String sessionId;
    private ConnectionState connectionState;
    private MediaState mediaState;
    private Map<String, Object> attributes;
    private long lastActivity;
}

public enum ConnectionState {
    CONNECTING,
    CONNECTED, 
    NEGOTIATING,
    ESTABLISHED,
    DISCONNECTED,
    FAILED
}
```

### 状态持久化
- Redis集群存储会话状态
- 定期状态快照
- 故障恢复时状态重建
- 状态变更事件通知

## 部署方案

### 容器化配置
```yaml
# docker-compose.yml
version: '3.8'
services:
  signaling-service:
    image: ai-interaction/signaling-service:latest
    ports:
      - "8081:8081"
    environment:
      - REDIS_CLUSTER_NODES=redis1:6379,redis2:6379,redis3:6379
      - KURENTO_WS_URI=ws://kurento:8888/kurento
    depends_on:
      - redis-cluster
      - kurento
```

### 负载均衡
- 基于Session Affinity的负载均衡
- WebSocket连接状态共享
- 故障节点自动剔除
- 弹性伸缩支持

## 测试策略

### 单元测试
- 信令协商逻辑测试
- 消息序列化/反序列化测试
- 状态机转换测试

### 集成测试
- WebRTC端到端连接测试
- 与Kurento媒体服务集成测试
- Redis状态同步测试

### 性能测试
```java
@Test
public void concurrentConnectionTest() {
    // 模拟10000并发WebSocket连接
    // 验证连接建立成功率
    // 测试消息吞吐性能
}
```

### 网络测试
- 弱网环境下的信令可靠性
- 断网重连恢复测试
- 高延迟网络下的协商成功率

---

*技术方案版本: v1.0*
*最后更新: 2024-12-XX*