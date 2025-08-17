# AI 实时互动服务端框架 - 开发计划与里程碑

## 项目概述

基于 WebRTC 的一对一 AI 实时音频互动服务端框架，采用 Java + Spring 生态实现，支持高并发、低延迟、弱网对抗的音频交互能力。

## 单体架构设计

```
ai-interaction/ (单体Spring Boot应用)
├── src/main/java/com/ai/interaction/
│   ├── gateway/          # API网关与路由模块
│   ├── signaling/        # WebRTC信令模块  
│   ├── media/           # 媒体处理模块
│   ├── orchestrator/    # AI服务编排模块
│   ├── session/         # 会话管理模块
│   ├── monitor/         # 简化监控模块
│   ├── config/          # 配置管理模块
│   └── common/          # 公共组件
├── src/main/resources/
│   ├── application.yml  # 主配置
│   ├── mapper/         # MyBatis-Plus映射文件
│   └── static/         # 静态资源
└── docker/             # 容器化部署
    ├── Dockerfile
    └── docker-compose.yml
```

## 开发里程碑

## 技术栈选型

### 核心框架
- **Spring Boot 3.x** - 应用主框架
- **Spring WebFlux** - 响应式Web框架
- **Spring WebSocket** - WebSocket支持
- **MyBatis-Plus** - 数据访问层 (支持软删除、代码生成)

### 媒体处理
- **Kurento Media Server** - WebRTC媒体服务器
- **Kurento Java Client** - Java客户端SDK

### 数据存储
- **PostgreSQL** - 主数据库
- **Redis** - 缓存和会话存储

### 基础组件
- **Netty** - 网络通信
- **Jackson** - JSON序列化
- **Caffeine** - 本地缓存
- **Spring Boot Actuator** - 健康检查和指标

### 部署运维
- **Docker** - 容器化
- **Docker Compose** - 本地开发环境
- **Prometheus + Grafana** - 可选监控 (开源版)

### 里程碑 1: 核心基础设施 (3-4周)
**目标**: 实现基础的一对一音频互通能力

- [ ] **Week 1-2: 项目脚手架与基础组件**
  - [x] 项目架构设计与技术选型
  - [ ] Spring Boot单体项目搭建
  - [ ] MyBatis-Plus + PostgreSQL集成
  - [ ] Redis集成与会话管理
  - [ ] Docker开发环境搭建

- [ ] **Week 3-4: WebRTC音频通话**
  - [ ] WebSocket信令模块
  - [ ] Kurento媒体处理集成
  - [ ] WebRTC SDP/ICE协商
  - [ ] 基础音频通话功能测试

**交付物**: 
- 可运行的一对一音频通话Demo
- 单体应用 + 数据库 + Redis的简单部署
- 基础API文档

### 里程碑 2: AI服务集成 (2-3周)  
**目标**: 完成ASR→LLM→TTS基础AI流水线

- [ ] **Week 5-6: AI编排模块**
  - [ ] AI服务抽象接口设计
  - [ ] ASR/LLM/TTS适配器实现
  - [ ] 服务编排流程 (同步调用)
  - [ ] 基础AI交互测试

**交付物**:
- 完整的音频→文本→音频AI交互
- AI服务适配器文档

### 里程碑 3: 弱网优化与监控 (2-3周)
**目标**: 实现生产级稳定性

- [ ] **Week 7-8: 弱网对抗**
  - [ ] 自适应抖动缓冲
  - [ ] 动态码率调整
  - [ ] 音频质量优化
  - [ ] 弱网测试工具

- [ ] **Week 9: 基础监控**
  - [ ] Spring Boot Actuator指标
  - [ ] 应用日志优化
  - [ ] 性能监控面板

**交付物**:
- 弱网环境稳定运行
- 基础监控面板

### 里程碑 4: 生产就绪 (2-3周)
**目标**: 生产级部署能力

- [ ] **Week 10-11: 部署优化**
  - [ ] Docker生产配置
  - [ ] 数据库迁移脚本
  - [ ] 配置外部化
  - [ ] 安全加固

- [ ] **Week 12: 文档与示例**
  - [ ] API文档完善
  - [ ] 部署指南
  - [ ] 示例客户端
  - [ ] 性能测试报告

**交付物**:
- 生产就绪的单体应用
- 完整部署文档
- 示例代码

## 简化后的架构优势

### 部署简化
- **单一部署单元**: 一个JAR包 + 数据库 + Redis
- **开发友好**: 本地一键启动，无需复杂的服务编排
- **运维简单**: 传统的应用部署，降低运维门槛

### 成本优势
- **基础设施成本**: 最小2核4G服务器即可运行
- **开发成本**: 降低学习曲线，加快开发速度
- **监控成本**: 使用开源方案，避免付费服务

### 技术债务管理

- [ ] **性能优化**: JVM调优、音频处理优化
- [ ] **并发模型**: WebFlux响应式编程优化
- [ ] **缓存策略**: 多级缓存设计
- [ ] **数据库优化**: 索引优化、查询优化

## 风险与应对

### 技术风险
- **单点故障**: 通过容器化 + 负载均衡解决
- **性能瓶颈**: 模块化设计便于后续拆分
- **WebRTC复杂性**: 采用Kurento降低风险

### 扩展策略
- **垂直扩展**: 增加服务器配置
- **水平扩展**: 负载均衡 + 会话粘性
- **模块拆分**: 当性能需要时可拆分为微服务

## 资源估算

### 最小开发团队
- **全栈工程师**: 1-2人 (核心功能开发)  
- **音视频工程师**: 1人 (WebRTC集成，可兼职)

### 基础设施需求
- **开发环境**: 本地Docker Compose
- **测试环境**: 单台云服务器 (2核4G)
- **生产环境**: 2-3台服务器 + 负载均衡

## 质量保证

### 测试策略
- **单元测试**: 覆盖率 ≥ 80%
- **集成测试**: 关键业务流程端到端测试
- **性能测试**: 并发连接、延迟、弱网测试
- **安全测试**: 渗透测试与安全扫描

### 代码质量
- **代码规范**: Checkstyle + SpotBugs静态检查
- **代码审查**: 必须经过同行评审
- **文档同步**: API文档与代码同步更新

---

## 简化部署方案

### 本地开发环境

```bash
# 1. 克隆项目
git clone https://github.com/ai-interaction/ai-interaction.git
cd ai-interaction

# 2. 启动依赖服务
docker-compose up -d postgres redis kurento

# 3. 运行应用
./mvnw spring-boot:run
```

### Docker Compose 一键部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/ai_interaction
      - SPRING_REDIS_HOST=redis
      - KURENTO_WS_URI=ws://kurento:8888/kurento
    depends_on:
      - postgres
      - redis  
      - kurento
    
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=ai_interaction
      - POSTGRES_USER=ai_user
      - POSTGRES_PASSWORD=ai_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
      
  kurento:
    image: kurento/kurento-media-server:latest
    ports:
      - "8888:8888"
      - "5000-5100:5000-5100/udp"
    environment:
      - KMS_MIN_PORT=5000
      - KMS_MAX_PORT=5100

volumes:
  postgres_data:
  redis_data:
```

### 生产环境部署

```bash
# 1. 构建生产镜像
docker build -t ai-interaction:latest .

# 2. 启动完整环境
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 3. 健康检查
curl http://localhost:8080/actuator/health
```

### 配置外部化

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/ai_interaction}
    username: ${DATABASE_USER:ai_user}
    password: ${DATABASE_PASSWORD:ai_password}
    
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}
    
kurento:
  ws-uri: ${KURENTO_WS_URI:ws://localhost:8888/kurento}

ai:
  services:
    openai:
      api-key: ${OPENAI_API_KEY}
    baidu:
      app-id: ${BAIDU_APP_ID}
      app-secret: ${BAIDU_APP_SECRET}
```

### 扩展策略

```bash
# 负载均衡部署
# 1. 启动多个应用实例
docker-compose up --scale app=3

# 2. 配置Nginx负载均衡
# nginx.conf
upstream ai_backend {
    server app:8080;
    # Docker Compose 自动负载均衡到多个app实例
}

server {
    listen 80;
    location / {
        proxy_pass http://ai_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 监控部署 (可选)

```bash
# 启用Prometheus + Grafana监控
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
```

## 项目结构

### 单体应用目录结构

```
ai-interaction/
├── src/main/java/com/ai/interaction/
│   ├── AiInteractionApplication.java     # 主启动类
│   ├── config/                          # 配置类
│   │   ├── WebConfig.java
│   │   ├── RedisConfig.java
│   │   ├── MyBatisPlusConfig.java
│   │   └── WebSocketConfig.java
│   ├── gateway/                         # 网关模块
│   │   ├── filter/
│   │   ├── security/
│   │   └── controller/
│   ├── signaling/                       # 信令模块
│   │   ├── handler/
│   │   ├── message/
│   │   └── service/
│   ├── media/                          # 媒体处理模块
│   │   ├── kurento/
│   │   ├── audio/
│   │   └── network/
│   ├── orchestrator/                    # AI编排模块
│   │   ├── adapter/
│   │   ├── service/
│   │   └── workflow/
│   ├── session/                        # 会话管理模块
│   │   ├── entity/
│   │   ├── mapper/
│   │   ├── service/
│   │   └── controller/
│   ├── monitor/                        # 监控模块
│   │   ├── health/
│   │   ├── metrics/
│   │   └── alert/
│   └── common/                         # 公共模块
│       ├── constant/
│       ├── exception/
│       ├── util/
│       └── vo/
├── src/main/resources/
│   ├── application.yml                  # 主配置
│   ├── application-dev.yml              # 开发环境
│   ├── application-prod.yml             # 生产环境
│   ├── mapper/                         # MyBatis映射
│   ├── db/migration/                   # 数据库迁移
│   └── static/                         # 静态资源
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── docker-compose.monitoring.yml
├── docs/                               # 文档
├── scripts/                            # 部署脚本
└── pom.xml                            # Maven配置
```

*本计划将根据实际开发进度和反馈进行动态调整*