[云框架]基于Spring Cloud的微服务架构实例[PiggyMetrics](https://github.com/lennyhuanga/springclound)，点击查看[用户指南](https://github.com/lennyhuanga/springcloud-user-guide)
http://blog.csdn.net/F1576813783/article/details/76805195
如何使用Spring Boot、Spring Cloud、Docker和Netflix的一些开源工具来构建一个微服务架构。

本文通过使用Spring Boot、Spring Cloud和Docker构建的概念型应用示例，提供了了解常见的微服务架构模式的起点。

我选择了一个老项目作为这个系统的基础，它的后端以前是单一应用。此应用提供了处理个人财务、整理收入开销、管理储蓄、分析统计和创建简单预测等功能。

功能服务
整个应用分解为三个核心微服务。它们都是可以独立部署的应用，围绕着某些业务功能进行组织。

使用Spring Cloud和Docker构建微服务架构

账户服务

包含一般用户输入逻辑和验证 官网：www.fhadmin.org：收入/开销记录、储蓄和账户设置。

方法	路径	描述	用户验证	UI可用
GET	/accounts/{account}	获取指定账户数据	 	 
GET	/accounts/current	获取当前账户数据	x	x
GET	/accounts/demo	获取演示账户数据（预先填入收入/开销记录等）	 	x
PUT	/accounts/current	保存当前账户数据	x	x
POST	/accounts/	注册新账户	 	x
统计服务

计算主要的统计参数，并捕获每一个账户的时间序列。数据点包含基于货币和时间段正常化后的值。该数据可用于跟踪账户生命周期中的现金流量动态。

方法	路径	描述	用户验证	UI可用
GET	/statistics/{account}	获取指定账户统计数据	 	 
GET	/statistics/current	获取当前账户的统计数据	x	x
GET	/statistics/demo	获取演示账户统计数据	 	x
PUT	/statistics/{account}	创建或更新指定账户的时间序列数据点。	 	 
通知服务

存储用户的联系信息和通知设置（如提醒和备份频率）。安排工作人员从其它服务收集所需的信息并向订阅的客户发送电子邮件。

方法	路径	描述	用户验证	UI可用
GET	/notifications/settings/current	获取当前账户的通知i设置	x	x
PUT	/notifications/settings/current	保存当前账户的通知设置	x	x
注意

每一个微服务拥有自己的数据库，因此没有办法绕过API直接访问持久数据。
在这个项目中，我使用MongoDB作为每一个服务的主数据库。拥有一个多种类持久化架构（polyglot persistence architecture）也是很有意义的。
服务间（Service-to-service）通信是非常简单的：微服务仅使用同步的REST API进行通信。现实中的系统的常见做法是使用互动风格的组合。例如，执行同步的GET请求检索数据，官网：www.fhadmin.org 并通过消息代理（broker）使用异步方法执行创建/更新操作，以便解除服务和缓冲消息之间的耦合。然而
基础设施服务

分布式系统中常见的模式，可以帮助我们描述核心服务是怎样工作的，可以增强Spring Boot应用的行为来实现这些模式。我会简要介绍一下：

基础设施服务

配置服务

Spring Cloud Config是分布式系统的水平扩展集中式配置服务。它使用了当前支持的本地存储、Git和Subversion等可拔插存储库层（repository layer）。

在此项目中，我使用了native profile，它简单地从本地classpath下加载配置文件。您可以在配置服务资源中查看shared目录。现在，当通知服务请求它的配置时，配置服务将响应回shared/notification-service.yml和shared/application.yml（所有客户端应用之间共享）。

客户端使用

只需要使用sprng-cloud-starter-config依赖构建Spring Boot应用，自动配置将会完成其它工作。

现在您的应用中不需要任何嵌入的properties，只需要提供有应用名称和配置服务url的bootstrap.yml即可：

spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
使用Spring Cloud Config，您可以动态更改应用配置

比如，EmailService bean使用了@RefreshScope注解。这意味着您可以更改电子邮件的内容和主题，而无需重新构建和重启通知服务应用。

首先，在配置服务器中更改必要的属性。然后，对通知服务执行刷新请求：curl -H "Authorization: Bearer #token#" -XPOST http://127.0.0.1:8000/notifications/refresh。

您也可以使用webhook来自动执行此过程。

注意

动态刷新存在一些限制。@RefreshScope不能和@Configuraion类一同工作，并且不会作用于@Scheduled方法。
fail-fast属性意味着如果Spring Boot应用无法连接到配置服务，将会立即启动失败。当您一起启动所有应用时，这非常有用。
下面有重要的安全提示
授权服务

负责授权的部分被完全提取到单独的服务器，它为后端资源服务提供OAuth2令牌。授权服务器用于用户授权以及在周边内进行安全的机器间通信。

在此项目中，我使用密码凭据作为用户授权的授权类型（因为它仅由本地应用UI使用）和客户端凭据作为微服务授权的授权类型。

Spring Cloud Security提供了方便的注解和自动配置，使其在服务器端或者客户端都可以很容易地实现。您可以在文档中了解到更多信息，并在授权服务器代码中检查配置明细。

从客户端来看，一切都与传统的基于会话的授权完全相同。您可以从请求中检索Principal对象、检查用户角色和其它基于表达式访问控制和@PreAuthorize注解的内容。

PiggyMetrics（帐户服务、统计服务、通知服务和浏览器）中的每一个客户端都有一个范围：用于后台服务的服务器、用于浏览器展示的UI。所以我们也可以保护控制器避免受到外部访问，例如：

@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
    return statisticsService.findByAccountName(name);
}
API网关

您可以看到，有三个核心服务。它们向客户端暴露外部API。在现实系统中，这个数量可以非常快速地增长，同时整个系统将变得非常复杂。实际上，一个复杂页面的渲染可能涉及到数百个服务。

理论上，客户端可以直接向每个微服务直接发送请求。但是这种方式是存在挑战和限制的，如果需要知道所有端点的地址，分别对每一段信息执行http请求，将结果合并到客户端。另一个问题是，这不是web友好协议，可能只在后端使用。

通常一个更好的方法是使用API网关。它是系统的单个入口点，用于通过将请求路由到适当的后端服务或者通过调用多个后端服务并聚合结果来处理请求。此外，它还可以用于认证、insights、压力测试、金丝雀测试（canary testing）、服务迁移、静态响应处理和主动变换管理。

Netflix开源这样的边缘服务，现在用Spring Cloud，我们可以用一个@EnabledZuulProxy注解来启用它。在这个项目中，我使用Zuul存储静态内容（UI应用），并将请求路由到适当的微服务。以下是一个简单的基于前缀（prefix-based）路由的通知服务配置：

zuul:
  routes:
    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false
这意味着所有以/notification开头的请求将被路由到通知服务。您可以看到，里面没有硬编码的地址。Zuul使用服务发现机制来定位通知服务实例以及断路器和负载均衡器，如下所述。

服务发现

另一种常见的架构模式是服务发现。它允许自动检测服务实例的网络位置，由于自动扩展、故障和升级，它可能会动态分配地址。

服务发现的关键部分是注册。我使用Netflix Eureka进行这个项目，当客户端需要负责确定可以用的服务实例（使用注册服务器）的位置和跨平台的负载均衡请求时，Eureka就是客户端发现模式的一个很好的例子。

使用Spring Boot，您可以使用spring-cloud-starter-eureka-server依赖、@EnabledEurekaServer注解和简单的配置属性轻松构建Eureka注册中心（Eureka Registry）。

使用@EnabledDiscoveryClient注解和带有应用名称的bootstrap.yml来启用客户端支持：

spring:
  application:
    name: notification-service
现在，在应用启动时，它将向Eureka服务器注册并提供元数据，如主机和端口、健康指示器URL、主页等。Eureka接收来自从属于某服务的每个实例的心跳消息。如果心跳失败超过配置的时间表，该实例将从注册表中删除。

此外，Eureka还提供了一个简单的界面，您可以通过它来跟踪运行中的服务和可用实例的数量：http://localhost:8761

Eureka仪表盘

负载均衡器、断路器和Http客户端

Netflix OSS提供了另一套很棒的工具。

Ribbon

Ribbon是一个客户端负载均衡器，可以很好地控制HTTP和TCP客户端的行为。与传统的负载均衡器相比，每次线上调用都不需要额外的跳跃——您可以直接联系所需的服务。

它与Spring Cloud和服务发现是集成在一起的，可开箱即用。Eureka客户端提供了可用服务器的动态列表，因此Ribbon可以在它们之间进行平衡。

Hystrix

Hystrix是断路器模式的一种实现，它可以通过网络访问依赖来控制延迟和故障。中心思想是在具有大量微服务的分布式环境中停止级联故障。这有助于快速失败并尽快恢复——自我修复在容错系统中是非常重要的。

除了断路器控制，在使用Hystrix，您可以添加一个备用方法，在主命令失败的情况下，该方法将被调用以获取默认值。

此外，Hystrix生成每个命令的执行结果和延迟的度量，我们可以用它来监视系统的行为。

Feign

Feign是一个声明式HTTP客户端，能与Ribbon和Hystrix无缝集成。实际上，通过一个spring-cloud-starter-feign依赖和@EnabledFeignClients注解，您可以使用一整套负载均衡器、断路器和HTTP客户端，并附带一个合理的的默认配置。

以下是账户服务的示例：

@FeignClient(name = "statistics-service")
public interface StatisticsServiceClient {
    @RequestMapping(method = RequestMethod.PUT, value = "/statistics/{accountName}", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    void updateStatistics(@PathVariable("accountName") String accountName, Account account);
}
您需要的只是一个接口
您可以在Spring MVC控制器和Feign方法之间共享@RequestMapping部分
以上示例仅指定所需要的服务ID——statistics-service，这得益于Eureka的自动发现（但显然您可以使用特定的URL访问任何资源）。
监控仪表盘

在这个项目配置中，Hystrix的每一个微服 官网：www.fhadmin.org 务都通过Spring Cloud Bus（通过AMQP broker）将指标推送到Turbine。监控项目只是一个使用了Turbine和Hystrix仪表盘的小型Spring Boot应用。

让我们看看系统行为在负载下：账户服务调用统计服务和它在一个变化的模拟延迟下的响应。响应超时阈值设置为1秒。

监控仪表盘

日志分析

集中式日志记录在尝试查找分布式环境中的问题时非常有用。Elasticsearch、Logstash和Kibana技术栈可让您轻松搜索和分析您的日志、利用率和网络活动数据。在我的另一个项目中已经有现成的Docker配置。

安全
高级安全配置已经超过了此概念性项目的范围。官网：www.fhadmin.org  为了更真实地模拟真实系统，请考虑使用https和JCE密钥库来加密微服务密码和配置服务器的properties内容（有关详细信息，请参阅文档）。

基础设施自动化
部署微服务比部署单一的应用的流程要复杂得多，因为它们相互依赖。拥有完全基础设置自动化是非常重要的。我们可以通过持续交付的方式获得以下好处：

随时发布软件的能力。
任何构建都可能最终成为一个发行版本。
构建工件（artifact）一次，根据需要进行部署。
这是一个简单的持续交付工作流程，在这个项目的实现：

在此配置中，Travis CI为每一个成功的Git推送创建了标记镜像。因此，每一个微服务在Docker Hub上的都会有一个latest镜像，而较旧的镜像则使用Git提交的哈希进行标记。如果有需要，可以轻松部署任何一个，并快速回滚。

基础设施自动化

如何运行全部？
这真的很简单，我建议您尝试一下。请记住，您将要启动8个Spring Boot应用、4个MongoDB实例和RabbitMq。确保您的机器上有4GB的内存。您可以随时通过网关、注册中心、配置、认证服务和账户中心运行重要的服务。

运行之前

安装Docker和Docker Compose。
配置环境变量：CONFIG_SERVICE_PASSWORD, NOTIFICATION_SERVICE_PASSWORD, STATISTICS_SERVICE_PASSWORD, ACCOUNT_SERVICE_PASSWORD, MONGODB_PASSWORD
生产模式

在这种模式下，所有最新的镜像都将从Docker Hub上拉取。只需要复制docker-compose.yml并执行docker-compose up -d即可。

开发模式

如果您想自己构建镜像（例如，在代码中进行一些修改），您需要克隆所有仓库（repository）并使用Mavne构建工件（artifact）。然后，运行docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

docker-compose.dev.yml继承了docker-compose.yml，附带额外配置，可在本地构建镜像，并暴露所有容器端口以方便开发。

重要的端点（Endpoint）

localhost:80 —— 网关
localhost:8761 —— Eureka仪表盘
localhost:9000 —— Hystrix仪表盘
localhost:8989 —— Turbine stream（Hystrix仪表盘来源）
localhost:15672 —— RabbitMq管理
注意

所有Spring Boot应用都需要运行配置服务器才能启动。得益于Spring Boot的fail-fast属性和docker-compsoe的restart:always选项，我们可以同时启动所有容器。这意味着所有依赖的容器将尝试重新启动，直到配置服务器启动运行为止。

此外，服务发现机制在所有应用启动后需要一段时间。在实例、Eureka服务器和客户端在其本地缓存中都具有相同的元数据之前，任何服务都不可用于客户端发现，因此可能需要3次心跳。默认的心跳周期为30秒。
