# BluePlan Research 部署指南

## 📋 部署方案概览

你的架构**完全支持自托管**，提供以下部署选项：

### 🎯 推荐部署方案

1. **阿里云ACK + Redis云服务** (最推荐)
2. **AWS EKS + ElastiCache**
3. **Docker Compose** (中小规模)

## 🚀 快速开始

### 方案1：Docker Compose 部署（推荐用于快速测试）

```bash
# 1. 克隆项目
git clone <your-repo>
cd blueplan_research

# 2. 配置环境变量
cp env.example .env
# 编辑 .env 文件，设置你的API密钥

# 3. 一键部署
chmod +x deploy.sh
./deploy.sh latest docker production

# 4. 访问服务
# API: http://localhost
# 文档: http://localhost/docs
# 健康检查: http://localhost/health
```

### 方案2：Kubernetes 部署（推荐用于生产）

```bash
# 1. 部署到K8s集群
./deploy.sh latest k8s production

# 2. 端口转发测试
kubectl port-forward svc/blueplan-api-service 8080:80 -n blueplan

# 3. 访问服务
# API: http://localhost:8080
```

## 🌐 云平台详细配置

### 阿里云 ACK 部署（最推荐）

#### 优势
- ✅ 与现有阿里云Redis深度集成
- ✅ 中国大陆访问速度快
- ✅ 成本相对较低
- ✅ 支持弹性伸缩

#### 配置步骤

1. **创建ACK集群**
```bash
# 创建标准托管集群
aliyun cs CreateCluster \
  --name blueplan-cluster \
  --kubernetes-version 1.24.6-aliyun.1 \
  --region cn-hangzhou \
  --instance-type ecs.c6.large \
  --num-of-nodes 3
```

2. **配置kubectl**
```bash
# 下载集群配置
aliyun cs DescribeClusterKubeConfig --ClusterId <cluster-id> > ~/.kube/config
```

3. **部署应用**
```bash
# 修改k8s-deployment.yaml中的Redis配置
# 然后部署
kubectl apply -f k8s-deployment.yaml
```

### AWS EKS 部署

#### 优势
- ✅ 全球化支持
- ✅ Fargate无服务器选项
- ✅ 丰富的生态系统

#### 配置步骤

1. **创建EKS集群**
```bash
eksctl create cluster \
  --name blueplan-cluster \
  --version 1.24 \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10
```

2. **配置ElastiCache Redis**
```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id blueplan-redis \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-nodes 1
```

## ⚡ 高并发优化配置

### 1. 应用层优化

编辑 `config/app_config.yaml`:

```yaml
performance:
  max_concurrent_users: 1000      # 增加并发用户数
  max_concurrent_agents: 500      # 增加Agent并发数
  response_timeout: 300           # 增加响应超时
  llm_timeout: 120               # 增加LLM超时

api:
  timeout: 600                   # 增加API超时
```

### 2. Redis 优化

```yaml
memory:
  redis_host: "your-redis-cluster-endpoint"
  redis_port: 6379
  max_sessions: 50000           # 增加会话存储
  session_timeout: 3600         # 1小时会话超时
```

### 3. Kubernetes HPA 配置

已在 `k8s-deployment.yaml` 中配置：

- **最小副本数**: 3
- **最大副本数**: 20  
- **CPU阈值**: 70%
- **内存阈值**: 80%

### 4. 数据库连接池优化

在你的代码中已经使用了Redis连接池，这很好！

## 📊 监控和观测

### 1. Kubernetes 监控

```bash
# 查看Pod状态
kubectl get pods -n blueplan

# 查看HPA状态
kubectl get hpa -n blueplan

# 查看资源使用
kubectl top pods -n blueplan
```

### 2. 应用监控

你的项目已集成LangSmith，可以监控：
- Agent执行链路
- LLM调用性能  
- 错误率统计

### 3. 添加Prometheus监控

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'blueplan-api'
      static_configs:
      - targets: ['blueplan-api-service:8000']
```

## 🔧 运维最佳实践

### 1. 健康检查

你的应用已包含 `/health` 端点，Kubernetes会自动进行健康检查。

### 2. 日志管理

```bash
# 查看应用日志
kubectl logs -f deployment/blueplan-api -n blueplan

# 使用ELK Stack收集日志
# 可部署Fluentd + Elasticsearch + Kibana
```

### 3. 备份策略

```bash
# Redis数据备份
redis-cli --rdb /backup/dump.rdb

# 配置文件备份
kubectl get configmap -n blueplan -o yaml > backup/configmaps.yaml
```

### 4. 滚动更新

```bash
# 更新应用
kubectl set image deployment/blueplan-api blueplan-api=blueplan/research:v2.0.0 -n blueplan

# 回滚
kubectl rollout undo deployment/blueplan-api -n blueplan
```

## 💰 成本优化建议

### 1. 阿里云成本预估

| 资源 | 规格 | 月费用(CNY) |
|------|------|-------------|
| ACK集群 | 3节点 2C4G | ~800 |
| Redis | 1G内存 | ~200 |
| SLB负载均衡 | 标准版 | ~50 |
| **总计** | | **~1050/月** |

### 2. AWS成本预估

| 资源 | 规格 | 月费用(USD) |
|------|------|-------------|
| EKS集群 | 3节点 t3.medium | ~150 |
| ElastiCache | cache.t3.micro | ~15 |
| ALB | 标准版 | ~25 |
| **总计** | | **~190/月** |

### 3. 成本优化策略

- 使用Spot实例降低计算成本
- 配置弹性伸缩，按需扩缩容
- 使用预留实例获得折扣
- 监控资源使用，及时调整配置

## 🚨 故障排查

### 常见问题

1. **Pod启动失败**
```bash
kubectl describe pod <pod-name> -n blueplan
kubectl logs <pod-name> -n blueplan
```

2. **Redis连接失败**
```bash
# 检查Redis连接
redis-cli -h <redis-host> -p 6379 ping
```

3. **API响应慢**
```bash
# 检查CPU/内存使用
kubectl top pods -n blueplan
# 检查HPA状态
kubectl get hpa -n blueplan
```

4. **LLM调用失败**
```bash
# 检查API密钥配置
kubectl get secret blueplan-secrets -n blueplan -o yaml
```

## 📚 进阶配置

### 1. SSL证书配置

```bash
# 使用cert-manager自动管理SSL证书
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

### 2. 自定义域名

在 `k8s-deployment.yaml` 中修改Ingress配置：

```yaml
spec:
  rules:
  - host: api.your-domain.com  # 修改为你的域名
```

### 3. 多区域部署

```bash
# 创建多个区域的集群
# 使用DNS负载均衡分发流量
```

## 🎯 性能测试

```bash
# 压力测试
ab -n 1000 -c 50 http://your-api-endpoint/health

# WebSocket连接测试  
wscat -c ws://your-api-endpoint/chat
```

## 📞 技术支持

如需帮助，请检查：
1. 应用日志：`kubectl logs -f deployment/blueplan-api -n blueplan`
2. 系统资源：`kubectl top nodes`
3. 网络连接：`kubectl get svc -n blueplan`

---

**恭喜！🎉 你的BluePlan Research现在已经具备了企业级的自托管和高并发能力！** 
