# Session Manager - 会话管理服务

## 组件职责

负责一对一AI交互会话的全生命周期管理，包括会话创建、状态维护、资源分配、故障恢复等核心功能。

## 核心功能

### 1. 会话生命周期管理
- 会话创建与初始化
- 状态跟踪与同步
- 会话终止与资源清理
- 超时管理与自动回收

### 2. 状态持久化
- 会话状态实时存储
- 断点续连支持
- 故障恢复机制
- 数据一致性保障

### 3. 资源分配与隔离
- 单用户资源隔离
- 动态资源分配
- 负载均衡调度
- 资源使用监控

### 4. 会话路由
- 基于会话ID的路由
- 节点亲和性管理
- 故障转移支持
- 负载感知调度

## 技术架构

### 技术栈选型
```
- Spring Boot + Data JPA (应用框架)
- Redis Cluster (状态存储)
- PostgreSQL (持久化存储)
- Apache Kafka (事件流)
- Consul (服务发现)
- Micrometer (监控指标)
```

### 架构设计
```
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│  Gateway API    │───►│   Session Manager    │───►│ Backend Services│
└─────────────────┘    └──────────────────────┘    └─────────────────┘
                                │                    ┌──────────────┐
                                │                    │ Media Service│
                                ▼                    │ AI Service   │
                        ┌──────────────────┐         │ Signaling    │
                        │  Session Store   │         └──────────────┘
                        │ ┌──────────────┐ │
                        │ │ Redis Cluster│ │ ◄─── Hot Data
                        │ │ (Hot State)  │ │
                        │ └──────────────┘ │
                        │ ┌──────────────┐ │
                        │ │ PostgreSQL   │ │ ◄─── Cold Data
                        │ │ (Persistent) │ │
                        │ └──────────────┘ │
                        └──────────────────┘
                                │
                                ▼
                        ┌──────────────────┐
                        │  Event Stream    │
                        │ ┌──────────────┐ │
                        │ │ Kafka Topics │ │
                        │ │ - Session    │ │
                        │ │ - State      │ │
                        │ │ - Events     │ │
                        │ └──────────────┘ │
                        └──────────────────┘
```

### 核心组件

#### 1. Session Repository
```java
@Repository
public class SessionRepository {
    
    private final RedisTemplate<String, SessionState> redisTemplate;
    private final SessionJpaRepository jpaRepository;
    
    public Mono<SessionState> findBySessionId(String sessionId) {
        // 优先从Redis缓存读取
        return redisTemplate.opsForValue()
            .get("session:" + sessionId)
            .cast(SessionState.class)
            .switchIfEmpty(
                // 缓存未命中，从数据库读取并缓存
                Mono.fromCallable(() -> jpaRepository.findBySessionId(sessionId))
                    .flatMap(sessionEntity -> {
                        SessionState state = convertToSessionState(sessionEntity);
                        return cacheSessionState(sessionId, state)
                            .thenReturn(state);
                    })
            );
    }
    
    public Mono<Void> saveSessionState(SessionState sessionState) {
        return Mono.zip(
            // 保存到Redis缓存
            redisTemplate.opsForValue()
                .set("session:" + sessionState.getSessionId(), sessionState, Duration.ofHours(24)),
            // 异步保存到数据库
            Mono.fromRunnable(() -> 
                jpaRepository.save(convertToSessionEntity(sessionState)))
        ).then();
    }
}
```

#### 2. Session Manager
```java
@Service
public class SessionManager {
    
    public Mono<SessionInfo> createSession(CreateSessionRequest request) {
        String sessionId = generateSessionId();
        
        return Mono.fromCallable(() -> {
            SessionState sessionState = SessionState.builder()
                .sessionId(sessionId)
                .userId(request.getUserId())
                .status(SessionStatus.INITIALIZING)
                .createdAt(Instant.now())
                .lastActivityAt(Instant.now())
                .configuration(request.getConfiguration())
                .build();
                
            return sessionState;
        })
        .flatMap(sessionState -> {
            // 分配资源
            return resourceAllocator.allocateResources(sessionState)
                .flatMap(allocation -> {
                    sessionState.setResourceAllocation(allocation);
                    sessionState.setStatus(SessionStatus.ACTIVE);
                    
                    // 保存会话状态
                    return sessionRepository.saveSessionState(sessionState)
                        .thenReturn(convertToSessionInfo(sessionState));
                });
        })
        .doOnSuccess(sessionInfo -> {
            // 发送会话创建事件
            eventPublisher.publishEvent(new SessionCreatedEvent(sessionInfo));
        });
    }
    
    public Mono<Void> terminateSession(String sessionId, TerminationReason reason) {
        return sessionRepository.findBySessionId(sessionId)
            .flatMap(sessionState -> {
                sessionState.setStatus(SessionStatus.TERMINATED);
                sessionState.setTerminatedAt(Instant.now());
                sessionState.setTerminationReason(reason);
                
                return sessionRepository.saveSessionState(sessionState)
                    .then(resourceAllocator.releaseResources(sessionState.getResourceAllocation()))
                    .doOnSuccess(unused -> {
                        eventPublisher.publishEvent(new SessionTerminatedEvent(sessionId, reason));
                    });
            });
    }
}
```

#### 3. Resource Allocator
```java
@Component
public class ResourceAllocator {
    
    public Mono<ResourceAllocation> allocateResources(SessionState sessionState) {
        return Mono.fromCallable(() -> {
            // 选择最优节点
            ServiceNode mediaNode = selectOptimalNode(ServiceType.MEDIA, sessionState);
            ServiceNode aiNode = selectOptimalNode(ServiceType.AI, sessionState);
            ServiceNode signalingNode = selectOptimalNode(ServiceType.SIGNALING, sessionState);
            
            return ResourceAllocation.builder()
                .sessionId(sessionState.getSessionId())
                .mediaServiceNode(mediaNode)
                .aiServiceNode(aiNode)
                .signalingServiceNode(signalingNode)
                .allocatedAt(Instant.now())
                .build();
        });
    }
    
    private ServiceNode selectOptimalNode(ServiceType serviceType, SessionState sessionState) {
        List<ServiceNode> availableNodes = serviceDiscovery.getHealthyNodes(serviceType);
        
        // 负载均衡策略：最少连接数
        return availableNodes.stream()
            .min(Comparator.comparing(ServiceNode::getCurrentConnections))
            .orElseThrow(() -> new NoAvailableNodeException(serviceType));
    }
}
```

## 会话状态模型

### 状态定义
```java
@Entity
@Table(name = "sessions")
public class SessionEntity {
    
    @Id
    private String sessionId;
    
    @Column(nullable = false)
    private String userId;
    
    @Enumerated(EnumType.STRING)
    private SessionStatus status;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    @Column(name = "last_activity_at") 
    private Instant lastActivityAt;
    
    @Column(name = "terminated_at")
    private Instant terminatedAt;
    
    @Enumerated(EnumType.STRING)
    private TerminationReason terminationReason;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private SessionConfiguration configuration;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private ResourceAllocation resourceAllocation;
}

public enum SessionStatus {
    INITIALIZING,    // 初始化中
    ACTIVE,          // 活跃状态
    IDLE,           // 空闲状态
    SUSPENDED,      // 暂停状态
    TERMINATED,     // 已终止
    FAILED          // 失败状态
}

public enum TerminationReason {
    USER_INITIATED,     // 用户主动终止
    TIMEOUT,           // 超时终止
    ERROR,             // 错误终止
    RESOURCE_LIMIT,    // 资源限制
    MAINTENANCE        // 维护终止
}
```

### 状态机
```java
@Component
public class SessionStateMachine {
    
    private final Map<StateTransition, SessionStatus> transitions = Map.of(
        new StateTransition(SessionStatus.INITIALIZING, SessionEvent.RESOURCES_ALLOCATED), SessionStatus.ACTIVE,
        new StateTransition(SessionStatus.ACTIVE, SessionEvent.USER_INACTIVE), SessionStatus.IDLE,
        new StateTransition(SessionStatus.IDLE, SessionEvent.USER_ACTIVITY), SessionStatus.ACTIVE,
        new StateTransition(SessionStatus.ACTIVE, SessionEvent.TERMINATION_REQUESTED), SessionStatus.TERMINATED,
        new StateTransition(SessionStatus.IDLE, SessionEvent.TIMEOUT), SessionStatus.TERMINATED,
        new StateTransition(SessionStatus.ACTIVE, SessionEvent.ERROR_OCCURRED), SessionStatus.FAILED
    );
    
    public SessionStatus transition(SessionStatus currentStatus, SessionEvent event) {
        StateTransition transition = new StateTransition(currentStatus, event);
        SessionStatus newStatus = transitions.get(transition);
        
        if (newStatus == null) {
            throw new IllegalStateTransitionException(currentStatus, event);
        }
        
        return newStatus;
    }
}
```

## 超时与清理机制

### 超时管理
```java
@Component
@Scheduled
public class SessionTimeoutManager {
    
    @Scheduled(fixedDelay = 60000) // 每分钟检查一次
    public void checkSessionTimeouts() {
        Instant cutoffTime = Instant.now().minus(Duration.ofMinutes(30)); // 30分钟超时
        
        sessionRepository.findIdleSessionsBefore(cutoffTime)
            .flatMap(sessionId -> {
                log.info("Session {} timed out due to inactivity", sessionId);
                return sessionManager.terminateSession(sessionId, TerminationReason.TIMEOUT);
            })
            .doOnError(error -> log.error("Failed to terminate timed out session", error))
            .subscribe();
    }
    
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点
    public void cleanupTerminatedSessions() {
        Instant cutoffTime = Instant.now().minus(Duration.ofDays(7)); // 保留7天
        
        sessionRepository.deleteTerminatedSessionsBefore(cutoffTime)
            .doOnSuccess(count -> log.info("Cleaned up {} terminated sessions", count))
            .subscribe();
    }
}
```

### 资源回收
```java
@Component
public class ResourceCleanupService {
    
    @EventListener
    public void onSessionTerminated(SessionTerminatedEvent event) {
        String sessionId = event.getSessionId();
        
        // 清理Redis缓存
        redisTemplate.delete("session:" + sessionId);
        redisTemplate.delete("context:" + sessionId);
        
        // 清理媒体资源
        mediaService.releaseMediaPipeline(sessionId)
            .doOnError(error -> log.warn("Failed to release media pipeline for session {}", sessionId, error))
            .subscribe();
            
        // 清理AI上下文
        aiOrchestrator.clearSessionContext(sessionId)
            .doOnError(error -> log.warn("Failed to clear AI context for session {}", sessionId, error))
            .subscribe();
    }
}
```

## 断点续连

### 状态恢复
```java
@Service
public class SessionRecoveryService {
    
    public Mono<SessionInfo> recoverSession(String sessionId, String userId) {
        return sessionRepository.findBySessionId(sessionId)
            .filter(session -> session.getUserId().equals(userId))
            .filter(session -> session.getStatus() == SessionStatus.SUSPENDED)
            .flatMap(sessionState -> {
                // 重新分配资源
                return resourceAllocator.allocateResources(sessionState)
                    .flatMap(newAllocation -> {
                        sessionState.setResourceAllocation(newAllocation);
                        sessionState.setStatus(SessionStatus.ACTIVE);
                        sessionState.setLastActivityAt(Instant.now());
                        
                        return sessionRepository.saveSessionState(sessionState)
                            .thenReturn(convertToSessionInfo(sessionState));
                    });
            })
            .doOnSuccess(sessionInfo -> {
                eventPublisher.publishEvent(new SessionRecoveredEvent(sessionInfo));
            });
    }
}
```

### 心跳机制
```java
@Component
public class SessionHeartbeatService {
    
    @EventListener
    public void onHeartbeat(SessionHeartbeatEvent event) {
        String sessionId = event.getSessionId();
        
        sessionRepository.updateLastActivity(sessionId, Instant.now())
            .doOnError(error -> log.warn("Failed to update session activity for {}", sessionId, error))
            .subscribe();
    }
    
    @Scheduled(fixedDelay = 30000) // 30秒检查一次
    public void checkMissedHeartbeats() {
        Instant cutoffTime = Instant.now().minus(Duration.ofMinutes(2)); // 2分钟无心跳
        
        sessionRepository.findSessionsWithoutHeartbeatSince(cutoffTime)
            .flatMap(sessionId -> {
                log.warn("Session {} missed heartbeat, suspending", sessionId);
                return suspendSession(sessionId);
            })
            .subscribe();
    }
    
    private Mono<Void> suspendSession(String sessionId) {
        return sessionRepository.findBySessionId(sessionId)
            .flatMap(sessionState -> {
                sessionState.setStatus(SessionStatus.SUSPENDED);
                return sessionRepository.saveSessionState(sessionState);
            })
            .then();
    }
}
```

## 监控与指标

### 会话指标
```java
@Component
public class SessionMetrics {
    
    private final AtomicLong activeSessions = new AtomicLong(0);
    private final AtomicLong totalSessions = new AtomicLong(0);
    
    @EventListener
    public void onSessionCreated(SessionCreatedEvent event) {
        activeSessions.incrementAndGet();
        totalSessions.incrementAndGet();
        
        meterRegistry.gauge("sessions.active", activeSessions);
        meterRegistry.counter("sessions.created").increment();
    }
    
    @EventListener
    public void onSessionTerminated(SessionTerminatedEvent event) {
        activeSessions.decrementAndGet();
        
        meterRegistry.gauge("sessions.active", activeSessions);
        meterRegistry.counter("sessions.terminated", 
            Tags.of("reason", event.getReason().name())).increment();
    }
    
    @Scheduled(fixedDelay = 60000)
    public void recordSessionDurationMetrics() {
        Instant oneHourAgo = Instant.now().minus(Duration.ofHours(1));
        
        sessionRepository.getAverageSessionDuration(oneHourAgo, Instant.now())
            .doOnSuccess(avgDuration -> {
                meterRegistry.gauge("sessions.average_duration_minutes", avgDuration.toMinutes());
            })
            .subscribe();
    }
}
```

### 性能目标
- 会话创建延迟: P99 < 200ms
- 会话查询延迟: P99 < 50ms  
- 并发活跃会话: 50000+ /集群
- 会话状态一致性: 99.99%
- 故障恢复时间: < 30s

## 接口设计

### REST API
```http
# 会话管理
POST   /api/sessions
GET    /api/sessions/{sessionId}
PUT    /api/sessions/{sessionId}
DELETE /api/sessions/{sessionId}

# 会话状态
GET    /api/sessions/{sessionId}/status
PUT    /api/sessions/{sessionId}/status
POST   /api/sessions/{sessionId}/heartbeat

# 会话恢复
POST   /api/sessions/{sessionId}/recover

# 批量操作
GET    /api/sessions?userId={userId}&status={status}
DELETE /api/sessions?status=terminated&before={timestamp}

# 监控接口
GET    /api/sessions/metrics
GET    /api/sessions/health
```

### 事件API
```java
public interface SessionEventListener {
    
    // 会话生命周期事件
    void onSessionCreated(SessionCreatedEvent event);
    void onSessionTerminated(SessionTerminatedEvent event);
    void onSessionRecovered(SessionRecoveredEvent event);
    
    // 状态变更事件
    void onSessionStatusChanged(SessionStatusChangedEvent event);
    
    // 心跳事件
    void onSessionHeartbeat(SessionHeartbeatEvent event);
    
    // 错误事件
    void onSessionError(SessionErrorEvent event);
}
```

## 数据库设计

### 表结构
```sql
-- 会话表
CREATE TABLE sessions (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    last_activity_at TIMESTAMP WITH TIME ZONE NOT NULL,
    terminated_at TIMESTAMP WITH TIME ZONE,
    termination_reason VARCHAR(32),
    configuration JSONB,
    resource_allocation JSONB,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_last_activity (last_activity_at)
);

-- 会话事件表
CREATE TABLE session_events (
    id BIGSERIAL PRIMARY KEY,
    session_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    event_data JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    INDEX idx_session_id (session_id),
    INDEX idx_event_type (event_type),
    INDEX idx_created_at (created_at)
);
```

## 部署方案

### 容器化部署
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: session-manager
spec:
  replicas: 3
  selector:
    matchLabels:
      app: session-manager
  template:
    spec:
      containers:
      - name: session-manager
        image: ai-interaction/session-manager:latest
        ports:
        - containerPort: 8084
        env:
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://postgres:5432/sessions"
        - name: SPRING_REDIS_CLUSTER_NODES
          value: "redis1:6379,redis2:6379,redis3:6379"
        - name: SPRING_KAFKA_BOOTSTRAP_SERVERS
          value: "kafka1:9092,kafka2:9092"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8084
          initialDelaySeconds: 30
          periodSeconds: 10
```

## 测试策略

### 单元测试
```java
@Test
public void testSessionLifecycle() {
    // 测试会话创建、状态变更、终止流程
    // 验证状态机转换逻辑
    // 测试超时和清理机制
}
```

### 集成测试
```java
@Test
public void testSessionRecovery() {
    // 测试断点续连功能
    // 验证故障恢复机制
    // 测试数据一致性
}
```

### 性能测试
```java
@Test
public void testConcurrentSessions() {
    // 大量并发会话创建测试
    // 会话查询性能测试
    // 内存使用和GC压力测试
}
```

---

*技术方案版本: v1.0*
*最后更新: 2024-12-XX*