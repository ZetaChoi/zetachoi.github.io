---
title: KTech New Platfrom 架构原则
date: 2024-04-29 13:34:02 +0800
categories: [企业架构设计]
tags: [架构设计]
---

基于![技术架构选型]中对各个架构模式的理解，并结合银行产品特性以及对云原生架构的适应性，本文说明KTECH New Platfrom的架构原则。

## 架构模式
KTECH New Platfrom选择以Mini-Service架构为基础的微服务模式，遵循以下原则：
1. 项目工程模块以微服务方式设计，并组合为单个web项目部署，从而确保在拆分服务时更便捷。
2. 允许多个业务模块共享同一个数据库，同时仅在数据库层完全解耦时允许微服务拆分。
3. 通用服务与业务服务间通过REST API或异步方式通信。
4. 除微服务网关，不同微服务必须采用基于事件的异步方式通信。
5. 微服务间应各自保证可用性。

### 架构原则说明：

#### 原则1
考虑到目前银行项目的性能要求和硬件资源成本，采用New Platfrom的项目应从单体模式开始发展架构，同时应保持拆分微服务的能力，参考以下New Platfrom项目结构：

~~~
Halo-Cloud     
├── halo-gateway         // 网关模块 [8080]
├── halo-admin           // 管理端 [9200]
├── halo-api             // 接口模块
│       └── halo-api-resource                        // 资源api模块
│       └── halo-api-system                          // 系统接口
├── halo-common          // 通用模块
│       └── halo-common-core                         // 核心功能模块
│       └── ruoyi-common-dict                        // 字典集成模块
│       └── halo-common-datasource                   // 多数据源
│       └── halo-common-log                          // 日志记录
│       └── halo-common-redis                        // 缓存服务
│       └── halo-common-seata                        // 分布式事务
│       └── halo-common-security                     // 安全模块
├── halo-modules         // 业务功能
│       └── halo-system                              // 系统模块 [9201]
│       └── halo-job                                 // 定时任务 [9202]
│       └── halo-file                                // 文件服务 [9300]
│       └── halo-auth                                // 认证中心 [9200]
├──pom.xml                // 公共依赖
~~~

`halo-admin`本身不包含业务功能，而是作为web容器集成各个模块；`halo-modules`是按照微服务领域拆分的业务模块，通常被集成到`halo-admin`中部署为单个web项目，也可以拆分为以微服务单独部署。`halo-common`是通用模块，以jar包形式集成到各个业务模块中使用。

这么一来从部署模式上是单体架构，构建小型项目时可以节省硬件资源，必要时也可以微服务模式拆分部署，从而适应更高的性能和负载需求。

#### 原则2
鉴于前文中提到的银行系统的高数据一致性需求，通常难以基于业务拆分数据库，强行拆分还可能导致更高的分布式事务成本。因此平台以Mini-Service模式为指导思想，允许不同业务领域共享数据库，但此时不应拆分微服务。服务若存在相互抢占资源现象，仍然保持集群部署，并通过负载均衡策略将不同服务请求分别代理到不同服务组。反过来说，如果业务数据库可完全解耦，则可以考虑进行微服务拆分，形成更清晰的项目结构。

#### 原则3
短信服务中心、工作流中心、文件中心等服务，属于通用基础功能，本身不包含业务逻辑，可以拆分微服务。并且，由于业务服务对基础功能基本上是强依赖的（无论工作流引擎是集成到业务服务还是单独部署，引擎不可用均会导致业务功能异常，不适用服务降级）因此保证通信可靠即可，而基于Http的通信在内网中完全可靠，因此允许通用服务与业务服务间通过REST API方式通信。

#### 原则4
不同微服务必须采用基于事件的异步方式通信，反之则不应拆分微服务。例如在信贷业务中，SCF服务与Lending服务共同完成授信业务，此时相较于REST API，事件驱动的通信模式可保证服务不会因为某个业务环节异常而丢失，因此完全适合拆分为两个服务，分别关注SCF产品功能和贷款审批功能。

#### 原则5
在SpringCloud体系中，微服务应注册到同一个注册中心，从而监控整个集群健康度。基于前几条原则，在KTech Platform的设计中不存在超大型的微服务集群和超长的业务链路，此时每个服务各自保证可用性在管理时更为清晰。就像，在项目中集成Apollo配置中心时，完全可以让Apollo单独部署到自己的Eureka中，业务服务不需关注Apollo的可用性。

这样做的另一个好处是，服务不一定需要Eureka等注册中心组件，通过Nginx+集群的模式也能保证服务高可用（此时牺牲的是网关动态发现服务的能力），此时在K8s体系下可以以云原生方式部署从而更好地利用云原生架构优势。更利于项目后期在SpringCloud体系和K8s体系下演进。

## 技术框架选型
为了兼容Mini-Service(SpringBoot)/MicroService(SpringCloud)/云原生(K8s)架构，同时保持项目架构简洁清晰，KTech New Platfrom选择以下微服务组件：

### 注册中心
采用Nacos做注册中心
- 注册中心组件本身代码侵入性低，无论是k8s切换到spring cloud，还是spring cloud到k8s，迁移难度都不高。

### 服务间通信/负载均衡
采用经裁剪的Feign+Ribbon
- 相较原生Rest API，经过RPC包装后的协议使用更方便，也利于框架统一处理通信层逻辑。
- 依照Ktech Platform架构设计原则，采用url配置FeignClient，从而更好地兼容云原生场景。
```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(url = "http://remote-service:8080")
public interface RemoteServiceClient {
    @GetMapping("/hello")
    String sayHello();
}
```

### 熔断/限流/降级
小规模项目不采用 > Istio+Sentinel > Sentinel

- 由于该组件代码侵入性非常高，架构迁移的成本也会更高。目前没有较兼容的解决方案，最好在项目架设初期就确定采用的方案。

### 链路监控
采用Zipkin
- Zipkin在SpringCloud和K8s体系均有较好表现

### 消息总线
采用RabbitMQ
- RabbitMQ相较RocketMQ在AWS等海外云中支持性更好
- RabbitMQ适合银行系统等可靠性要求较高的场景

## 服务拆分指引

### 基于组织结构拆分
康威定律: 一个组织的系统通常被设计成这个组织通信结构的副本。

> 定律一: 组织沟通方式会通过系统设计表达出来，就是说架构的布局和组织结构会有相似。    
定律二: 时间再多一件事情也不可能做的完美，但总有时间做完一件事情。一口气吃不成胖子，先搞定能搞定的。    
定律三: 线型系统和线型组织架构间有潜在的异质同态特性。种瓜得瓜，做独立自治的子系统减少沟通成本。    
定律四: 大的系统组织总是比小系统更倾向于分解。合久必分，分而治之。

### 基于领域（业务模块）拆分
微服务只是手段，不是目的。从业务出发，通过领域模型的方式反映系统的抽象，围绕业务领域按职责单一性、功能完整性拆分。

考虑以下几个阶段：
- 进行业务建模，从业务中获取抽象的模型（例如订单、用户），根据模型的关系进行划分限界上下文。
- 检验模型是否得到合适的的抽象，并能反映系统设计和响应业务变化。
- 从限界上下文往微服务转化，并得到系统架构、API列表、集成方式等产出。
