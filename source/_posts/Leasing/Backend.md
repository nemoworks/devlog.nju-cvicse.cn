---
title: 后端设计
categories:
  - Leasing
date: 2019-07-26 20:45:04
---

### 文档说明
本文档是针对cvicse-leasing项目的后端设计与开发的说明文档，旨在介绍后端代码中的架构设计，代码设计，并介绍在后端开发中所使用的一些第三方模块的使用。希望通过本文档，能够帮助后端开发人员更好地理解我们的后端代码。

### 架构设计
首先介绍一下本项目后端整体设计架构，本项目后端采用的是微服务架构。
#### 1 什么是微服务
所谓微服务架构是一种架构模式，它的主要思想其实很简单，就是将一个功能繁杂的单一服务拆分成多个微小的服务，而拆分的标准就是服务本身的独立性。举一个简单的例子，试想我们为一个宠物店写一套服务，我们会提供针对宠物的管理服务，会提供针对宠物店员工的管理服务，还会提供针对顾客的管理服务，按照单一服务的实现，我们会在一个服务的每一层写多个类来实现针对不同类型的服务，而按照微服务的设计思想，我们可以将这一整套服务拆分成多个不同的服务，每个服务运行在其独立的进程中，服务和服务之间采用轻量级的通信机制相互沟通（通常是基于HTTP的Restful API)。而这个宠物店的服务设计可以参考 <https://github.com/spring-petclinic/spring-petclinic-microservices>。
微服务的架构模式在我看来有以下这些好处：
- 服务与服务之前独立，充分解耦。
- 当某个服务出现问题时，开发人员可以快速定位，快速解决，并可以做到不影响其他服务。 
- 服务完全独立测试、部署、升级、发布。

#### 2 微服务的管理
当采用微服务架构之后，就会需要进行微服务的管理工作，试想一个企业级的大型服务，可能动辄会有上百个微服务组成。本项目应用[Spring Cloud](https://spring.io/projects/spring-cloud "Spring Cloud")的一系列框架来对微服务进行管理，微服务是可以独立部署、水平扩展、独立访问的服务单元，Spring Cloud就是这些微服务的大管家，Spring Cloud中有很多非常好用的轮子，大家可以从Spring Cloud的[官方文档](https://cloud.spring.io/spring-cloud-static/Greenwich.SR2/multi/multi_spring-cloud.html)上面查看。
#### 3 本项目架构
本项目目前主要应用了Spring Cloud中的服务注册中心*Eureka*和*Api-GateWay*两个框架来管理微服务，可以先看一下本项目目前的微服务架构图。
![](http://cdn.njuics.cn/microservice.png)
##### 3.1 服务注册与发现
目前后端有两个服务，分别是流程管理服务和合同服务，首先介绍一下[Eureka](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.0.M1/)，Eureka是Netflix开源的一款提供服务注册和发现的产品，它提供了完整的Service Registry和Service Discovery实现。也是springcloud体系中最重要最核心的组件之一。简而言之，Eureka是所有微服务的注册中心，它本身也是一个独立的服务，其他所有的服务可以注册在Eureka上进行统一的管理。  
而Eureka的实现也很简单，首先需要构建一个Eureka Server。首先在pom.xml中添加依赖。
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
然后在启动代码application添加`@EnableEurekaServer`注解，表示该服务是一个Eureka Server。
```java
@SpringBootApplication
@EnableEurekaServer
public class LeasingEurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(LeasingEurekaApplication.class, args);
    }
}
```
最后需要在`application.yml`进行一些必要的配置，在项目的`resource`目录下有三个`.yml`文件，`application-dev.yml`表示本地运行时的配置文件，所以看该文件即可。
```yml
server:
  port: 8761 #服务的端口号
eureka:
  instance:
    hostname: localhost
  # standalone mode
  client:
    registerWithEureka: false   #表示是否将自己注册到Eureka Server，默认为true
    fetchRegistry: false     #表示是否从Eureka Server获取注册信息，默认为true
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/  #设置与Eureka Server交互的地址
```
主要看`第8行`和`第9行`，因为本服务是Eureka Server,所以需要将这两个属性设置为`false`。而其他需要注册到Eureka上的服务采用默认值即可。  
然后需要将其他微服务配置为Eureka CLient，下面以合同管理服务为例进行配置。  
同样先在`pom.xml`中添加依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-commons</artifactId>
    <version>2.1.2.RELEASE</version>
</dependency>
```
然后在启动代码application添加`@EnableDiscoveryClient`注解，表示该服务是一个Eureka Client。
```java
@SpringBootApplication
@EnableDiscoveryClient
public class LeasingBackendApplication {
	public static void main(String[] args) {
		SpringApplication.run(LeasingBackendApplication.class, args);
	}
}
```
最后在`application-dev.yml`中加入Eureka Server的地址
```yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```
需要注意的是，要先启动**eureka-service-discovery**服务，再启动其他服务。访问`localhost:8761`看一下。
![](http://cdn.njuics.cn/eureka.png)
这样简单的微服务注册与发现就基本实现了。

##### 3.2 Api-Gateway
本项目用Spring Cloud的[Api-GateWay](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.0.M1/)来实现路由的统一获取和转发，简单来说，因为不同的微服务需要运行在不同的端口，这样的话，前端访问不同的服务就要通过不同的端口进行访问，这样实际上是非常麻烦和不优雅的。而Api-GateWay能实现的最基本的功能就是让前端访问Api-GateWay的端口，然后在Api-Gateway内部进行统一的路由转发，将不同的访问分发给不同的服务，就像项目架构图中展示的那样，因此Api-Gateway本身也是一个微服务，需要将其注册到eureka上去。然后看一下具体的配置代码。
首先需要在`pom.xml`中加入必要的依赖:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```
然后在`application-dev.yml`中配置基本的路由
```yml
server:
  port: 8082
spring:
  application:
    name: spring-cloud-apiGateway
  cloud:
    gateway:
      routes:
        - id: activiti_route
          uri: http://localhost:8081/
          predicates:
            - Path=/api/processInstances/**
        - id: contract_route
          uri: http://localhost:8000/
          predicates:
            - Path=/api/contracts/**
        - id: t_route
          uri: http://localhost:8000/
          predicates:
            - Path=/api/templates/**
        - id: front_route
          uri: http://localhost:8000/
          predicates:
            - Path=/**
```
各字段含义如下:  
+ id：我们自定义的路由 ID，保持唯一
+ uri：目标服务地址
+ predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
+ filters：过滤规则，本示例暂时没用。

`**`表示完全匹配，所以前端访问`localhost:8082/api/processInstances`时，会转发到`localhost:8081/api/processInstances`，即流程管理服务的端口。这样就可以简单地实现路由的统一获取和内部转发。当然Api-Gateway的功能远不止此，它还能实现路由的筛选，过滤等功能。

##### 3.3 小结
本项目利用Spring Cloud提供的两个轮子Eureka和Api-Gateway简单地搭建了一个微服务架构，当然Spring Cloud家族有很多高效简单的工具，可以参考Spring官方的[宠物店例子](https://github.com/spring-petclinic/spring-petclinic-microservices)。

### 合同管理服务
合同模块采用[Spring Boot](https://spring.io/projects/spring-boot)+[JPA](https://docs.spring.io/spring-data/jpa/docs/2.1.9.RELEASE/reference/html/)+[MongoDB](https://docs.mongodb.com/)+[JaVers](https://javers.org/documentation/repository-examples/)实现。代码实现中需要对合同模板和合同这两种对象进行处理，但是这两种对象除了数据格式不同之外没有其他的区别，包括调用的接口也是完全相似的，所以在本文档中主要针对合同模块进行说明。
#### 1 数据层设计
合同模块没有采用传统的关系型数据库，例如MySQL数据库，而是采用了文档型数据库，主要是因为本服务数据主要是文档型的，不适合采用key-value型数据库，试想将动辄上万字的合同内容存在MySQL中是不合理的，而文档型数据库存储的数据格式类似JSON，如下图所示，增删改查的效率也更高。本服务采用MongoDB，它是一种典型的文档型数据库。
```
{
  id: 1
  content:[
    {
      title: "demo",
    },
    {
      content: "demo",
    }
  ]
}
```
#### 2 接口设计
所有后端接口全都采用[Restful](https://spring.io/guides/gs/rest-service/)风格，服务中采用[swagger-ui](https://www.baeldung.com/swagger-2-documentation-for-spring-rest-api)来实现API文档可视化，而它的配置也很简单，首先添加必要的依赖：
```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
<version>2.9.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```
然后在`config`目录下，添加`SwaggerConfig.java`:
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select().apis(RequestHandlerSelectors.any()).paths(PathSelectors.any()).build();
    }
}
```
这样就可以在`http://localhost:8000/swagger-ui.html#/`中查看所有的接口，包括需要的参数类型，返回值类型等，并可以测试接口。如图所示：
![](http://cdn.njuics.cn/swagger.png)
可以看到，除了基本的增删改查接口之外，还有一个获取合同历史版本的接口，而这个接口是依靠[JaVers](https://javers.org/documentation/repository-examples/)来实现的。

#### 3 版本控制
JaVers是一个轻量级，完全开源的Java库，用于追踪和记录数据中的更改。它可以配合repository使用，只需要添加一行注解`@JaversSpringDataAuditable`即可。
```java
@JaversSpringDataAuditable
public interface ContractRepository extends MongoRepository<Contract, String> {
    Contract findByContent(JSONObject content);
}
```
这样，当数据库中有数据发生修改时，JaVers就会自动追踪和记录数据库中发生的修改。仔细看一遍它的文档，会发现JaVers使用起来应该相对简单的，它只有`Shadows`, `Snapshots`,`Changes`这三个类型，这三个不同的类型在我看来主要是追踪记录数据的范围不同。`Changes`侧重记录变化，`Shadows`侧重获取实体变化，而`Snapshots`则无脑地将所有的记录信息全部给你。举例来看，在`Changes`能够获取的信息最少，它只会记录某个实体发生的变化，这是JaVers官方文档中使用`Changes`的打印结果：
```
Changes:
* changes on org.javers.core.examples.model.Person/bob :
  - 'name' changed from 'Robert Martin' to 'Robert C.'
  - 'position' changed from '' to 'Developer'
```
`Shadows`记录的东西会多一些，可以通过它获取发生变化的某个实体的所有版本。
```
Shadows:
Person{login='bob', name='Robert C.', position=Developer}
Person{login='bob', name='Robert Martin', position=null}
```
但这还是不够的，因为理想的状态是，我们希望做到类似GitHub一样，用户可以通过`contractId`+`commitId`这两个属性来获取具体某个版本的实体，幸好用`Snapshots`可以做到，因为它记录了`commitId`。
```
Snapshots:
Snapshot{commit:2.0, id:...Person/bob, version:2, (login:bob, name:Robert C., position:Developer)}
Snapshot{commit:1.0, id:...Person/bob, version:1, (login:bob, name:Robert Martin)}
```

这样我们就可以利用`Snapshots`的接口，拿到我们需要的版本号和合同内容，实现版本控制。来看一下代码：
```java
public Contract getContractWithJaversCommitId(String contractId, String commitId) throws ContractNotFoundException{
    Contract contract = this.getContract(contractId);  //根据合同id获取合同
    JqlQuery jqlQuery= QueryBuilder.byInstance(contract).build();  //根据合同实体构建jqlQUery
    List<CdoSnapshot> snapshots = javers.findSnapshots(jqlQuery);  //获取合同实体的所有snapshots
    for(CdoSnapshot snapshot:snapshots){
        if(snapshot.getCommitId().getMajorId() == Integer.parseInt(commitId))   //遍历snapshots，根据commitId返回对应版本的合同
            return JSON.parseObject(javers.getJsonConverter().toJson(snapshot.getState()),Contract.class);
    }
    return null;
}
```
至于对版本进行其他类型的控制管理，请详细阅读JaVers的[官方文档](https://javers.org/documentation)。


### 流程管理服务
流程在后端设计中是非常常见的，除了对数据的增删改查，可以说流程是另外一种非常常见的服务，在本项目中，我们将流程从其他的服务中完全抽离出来，单独构建一个流程管理服务来管理项目中可能涉及的所有流程。本服务主要使用[Activiti](https://www.activiti.org/)流程引擎来帮助我们构建流程管理服务。
#### 1 BPMN图
#### 2 接口说明
#### 3 具体设计