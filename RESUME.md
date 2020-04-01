
## 基本信息

- 男 / 1988 / 8 年运维经验
- 山东大学 / 本科 / 环境工程
- 电话：186******21
- 邮箱：ab****y@gmail.com
- 期望职位：SRE/Dev

## 工作经历

### 2011 - 2015 学信咨询服务有限公司

- IDC 运维，高可用网络规划，交换路由配置
- 操作系统自动化安装，配置管理，系统优化
- 监控系统搭建及维护，故障处理
- 线上业务维护，运维平台开发，自动化运维

### 2015 - ？ 北京卡路里信息技术有限公司（Keep）

- 公有云平台运维，云上架构设计优化，VPC 网络及专线管理
- 数据库、消息队列、缓存等基础架构设计与搭建、维护、故障排除
- 线上业务运维，业务架构设计及优化
- 监控报警系统建设与维护
- CI/CD 规范制定及流程开发，自动化运维平台设计开发
- 日志系统、发布系统等设计及开发

## 技术能力

- 熟悉 TCP/IP、UDP 协议，OSI 网络模型，链路层及交换、路由原理
- 熟悉 H3C/Cisco 交换路由及防火墙配置
- 熟悉 DNS、ICMP、SMTP、HTTP(S)、HTTP2 等各种常见协议
- 熟悉 Linux 操作系统（RedHat/CentOS、Ubuntu 等）
- 熟练使用 tcpdump、wireshark 等网络故障排查工具
- 熟悉 zookeeper、etcd 等分布式一致性系统
- 熟悉 Kafka、RocketMQ、RabbitMQ 等消息队列
- 熟悉 Nginx 各种使用场景配置及优化，熟悉 OpenResty+Lua 开发
- 熟悉 MongoDB Replicaset & Sharded Cluster 原理及维护，查询性能分析、索引优化
- 熟悉 Redis 数据结构，性能监控，Codis 集群原理及维护，故障排查
- 熟悉 ElasticSearch 集群原理，搜索及日志场景优化
- 熟悉 CDH 集群维护，Hadoop HDFS、HBase、Yarn 等
- 熟悉 ELK 系列日志处理架构，以及 beats 系列，熟悉 loki 日志系统
- 熟悉 Open-Falcon、Nagios、Zabbix 等监控系统，熟悉 Grafana，了解 Prometheus
- 熟悉 Docker 原理及应用，了解 Kubernetes
- 熟悉 Git 版本管理操作，熟悉 Phabricator 维护及使用
- 熟悉 Saltstack/Puppet 自动化配置管理工具
- 开发能力：Go、Python、Shell、Angular

## 项目经验

### **OpenResty 基于 etcd 的动态 upstream 模块开发**

#### 背景

使用 NginX 作为反向代理为后端服务进行负载均衡的情况下，为了避免后端服务发布或者重启时造成请求错误，通常操作方式是手动修改配置文件中的 upstream 指令并进行 reload 操作。随着后端服务规模变大，这种操作方式带来以下问题：

1. 操作流程随着接入服务数量增加变复杂，且容易出错
1. 新增加服务也需要修改配置并 reload
1. 无法多个服务同时发布或者重启
1. 无法实现灵活的权重调整
1. 无法完成精细的灰度流量策略
1. 无法快速屏蔽或者新增后端服务实例
1. reload 会释放所有的 TCP 连接，客户端同时新建连接相当于 syn-flood 攻击

#### 目标

在无需 reload 的情况下实现如下功能：

- 实现动态的增加、删除后端服务（upstream）
- 实现动态的增加、删除、修改特定服务的实例
- 实现必要的负载均衡算法
- 实现实例动态权重设置

#### 设计

- **方案一**：使用 confd 等配置生成工具配合 consul/etcd 等组件实时监听并重写 nginx 配置文件，并执行 reload。该方案我们使用了一段时间，通过配合 etcd 服务注册的方式，很好的解决了问题 1，但是对于其他问题仍然存在。
- **方案二**：微博开源了 nginx-upsync-module，但是当时并不支持 etcd 注册中心，而我们在使用方案一时已经将服务注册到 etcd 中，并且 nginx-upsync-module 不支持动态添加 upstream，仅支持已配置的 upstream 内实例列表的实时变更。
- **方案三**：基于 OpenResty 使用 lua 自行开发满足需求的模块，使用 etcd 作为注册中心，复用已有的注册信息，实时监听变化并按照权重分发请求。

#### 成果

- 一套 Nginx 集群支持 300+ 后端服务组，2000+ 服务实例，日发布/重启次数 50+，无需修改任何 Nginx 配置，无需 reload 操作。
- 支持监听 Kubernetes apiserver 获取 Kubernetes service 对应的 Pod 列表，进行 Kubernetes 集群内外的混合流量转发。
- 实现了支持权重配置的 round-robin 负载均衡算法，可以实时动态调整权重。
- 支持主动式健康检查，通过发起 HTTP 探测请求到每个实例获取实例健康状态；支持被动式健康检查，通过分析 response status 将错误实例踢出服务列表。
- 支持接口级的状态码统计，可以输出最近 10min 内任意时间段的响应状态码信息。

### **ELK + Grafana loki 日志系统建设**

为了满足日志的存储和查询需求，针对不同的日志类型设计了不同的处理方式，并且尽量做到架构复用和降低成本。所有日志均使用 filebeat 作为收集器，输出到特定的 kafka 集群中，根据类型区分 topic，进行不同的后端处理。

Nginx 访问日志结构化清晰，经常需要精确的查询和统计分析，所以采用 Elasticsearch 存储，Kibana 查看。单日日志量达 1TB，采用冷热分离的 Elasticsearch 集群，热数据节点使用 SSD 磁盘存储，提高查询速度，冷数据节点采用 HDD，降低存储成本。超过 30 天的日志进行索引关闭操作，降低集群压力。

业务日志结构比较混乱，无需精细化存储，数据量大且仅需要支持模糊搜索，经过调研选择了 Grafana 开源的 loki 方案，并对 loki 进行适配性的改造，复用 filebeat -> kafka 的日志流而非增加 promtail 作为新的日志收集器。loki 使用 OSS 进行存储，存储成本极低，Grafana Explore 功能进行查看，支持按标签过滤和关键字模糊搜索。

### **Banana 发布系统开发及 CI/CD 流程设计及实现**

鉴于当时第一版发布系统使用不便，速度较低，团队扩大后无法满足更快速的迭代需求，因此开发了新的发布系统。后端使用 Go 开发并发度得到大幅提升，前端使用 Angular 配合 Angular Material 开发，简洁易用。支持多种语言，支持多环境、多集群，支持 Docker 模式发布。使用 SSH 连接到服务器进行操作，无需 agent。支持将构建结果存储到 aliyun OSS 或者 qcloud COS。支持发布时健康检查，灰度发布，每个发布任务记录详细的发布日志。当前每日支持数百次发布，包括数千实例。

基于 Phabricator 的功能实现提交代码后自动化触发 Jenkins CI 集群进行构建和单元测试，并在必要时构建程序版本并存储，用于后续发布。Jenkins 采用容器化的集群部署，扩容仅需要配置正确的 SSH 凭据。使用 Pipeline Shared Library 统一管理 CI 流程，代码中仅需添加 Jenkinsfile 和 build.yaml 两个文件即可无痛接入。根据配置可以将 master 分支自动部署到测试环境，并调用 apitest 服务进行集成测试。

### **其他项目**

- **服务容器化及 Kubernetes 推进**：设计了完整的改造和迁移方案，尽量兼容现有系统的情况下将业务迁移到 Kubernetes 上。实现 Banana 发布系统对接 Kubernetes API server 发布（Deployment），Falcon 监控系统改造，引入 Prometheus 监控系统进行业务监控，Jenkins CI 自动构建标准化的 Docker 镜像等。
- **MongoDB billion 级表 Sharding 改造**：ReplicaSet 集群中单表超过十亿数据量，查询性能无法保证，根据业务状况设计了 MongoDB Sharded Cluster 方案并指导实施。
- **阿里云经典网络到 VPC 迁移**：主导并带领团队将线上服务不中断、无故障的平滑迁移到 VPC 网络中，圆满完成任务。
- **基于 saltstack 的自动化运维建设**：全部服务器部署 saltstack 并制定相应规范，编写 formula 示例在团队内推广使用，所有基础设施部署均实现了自动化。
- **Redis 迁移至 Codis 并持续维护**：在 Redis 单机实例无法满足业务需求的情况下，选择 Codis 作为可扩展的缓存方案进行迁移，并制定相应的维护规范，开发对应的监控程序等。
- **Open Falcon 监控系统改造**：原有的 Zabbix 监控系统无法满足日益增长的监控需求，选择了 Open Falcon 作为新的监控方案部署并实施，编写 alarm-api 实现定制的报警消息推送等。
