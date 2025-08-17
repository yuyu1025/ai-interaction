# Config Service - 配置管理服务

## 组件职责

提供集中化配置管理、动态配置更新、环境隔离、配置版本管理等功能。

## 核心功能

### 1. 集中化配置
- 多环境配置管理
- 配置继承与覆盖
- 配置加密与敏感信息保护
- 配置校验与格式检查

### 2. 动态配置更新
- 无重启配置更新
- 配置变更通知
- 灰度发布支持
- 配置回滚机制

### 3. 服务发现
- 服务注册与发现
- 健康检查集成
- 负载均衡配置
- 故障转移支持

## 技术架构

### 技术栈
- Spring Cloud Config Server
- Consul (服务发现 + KV存储)
- Git Repository (配置版本管理)
- Vault (敏感信息加密)

### 架构设计
```
┌──────────────────────────────────┐
│        应用服务集群         │
│ ┌──────────────────────────────┐ │
│ │ Gateway + Signaling + Media │ │
│ │ + AI + Session + Edge + Mon │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
                      │
                      ▼
    ┌──────────────────────────────────┐
    │         配置管理集群        │
    │ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
    │ │Config   │ │  Consul  │ │  Vault   │ │
    │ │Server   │ │ Cluster │ │ Cluster │ │
    │ └──────────┘ └──────────┘ └──────────┘ │
    └──────────────────────────────────┘
                      │
                      ▼
             ┌──────────────────┐
             │  Git Repository  │
             │ (配置源码管理) │
             └──────────────────┘
```

## 配置级别

### 1. 全局配置
```yaml
# application-global.yml
spring:
  profiles:
    active: production
    
monitoring:
  enabled: true
  sample-rate: 0.1
  
security:
  jwt:
    expiry: 3600
```

### 2. 服务级配置
```yaml
# ai-orchestrator-production.yml
ai:
  services:
    asr:
      provider: baidu
      timeout: 3000ms
      retry: 2
    llm:
      provider: openai
      model: gpt-3.5-turbo
      max-tokens: 200
    tts:
      provider: xunfei
      voice: xiaoyan
      speed: 5
```

### 3. 环境特定配置
```yaml
# application-production.yml
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/ai_interaction
    
redis:
  cluster:
    nodes: 
      - prod-redis1:6379
      - prod-redis2:6379
      - prod-redis3:6379
```

## 动态配置更新

```java
@Component
@RefreshScope
public class DynamicConfiguration {
    
    @Value("${ai.orchestration.timeout:5000}")
    private int orchestrationTimeout;
    
    @Value("${ai.cache.enabled:true}")
    private boolean cacheEnabled;
    
    @EventListener
    public void onRefreshEvent(RefreshRemoteApplicationEvent event) {
        log.info("Configuration refreshed for service: {}", event.getDestinationService());
        // 重新初始化组件
        reinitializeComponents();
    }
}

@RestController
public class ConfigRefreshController {
    
    @PostMapping("/actuator/refresh")
    public ResponseEntity<String> refreshConfig() {
        // 触发配置刷新
        applicationEventPublisher.publishEvent(new RefreshEvent(this, null, "Refresh config"));
        return ResponseEntity.ok("Config refreshed");
    }
}
```

## 敏感信息管理

```java
@Component
public class VaultPropertySource {
    
    @Bean
    @Primary
    public PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        
        // 加载 Vault 中的敏感配置
        MutablePropertySources propertySources = new MutablePropertySources();
        propertySources.addFirst(new VaultPropertySource("vault", vaultTemplate));
        
        configurer.setPropertySources(propertySources);
        return configurer;
    }
}

// 使用加密配置
@Component
public class AIServiceConfiguration {
    
    @Value("${vault:secret/ai-services#openai.api.key}")
    private String openaiApiKey;
    
    @Value("${vault:secret/ai-services#baidu.app.secret}")
    private String baiduAppSecret;
}
```

## 环境管理

### 配置继承策略
```
│── application.yml (基础配置)
│    │
│    └── application-{env}.yml (环境特定)
│         │
│         └── {service}-{env}.yml (服务特定)
│              │
│              └── 本地配置文件 (最高优先级)
```

### 灰度发布
```java
@Component
public class FeatureToggleService {
    
    @Value("${feature.ai.interruption.enabled:false}")
    private boolean interruptionEnabled;
    
    @Value("${feature.ai.cache.enabled:true}")
    private boolean cacheEnabled;
    
    public boolean isFeatureEnabled(String featureName, String userId) {
        // 根据用户ID或百分比判断功能是否开启
        if ("ai.interruption".equals(featureName)) {
            return interruptionEnabled && isUserInBetaGroup(userId);
        }
        return false;
    }
    
    private boolean isUserInBetaGroup(String userId) {
        // 简单的哈希算法实现灰度发布
        return userId.hashCode() % 100 < betaPercentage;
    }
}
```

## 性能目标

- 配置获取延迟: < 50ms
- 配置更新传播: < 10s
- 服务发现延迟: < 30s
- 配置可用性: 99.95%

## 接口设计

```http
# 配置管理
GET    /api/config/{service}/{profile}
POST   /api/config/{service}/{profile}/refresh
PUT    /api/config/{service}/{profile}

# 服务发现
GET    /api/discovery/services
GET    /api/discovery/services/{serviceName}/instances
POST   /api/discovery/services/{serviceName}/register
DELETE /api/discovery/services/{serviceName}/deregister

# 健康检查
GET    /api/health
GET    /api/health/{serviceName}
```

---

*技术方案版本: v1.0*