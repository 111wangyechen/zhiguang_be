# 知光平台 - 知识获取与分享社区

## 项目简介

知光平台是一个知识社区 APP，支持发布知识内容、点赞/收藏、关注互动、首页 Feed 流展示等功能。项目采用前后端分离架构，针对高并发和高可用场景进行了深度优化设计。

- **后端仓库**：https://github.com/G-Pegasus/zhiguang_be
- **前端仓库**：https://github.com/G-Pegasus/zhiguang_fe

## 技术栈

### 后端技术
| 类别 | 技术 |
|------|------|
| 核心框架 | Java 21 + Spring Boot 3.2.4 |
| 安全认证 | Spring Security + JWT (RS256) |
| 数据访问 | MyBatis 3.0.3 + MySQL 8.0 |
| 缓存系统 | Redis + Redisson + Caffeine |
| 消息队列 | Kafka |
| 搜索引擎 | Elasticsearch 9.2 |
| 对象存储 | 阿里云 OSS |
| 数据同步 | Canal (MySQL Binlog) |
| AI 能力 | Spring AI + DeepSeek |

### 前端技术
React + Vite

## 项目结构

```
src/main/java/com/tongji/
├── auth/                 # 认证模块
│   ├── api/              # 认证接口 Controller & DTO
│   ├── audit/            # 登录日志审计
│   ├── config/           # 安全配置
│   ├── service/          # 认证服务
│   ├── token/            # JWT 令牌管理
│   └── verification/     # 验证码服务
├── cache/                # 缓存模块
│   ├── config/           # 缓存配置
│   └── hotkey/           # 热点 Key 探测
├── common/               # 公共模块
│   ├── exception/        # 异常处理
│   └── web/              # 全局异常处理器
├── config/               # 全局配置
├── counter/              # 计数模块
│   ├── api/              # 计数接口
│   ├── event/            # 计数事件消费
│   ├── schema/           # SDS 计数结构
│   └── service/          # 计数服务
├── knowpost/             # 知文模块
│   ├── api/              # 知文接口
│   ├── id/               # 雪花 ID 生成
│   ├── listener/         # 事件监听
│   ├── mapper/           # 数据访问
│   ├── model/            # 数据模型
│   └── service/          # 业务服务
├── llm/                  # AI 模块
│   ├── rag/              # RAG 知识问答
│   └── service/          # AI 服务
├── profile/              # 用户资料模块
├── relation/             # 用户关系模块
│   ├── api/              # 关系接口
│   ├── event/            # 关系事件
│   ├── mapper/           # 数据访问
│   ├── outbox/           # Outbox 模式
│   └── processor/        # 事件处理器
├── search/               # 搜索模块
├── storage/              # 存储模块 (OSS)
└── user/                 # 用户模块
```

## 核心功能模块

### 1. 认证系统
基于 Spring Security 的 JWT 双令牌认证系统：
- **RS256 签名**：使用 RSA 非对称加密，私钥签名、公钥验签
- **双令牌机制**：Access Token (15分钟) + Refresh Token (7天)
- **Redis 白名单**：刷新令牌存储于 Redis，支持即时撤销
- **验证码服务**：支持手机/邮箱验证码，Redis 存储并限制尝试次数

### 2. 计数系统
采用「位图为事实、SDS 为汇总、事件为桥梁」的架构：
- **位图幂等**：Lua 原子切换，状态变更才产出事件
- **事件聚合**：Kafka 消费 → 聚合桶 (Hash) → 定时刷写到 SDS
- **紧凑存储**：SDS 固定 20 字节存储 5 个计数指标
- **异常重建**：SDS 缺失时基于位图 BITCOUNT 真实重建

### 3. 发布系统
渐进式发布流程，支持图文内容：
- **预签名直传**：后端生成 OSS 预签名 URL，前端直传
- **内容校验**：ETag + SHA-256 双重校验
- **AI 摘要**：接入 DeepSeek 自动生成文章摘要

### 4. 用户关系系统
一主多从 + 事件驱动模型：
- **Outbox 模式**：关注事件写入 Outbox 表，Canal 订阅 Binlog 发布到 Kafka
- **伪从同步**：粉丝表、计数、缓存作为关注表的伪从，异步更新
- **令牌桶限流**：防止短时间恶意关注

### 5. Feed 流
三级缓存架构：
- **L1 本地缓存**：Caffeine 高性能 LRU 缓存
- **L2 页面缓存**：Redis 缓存整页数据
- **L3 片段缓存**：Redis 缓存数据片段
- **热点探测**：自定义 HotKey 探测，按热度延长缓存时长
- **防雪崩**：随机抖动 TTL + 单飞锁 (single-flight)

### 6. 搜索系统
基于 Elasticsearch 构建：
- **全文检索**：支持关键词搜索、标签过滤
- **游标分页**：search_after 保证深分页稳定性
- **智能排序**：function_score 融合 BM25 与业务权重
- **前缀联想**：completion suggester 实现搜索建议

### 7. AI 问答系统 (RAG)
知文智能问答：
- **向量检索**：Spring AI + Elasticsearch 向量存储
- **流式生成**：SSE 推送，低延迟响应
- **智能分块**：约 800 字切片，重叠 ~100 字

## 数据库设计

主要数据表：
- `users` - 用户表
- `login_logs` - 登录日志
- `know_posts` - 知文主表
- `following` - 关注关系表
- `follower` - 粉丝关系表
- `outbox` - 事件发件箱

详细 Schema 请参阅 [db/schema.sql](db/schema.sql)

## API 接口

| 模块 | 路径前缀 | 说明 |
|------|----------|------|
| 认证 | `/api/auth` | 注册、登录、令牌刷新、密码重置 |
| 用户资料 | `/api/v1/profile` | 个人信息管理、头像上传 |
| 知文 | `/api/v1/knowposts` | 内容发布、Feed 流、详情、RAG 问答 |
| 用户关系 | `/api/v1/relation` | 关注/取关、关注列表、粉丝列表 |
| 计数 | `/api/v1/counter` | 点赞/收藏计数查询 |
| 存储 | `/api/v1/storage` | OSS 预签名上传 |

详细接口文档请参阅 [docs/](docs/) 目录。

## 快速开始

### 环境要求
- JDK 21+
- Maven 3.8+
- MySQL 8.0+
- Redis 7.0+
- Kafka 3.0+
- Elasticsearch 8.0+

### 配置文件
在 `src/main/resources/` 目录下创建 `application.yml`，配置数据库、Redis、Kafka、Elasticsearch、OSS 等连接信息。

### 运行项目
```bash
# Windows PowerShell
mvn clean install -DskipTests
mvn spring-boot:run

# Linux/Mac
./mvnw clean install -DskipTests
./mvnw spring-boot:run
```

## 项目亮点

1. **高并发设计**：位图计数、事件聚合、三级缓存等机制应对高并发场景
2. **最终一致性**：Outbox 模式 + Kafka 确保数据最终一致
3. **容灾能力**：位图事实重建、Kafka 消息回放、分布式锁保护
4. **可观测性**：Actuator 健康检查、刷写失败告警、聚合桶监控
5. **AI 赋能**：智能摘要生成、RAG 知识问答

## 许可证

本项目仅供学习交流使用。
