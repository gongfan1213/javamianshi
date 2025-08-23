# Java 微服务中间件学习指南

## 📚 学习路径

本指南将带你从零基础开始，逐步掌握Java微服务中间件的核心概念和实践应用。

### 🎯 学习目标
- 理解微服务架构的基本概念和设计原则
- 掌握Spring Cloud生态系统的核心组件
- 学会使用各种微服务中间件
- 能够应对微服务相关的面试问题

### 📖 学习顺序
1. **[01.微服务架构基础.md](./01.微服务架构基础.md)** - 基本概念和设计原则
2. **[02.Spring Cloud概述.md](./02.Spring Cloud概述.md)** - Spring Cloud生态系统介绍
3. **[03.服务注册与发现.md](./03.服务注册与发现.md)** - Eureka、Consul、Nacos
4. **[04.配置中心.md](./04.配置中心.md)** - Spring Cloud Config、Nacos Config
5. **[05.服务网关.md](./05.服务网关.md)** - Spring Cloud Gateway、Zuul
6. **[06.负载均衡.md](./06.负载均衡.md)** - Ribbon、LoadBalancer
7. **[07.服务调用.md](./07.服务调用.md)** - OpenFeign、RestTemplate
8. **[08.熔断器.md](./08.熔断器.md)** - Hystrix、Sentinel、Resilience4j
9. **[09.分布式事务.md](./09.分布式事务.md)** - Seata、Saga模式
10. **[10.消息队列.md](./10.消息队列.md)** - RabbitMQ、Kafka、RocketMQ
11. **[11.链路追踪.md](./11.链路追踪.md)** - Sleuth、Zipkin、SkyWalking
12. **[12.监控告警.md](./12.监控告警.md)** - Actuator、Micrometer、Prometheus
13. **[13.面试题集锦.md](./13.面试题集锦.md)** - 常见面试题和答案
14. **[14.实战项目.md](./14.实战项目.md)** - 完整的微服务项目案例

### 🚀 快速开始
如果你已经有一定基础，可以直接跳转到：
- **[服务注册与发现](./03.服务注册与发现.md)** - 学习服务治理
- **[面试题集锦](./13.面试题集锦.md)** - 准备面试
- **[实战项目](./14.实战项目.md)** - 完整项目实践

### 💡 学习建议
- 每个文档都包含理论知识和实践代码
- 建议边学边练，动手实践
- 重点关注微服务架构的设计思想
- 理解各种中间件的作用和选择

### 📋 文档内容概览

#### 基础理论篇
- **微服务架构**：基本概念、设计原则、优缺点
- **Spring Cloud**：生态系统、版本选择、核心组件

#### 核心中间件篇
- **服务治理**：注册发现、配置管理、服务网关
- **服务通信**：负载均衡、服务调用、熔断降级
- **数据一致性**：分布式事务、消息队列
- **可观测性**：链路追踪、监控告警

#### 实践应用篇
- **项目实战**：完整微服务项目实现
- **面试准备**：常见问题和答案

## 🔧 技术栈选择

### 推荐技术栈
```
Spring Boot 2.7.x + Spring Cloud 2021.0.x + Spring Cloud Alibaba 2021.0.x
├── 注册中心：Nacos（推荐）
├── 配置中心：Nacos Config
├── 服务网关：Spring Cloud Gateway
├── 负载均衡：Spring Cloud LoadBalancer
├── 服务调用：OpenFeign
├── 熔断器：Resilience4j
└── 数据库：MySQL + Redis
```

### 版本兼容性
- Spring Boot 2.7.x → Spring Cloud 2021.0.x
- Spring Cloud 2021.0.x → Spring Cloud Alibaba 2021.0.x

## 💻 环境准备

### 必需软件
- **JDK 8+**：Java开发环境
- **Maven 3.6+**：项目构建工具
- **MySQL 8.0+**：关系型数据库
- **Redis 6.0+**：缓存数据库
- **Nacos 2.2+**：服务注册与配置中心

### 可选软件
- **RabbitMQ**：消息队列
- **Kafka**：高吞吐量消息队列
- **Prometheus**：监控系统
- **Grafana**：监控可视化

## 🚀 快速上手

### 1. 克隆项目
```bash
git clone <repository-url>
cd Java微服务中间件学习指南
```

### 2. 启动Nacos
```bash
# 下载并启动Nacos
wget https://github.com/alibaba/nacos/releases/download/2.2.3/nacos-server-2.2.3.tar.gz
tar -xzf nacos-server-2.2.3.tar.gz
cd nacos
bin/startup.sh -m standalone
```

### 3. 运行实战项目
```bash
# 按照实战项目文档的步骤
cd 14.实战项目
# 启动各个服务
```

## 📚 学习资源

### 官方文档
- [Spring Boot官方文档](https://spring.io/projects/spring-boot)
- [Spring Cloud官方文档](https://spring.io/projects/spring-cloud)
- [Spring Cloud Alibaba官方文档](https://github.com/alibaba/spring-cloud-alibaba)

### 推荐书籍
- 《Spring Cloud微服务实战》
- 《微服务设计》
- 《Spring Boot实战》

### 在线资源
- [Spring官方教程](https://spring.io/guides)
- [Baeldung教程](https://www.baeldung.com/)
- [Spring Cloud中文网](https://springcloud.cc/)

## 🤝 贡献指南

### 如何贡献
1. Fork本仓库
2. 创建特性分支
3. 提交更改
4. 推送到分支
5. 创建Pull Request

### 贡献内容
- 文档改进
- 代码示例
- 错误修复
- 新功能添加

## 📞 联系方式

### 问题反馈
- 创建Issue
- 发送邮件
- 加入讨论群

### 技术支持
- 查看常见问题
- 搜索相关文档
- 咨询技术专家

## 📄 许可证

本项目采用MIT许可证，详情请查看[LICENSE](LICENSE)文件。

---

**开始你的微服务中间件学习之旅吧！** 🎉

**记住：微服务不是银弹，要根据业务需求选择合适的架构！** 🎯
