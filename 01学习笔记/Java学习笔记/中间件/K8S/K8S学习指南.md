下面这份路线按 **Java 后端开发者学习 K8S** 来设计，目标不是“看懂概念”，而是最终能做到：

> 把一个 Spring Boot / Spring Cloud / AI 应用从本地 Docker Compose，逐步迁移到 Kubernetes，并具备基本部署、排障、扩缩容、配置管理、监控、灰度发布能力。

Kubernetes 官方定义是用于自动化部署、扩缩容和管理容器化应用的平台；学习时要抓住它的核心：**声明式配置 + 控制器持续调谐 + 集群资源编排**。([Kubernetes](https://kubernetes.io/docs/concepts/overview/?utm_source=chatgpt.com "Overview"))

---

# 一、K8S 学习总路线

```text
阶段 0：前置基础
  ↓
阶段 1：K8S 核心世界观
  ↓
阶段 2：Pod / Deployment / Service / Ingress
  ↓
阶段 3：配置、密钥、存储
  ↓
阶段 4：服务发现、网络、负载均衡
  ↓
阶段 5：调度、资源、弹性伸缩
  ↓
阶段 6：发布策略、健康检查、故障恢复
  ↓
阶段 7：权限、安全、命名空间、多环境
  ↓
阶段 8：日志、监控、链路追踪、排障
  ↓
阶段 9：Helm / Kustomize / GitOps
  ↓
阶段 10：生产实践与云原生体系
```

---

# 二、阶段 0：前置基础

K8S 不是第一个要学的东西。它建立在 Linux、网络、Docker、YAML、CI/CD 之上。

## 你需要先掌握

|方向|必须掌握|
|---|---|
|Linux|进程、端口、文件系统、环境变量、systemd、日志|
|网络|IP、端口、DNS、NAT、反向代理、负载均衡|
|Docker|镜像、容器、Dockerfile、volume、network、compose|
|后端部署|Spring Boot jar、环境变量配置、健康检查|
|YAML|缩进、数组、对象、配置拆分|
|Git|分支、PR、CI/CD 基础|

## 练习目标

用 Docker Compose 跑通：

```text
Spring Boot API
MySQL
Redis
Nginx
```

你要先理解：

```text
单机部署 → Docker 部署 → Docker Compose 编排 → Kubernetes 集群编排
```

否则一上来学 K8S 会非常抽象。

---

# 三、阶段 1：K8S 核心世界观

这一阶段先建立“操作系统级别”的理解。

## 核心问题

K8S 本质上解决什么问题？

```text
我有很多机器
我有很多容器
我希望系统自动决定容器跑在哪台机器
容器挂了能自动拉起
流量能自动转发
配置能统一管理
版本能滚动发布
资源能限制和扩缩容
```

## 必学概念

|概念|你应该怎么理解|
|---|---|
|Cluster|一组机器组成的集群|
|Node|集群里的工作机器|
|Control Plane|集群大脑|
|API Server|所有操作的入口|
|etcd|集群状态数据库|
|Scheduler|决定 Pod 放到哪个 Node|
|Controller Manager|持续维护期望状态|
|kubelet|每台 Node 上的执行代理|
|kubectl|你和集群交互的 CLI|

## 最重要的思想

K8S 不是“你命令它做一步，它做一步”。

而是：

```text
你声明期望状态：
我希望有 3 个订单服务实例

K8S 持续对比：
当前状态是不是 3 个？

如果不是：
自动创建、删除、重启、迁移 Pod
```

这叫：

```text
声明式 API + 控制器模式 + 最终一致性
```

这个思想非常重要，后面所有资源都围绕它展开。

---

# 四、阶段 2：Pod / Deployment / Service / Ingress

这是 K8S 入门最核心的一组资源。

Kubernetes 官方文档把 Workload 定义为运行在 Kubernetes 上的应用，Pod 是最小可部署计算对象，Deployment、StatefulSet 等是更高层的工作负载抽象。([Kubernetes](https://kubernetes.io/docs/concepts/workloads/?utm_source=chatgpt.com "Workloads"))

## 1. Pod

Pod 是 K8S 里最小部署单位。

不要把 Pod 简单理解成 Docker 容器，它更像：

```text
一个应用运行单元
里面可以有一个或多个容器
这些容器共享网络、存储、生命周期
```

常见结构：

```text
Pod
 ├── main container：Spring Boot 应用
 └── sidecar container：日志采集 / 代理 / 辅助进程
```

## 2. Deployment

Deployment 用来管理无状态应用，例如：

```text
用户服务
订单服务
商品服务
网关服务
AI Gateway
```

它负责：

```text
副本数
滚动更新
版本回滚
Pod 自愈
ReplicaSet 管理
```

你真正部署后端服务时，通常不是直接写 Pod，而是写 Deployment。

## 3. Service

Pod 的 IP 是不稳定的，Pod 重启后 IP 会变。

Service 解决的是：

```text
如何稳定访问一组动态变化的 Pod？
```

常见类型：

|类型|用途|
|---|---|
|ClusterIP|集群内部访问|
|NodePort|通过节点端口暴露|
|LoadBalancer|云厂商负载均衡|
|ExternalName|映射外部 DNS|

## 4. Ingress

Ingress 用来做 HTTP/HTTPS 七层入口。

典型用途：

```text
api.example.com       → backend-service
admin.example.com     → admin-service
www.example.com       → frontend-service
```

对应你做博客 / DevWiki / Spring Boot 项目时，Ingress 就相当于集群里的 Nginx 网关入口。

---

# 五、阶段 3：配置、密钥、存储

这一阶段对应后端开发最熟悉的配置问题。

## 1. ConfigMap

用于保存非敏感配置：

```text
application.yml
业务开关
服务地址
日志级别
外部 API endpoint
```

示例：

```text
Spring Boot 启动时读取 ConfigMap 注入的环境变量
```

## 2. Secret

用于保存敏感配置：

```text
数据库密码
Redis 密码
JWT secret
API key
OpenAI / DeepSeek / Claude token
```

注意：K8S Secret 默认只是 base64 编码，不等于强加密。生产环境通常要结合：

```text
云厂商 KMS
External Secrets
Sealed Secrets
Vault
```

## 3. Volume / PVC / PV / StorageClass

K8S 中存储这块需要分层理解：

|资源|作用|
|---|---|
|Volume|Pod 挂载的存储|
|PersistentVolume, PV|集群里的持久化存储资源|
|PersistentVolumeClaim, PVC|应用申请存储|
|StorageClass|动态创建存储的模板|

## 适合放存储的服务

```text
MySQL
PostgreSQL
Redis 持久化
MinIO
Elasticsearch
Prometheus
```

但新手阶段建议先把有状态服务放外部，例如云数据库、本地 Docker Compose 或独立中间件，不要一开始就在 K8S 里折腾 MySQL 高可用。

---

# 六、阶段 4：服务发现、网络、负载均衡

这是 K8S 最容易让后端开发者卡住的地方。

## 必学内容

|概念|说明|
|---|---|
|Pod IP|Pod 自己的 IP|
|Service IP|稳定虚拟 IP|
|Cluster DNS|服务名解析|
|kube-proxy|Service 转发机制|
|CNI|容器网络插件|
|NetworkPolicy|网络访问控制|

## 你要建立这个认知

在 K8S 里，服务之间通常不是这样访问：

```text
http://192.168.1.10:8080
```

而是：

```text
http://order-service:8080
http://user-service.default.svc.cluster.local:8080
```

服务发现靠 DNS，而不是手工维护 IP。

## Java 后端重点

你要知道：

```text
Spring Cloud 注册中心模式
和
K8S Service DNS 服务发现模式
```

两者可以共存，但很多情况下，K8S 本身已经提供了服务发现能力。

例如：

```text
传统 Spring Cloud：
服务 → 注册到 Nacos / Eureka → 客户端发现服务

K8S 模式：
服务 → 通过 Service 暴露 → DNS 自动发现
```

---

# 七、阶段 5：资源、调度、弹性伸缩

这一阶段开始进入“生产感”。

## 1. Resource Request / Limit

每个 Pod 都应该配置资源：

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

含义：

|字段|作用|
|---|---|
|requests|调度时保证的最低资源|
|limits|容器最多能使用的资源|

## 2. HPA

Horizontal Pod Autoscaler，根据指标自动扩缩容：

```text
CPU 使用率高 → Pod 副本数增加
CPU 使用率低 → Pod 副本数减少
```

更高级可以基于：

```text
QPS
队列长度
Prometheus 指标
GPU 利用率
自定义业务指标
```

## 3. 调度机制

你需要理解：

```text
NodeSelector
NodeAffinity
PodAffinity
Taints
Tolerations
```

这些东西决定：

```text
哪些服务可以部署到哪些机器
哪些机器只跑特定服务
高优先级服务如何隔离
AI 推理服务如何调度到 GPU 节点
```

---

# 八、阶段 6：发布、健康检查、故障恢复

这是最贴近实际工作的部分。

## 1. 健康检查

K8S 有三类 Probe：

|Probe|作用|
|---|---|
|livenessProbe|判断容器是否需要重启|
|readinessProbe|判断是否可以接收流量|
|startupProbe|判断启动过程是否完成|

Spring Boot 项目通常接：

```text
/actuator/health/liveness
/actuator/health/readiness
```

## 2. 滚动发布

Deployment 默认支持 RollingUpdate：

```text
先启动新版本 Pod
等新 Pod ready
再逐步下线旧 Pod
```

## 3. 回滚

如果新版本异常，可以：

```bash
kubectl rollout undo deployment/order-service
```

## 4. 发布策略

|策略|说明|
|---|---|
|Recreate|先停旧版本，再启新版本|
|RollingUpdate|滚动更新|
|Blue-Green|蓝绿发布|
|Canary|金丝雀发布|
|Shadow|影子流量|

新手先掌握 RollingUpdate，后面再学 Argo Rollouts / Istio 做灰度。

---

# 九、阶段 7：权限、安全、命名空间、多环境

## 1. Namespace

Namespace 用来隔离环境或团队：

```text
dev
test
staging
prod
monitoring
ingress-nginx
```

不要把所有东西都丢在 default 里。

## 2. RBAC

RBAC 控制谁能操作什么资源。

核心对象：

```text
ServiceAccount
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

## 3. 安全基础

你需要掌握：

```text
镜像最小化
非 root 用户运行容器
禁止特权容器
Secret 管理
NetworkPolicy
镜像漏洞扫描
资源限制
Pod Security Standards
```

## 4. 多环境配置

常见方案：

```text
dev 环境：小资源、本地数据库
test 环境：接测试中间件
prod 环境：高可用、监控、备份、告警
```

可以用：

```text
Kustomize
Helm values
GitOps 分支 / 目录
```

来管理差异。

---

# 十、阶段 8：日志、监控、链路追踪、排障

这是真正从“会部署”到“会运维”的分界线。

## 1. 日志

基础命令：

```bash
kubectl logs pod-name
kubectl logs -f deployment/order-service
kubectl logs pod-name -c container-name
```

生产常见方案：

```text
EFK / ELK
Loki + Promtail + Grafana
云厂商日志服务
```

## 2. 监控

常见组合：

```text
Prometheus
Grafana
Alertmanager
kube-state-metrics
node-exporter
metrics-server
```

要监控：

```text
Node CPU / Memory / Disk
Pod CPU / Memory
容器重启次数
接口 QPS
接口延迟
错误率
JVM 指标
数据库连接池
消息队列积压
```

## 3. 链路追踪

常见组合：

```text
OpenTelemetry
Jaeger
Tempo
SkyWalking
Zipkin
```

Java 后端重点：

```text
一个请求经过 Gateway → User Service → Order Service → Payment Service
如何追踪整条链路？
```

## 4. 排障能力

必须熟悉：

```bash
kubectl get pods
kubectl describe pod xxx
kubectl logs xxx
kubectl exec -it xxx -- sh
kubectl get events
kubectl top pod
kubectl top node
kubectl rollout status deployment xxx
```

常见问题：

```text
ImagePullBackOff
CrashLoopBackOff
Pending
OOMKilled
Readiness probe failed
Liveness probe failed
Service 无法访问
Ingress 404 / 502
DNS 解析失败
PVC 挂载失败
```

---

# 十一、阶段 9：Helm / Kustomize / GitOps

当 YAML 多起来以后，手写和复制粘贴会失控。

## 1. Helm

Helm 类似 K8S 的包管理器。

适合：

```text
安装 MySQL
安装 Redis
安装 Prometheus
安装 Ingress Controller
安装业务系统
```

核心概念：

```text
Chart
Values
Template
Release
```

## 2. Kustomize

Kustomize 更适合管理环境差异：

```text
base/
overlays/dev/
overlays/prod/
```

例如：

```text
dev 副本数 1
prod 副本数 5
dev 用小资源
prod 用大资源
```

## 3. GitOps

GitOps 的核心思想：

```text
Git 仓库声明集群期望状态
控制器自动同步到 K8S
```

常见工具：

```text
Argo CD
Flux
```

推荐学习 Argo CD。

---

# 十二、阶段 10：生产实践与云原生体系

这一阶段不是 K8S 入门，而是云原生工程化。

## 需要逐步了解

|方向|工具 / 概念|
|---|---|
|Ingress|Nginx Ingress、Traefik、APISIX|
|Service Mesh|Istio、Linkerd|
|网关|Kong、APISIX、Spring Cloud Gateway|
|镜像仓库|Harbor、Docker Hub、GHCR|
|CI/CD|GitHub Actions、GitLab CI、Jenkins|
|GitOps|Argo CD、Flux|
|可观测性|Prometheus、Grafana、Loki、Tempo|
|安全|Trivy、OPA Gatekeeper、Kyverno|
|存储|Longhorn、Ceph、云盘|
|证书|cert-manager|
|自动扩缩容|HPA、VPA、KEDA|
|Serverless|Knative|
|AI 工作负载|GPU Node、NVIDIA Device Plugin、KServe、Ray|

---

# 十三、推荐学习项目路线

不要只看视频。你应该做一个完整项目，把每个概念串起来。

## Project 1：单服务部署

目标：

```text
部署一个 Spring Boot Hello API
```

涉及：

```text
Dockerfile
Deployment
Service
Ingress
ConfigMap
Secret
Probe
```

完成标准：

```text
可以通过域名访问 API
可以滚动发布
可以查看日志
可以回滚版本
```

---

## Project 2：博客系统部署

目标：

```text
部署一个前后端分离博客系统
```

组件：

```text
Vue / React 前端
Spring Boot 后端
MySQL
Redis
Nginx Ingress
```

涉及：

```text
Namespace
ConfigMap
Secret
PVC
Service
Ingress
Deployment
StatefulSet
```

完成标准：

```text
前端能访问
后端 API 能访问
数据库能持久化
Redis 能连接
配置不写死在镜像里
```

---

## Project 3：微服务系统部署

目标：

```text
部署一套简化电商系统
```

服务：

```text
gateway-service
user-service
order-service
payment-service
inventory-service
```

涉及：

```text
服务发现
资源限制
HPA
滚动发布
健康检查
日志聚合
链路追踪
```

完成标准：

```text
服务之间通过 K8S Service 通信
网关统一入口
订单服务可以扩缩容
某个服务挂掉后能自动恢复
```

---

## Project 4：DevWiki / AI Gateway 部署

这个更贴近你自己的项目。

组件可以是：

```text
frontend
backend
ai-gateway
postgres / mysql
redis
vector-db
object-storage
worker
scheduler
```

重点练习：

```text
AI API Key 用 Secret 管理
Embedding / RAG worker 独立部署
异步任务 worker 横向扩展
日志和指标监控
网关限流
GitOps 发布
```

这会非常适合你后续做 AI 后端项目。

---

# 十四、建议学习顺序表

## 第一轮：入门闭环

|顺序|主题|目标|
|---|---|---|
|1|Docker 复习|能写 Dockerfile 和 compose|
|2|K8S 架构|理解 Control Plane / Node|
|3|Pod|理解最小运行单元|
|4|Deployment|部署无状态服务|
|5|Service|让服务稳定访问|
|6|Ingress|暴露 HTTP 服务|
|7|ConfigMap / Secret|外部化配置|
|8|Probe|健康检查|
|9|kubectl|基础排障|
|10|项目部署|跑通 Spring Boot API|

---

## 第二轮：生产基础

|顺序|主题|目标|
|---|---|---|
|1|Namespace|环境隔离|
|2|Resource|CPU / Memory 管理|
|3|HPA|自动扩缩容|
|4|PVC|持久化存储|
|5|RBAC|权限控制|
|6|NetworkPolicy|网络隔离|
|7|Helm|应用打包|
|8|Kustomize|多环境配置|
|9|Prometheus / Grafana|监控|
|10|Loki / ELK|日志|

---

## 第三轮：工程化进阶

|顺序|主题|目标|
|---|---|---|
|1|Argo CD|GitOps 发布|
|2|cert-manager|自动证书|
|3|Ingress Controller|入口网关|
|4|Service Mesh|流量治理|
|5|Canary 发布|灰度发布|
|6|OpenTelemetry|链路追踪|
|7|KEDA|事件驱动扩缩容|
|8|镜像安全|漏洞扫描|
|9|多集群|高可用架构|
|10|AI workload|GPU / 推理服务部署|

---

# 十五、K8S 学习中的关键心法

## 1. 不要背资源名，要理解它解决什么问题

例如：

```text
Deployment 解决“如何维持多个无状态副本”
Service 解决“Pod IP 不稳定，如何访问”
Ingress 解决“HTTP 流量如何进入集群”
ConfigMap 解决“配置如何和镜像解耦”
Secret 解决“敏感信息如何注入”
PVC 解决“数据如何持久化”
HPA 解决“流量变化时如何自动扩缩容”
```

---

## 2. kubectl 是第一生产力

至少熟练这些：

```bash
kubectl get
kubectl describe
kubectl logs
kubectl exec
kubectl apply
kubectl delete
kubectl rollout
kubectl top
kubectl port-forward
kubectl explain
```

尤其是：

```bash
kubectl describe pod xxx
```

这个命令非常关键，很多问题不是看 logs，而是看 events。

---

## 3. 先单机集群，再云上集群

推荐顺序：

```text
Docker Desktop Kubernetes / Minikube / Kind
  ↓
本地多服务部署
  ↓
云服务器 k3s
  ↓
云厂商托管 K8S：ACK / EKS / GKE / AKS
```

如果只是个人学习，建议：

```text
Kind 或 Minikube 入门
k3s 做服务器实战
托管 K8S 做生产理解
```

---

## 4. 不要一开始就冲 Service Mesh

学习顺序应是：

```text
Deployment / Service / Ingress
  ↓
Config / Secret / PVC
  ↓
Probe / HPA / Resource
  ↓
Helm / Argo CD / Observability
  ↓
Istio / Service Mesh
```

Istio 很强，但不是入门第一优先级。

---

# 十六、最终学习地图

```text
K8S 基础层
├── Cluster / Node / Control Plane
├── Pod / Deployment / ReplicaSet
├── Service / Ingress
├── ConfigMap / Secret
└── Volume / PVC

K8S 运维层
├── kubectl
├── logs / describe / events
├── health check
├── resource limit
├── HPA
└── rollout / rollback

K8S 工程层
├── Namespace
├── RBAC
├── Helm
├── Kustomize
├── GitOps
└── CI/CD

K8S 生产层
├── Ingress Controller
├── Prometheus / Grafana
├── Loki / ELK
├── OpenTelemetry
├── cert-manager
├── NetworkPolicy
└── Security

K8S 高阶层
├── Service Mesh
├── Canary / Blue-Green
├── Multi Cluster
├── KEDA
├── GPU Workload
└── Platform Engineering
```

---

# 十七、给你的建议路线

结合你现在的背景，我建议你按这个顺序学：

```text
1. Docker → Docker Compose 复习
2. K8S 架构总览
3. Pod / Deployment / Service / Ingress
4. 用 K8S 部署一个 Spring Boot API
5. ConfigMap / Secret / Probe / Resource
6. 部署 Spring Boot + MySQL + Redis
7. Namespace / RBAC / PVC
8. HPA / 滚动发布 / 回滚
9. Prometheus / Grafana / Loki
10. Helm / Kustomize / Argo CD
11. 部署一个完整 DevWiki / AI Gateway 项目
12. 再学 Service Mesh、GitOps、AI workload
```

最小闭环不要超过两周，第一目标是：

> 在本地 K8S 集群里，把一个 Spring Boot 服务通过 Deployment + Service + Ingress 跑起来，并能做到配置外置、健康检查、滚动发布、日志查看、故障排查。

后面可以按这个路线逐章展开。下一步最适合从 **“K8S 是什么，以及它和 Docker / Docker Compose / Spring Cloud 的关系”** 开始。