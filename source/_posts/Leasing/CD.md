---
title: 集成与持续发布
categories:
  - Leasing
date: 2019-07-26 20:45:04
---

## 项目部署部分

### 1. Docker镜像部署

通过`Dockerfile`来对 `spring` 构建之后的 `jar` 文件build 出对应的镜像，并且使用 `docker-compose` 来进行多个 `docker` 镜像的整合。来起到最终的微服务架构部署的目标。

在 `docker` 进行部署的过程中，使用过两个不同 `docker-compose` 的版本。现分别进行解释

- `2.0` ：当前版本下，首先需要使用命令 `docker build . -t <image_name>` 来通过该目录下的 `Dockerfile` 进行构建镜像 。示例如下

```dockerfile
FROM java
ADD ./cvicse.jar /root/cvicse.jar
ENV JAVA_OPTS=""
EXPOSE 8000
ENTRYPOINT ["java","-jar","/root/cvicse.jar"]
```

该简单的 `Dockerfile` 指明镜像内存在的 `java` 环境，并且通过 `ADD ./cvicse.jar /root/cvicse.jar ` 来把执行 jar 包添加至镜像文件内部；`EXPOSE 8000` 指明镜像内部提供端口为 **8000** ；  `ENTRYPOINT ["java","-jar","/root/cvicse.jar"]` 指明了镜像生成容器时执行的相关指令内容。在这里，即为简单的 jar 包运行

- `3.0` ：当前版本下，可以直接在 `docker-compose` 文件内部进行镜像的构建；

通过 

```
build:
      context: .
      dockerfile: Dockerfile
      args:
        - ARTIFACT_NAME=leasing-eureka-server-0.0.1-SNAPSHOT
        - EXPOSED_PORT=8761
```

如上的配置方式，以  `context` 作为基路径内容、将需要进行镜像构建的文件名称作为 `dockerfile` 字段内容，同时可以附加镜像构建的参数。通过如上配置，不需要额外进行镜像构建的脚本，而可以采取直接 `docker-compose up --build -d ` 来实现镜像构建和容器启动的目标



### 2. 微服务

项目分为如下几个微服务

| 名称                       | 职能                               |
| -------------------------- | ---------------------------------- |
| `leasing-discovery-server` | 提供eureka注册服务，监控各个微服务 |
| `leasing-api-gateway`      | 负责将请求进行微服务分发           |
| `leasing-backend`          | 负责处理基本需求的业务             |
| `leasing-activiti`         | 负责处理流程相关业务               |
| `leasing-frontend`         | 负责前端静态文件展示               |

以上是最为简单的微服务架构方式：由 `discovery-server` 提供各个微服务的注册服务，各个微服务之间通过内网进行通讯。此时负责不同业务的微服务可以选择是否共享同一个数据库源。而 `api-gateway` 作为提供对外网关接口和对内的服务转发，能够及时分配各个微服务的职能(当业务量增大时，需要考虑扩展网关，避免其成为性能瓶颈)

#### 2.1 Eureka注册服务

首先建立 `discovery-server`, 其程序入口如下所示。`@EnableEurekaServer`指明该微服务作为 Eureka而给予其他微服务进行注册。

```java
@SpringBootApplication
@EnableEurekaServer
public class LeasingEurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(LeasingEurekaApplication.class, args);
    }

}
```

此外，还通过如下配置文件的方式，提供将要进行的微服务进行连接使用

```xml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  # standalone mode
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

对应的，在注册微服务端，按照如下方式编写即可完成注册操作：

```java
...main
@SpringBootApplication
@EnableDiscoveryClient
public class LeasingBackendApplication {
	public static void main(String[] args) {
		SpringApplication.run(LeasingBackendApplication.class, args);
	}

}
...application-buildDocker.yml
eureka:
  instance:
    hostname: discovery-server
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:8761/eureka/
      
...
```

所有其他的微服务都需要注册 `Eureka` 服务，故运行时，需要先开启 `Eureka` 服务之后再开启其他的服务。

#### 2.2 网关配置

对外界来说，微服务架构下的系统本身可以看做是一个系统，那么如何将单一的一个请求信息分发到指定的微服务进行实现，则是网关 Api-Gateway 需要进行配置的工作。

现配置如下：

在这里，将请求uri 前缀为

- `/api/processInstances/**` 分配给 `leasing-activiti` 微服务, 即 `localhost:8081`
- `/api/contracts/**` 和 `/api/tempaltes/**` 分配给 `leasing-backend` 微服务，即 `localhost:8000`
- `/**` 分配给 `leasing-frontend`微服务，作为静态文件的入口。

注意：如果不同的分配规则中出现了路由重复的现象，那么就采取贪心策略

```
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
          uri: http://localhost:9000/
          predicates:
            - Path=/**
```

### 3.前后端一体化打包实现

传统的Web项目前端部分会通过 `webpack` 构建的方式，根据已有的代码得到静态文件，并且委托 nginx 这类的代理工具来进行项目的部署工作。这里将前端 `umi` 打包后的静态文件直接作为 `spring-boot` 项目的静态文件，而后打包形成一个整体的 `jar` 包，通过运行 jar 包来直接访问前端内容，并且可以在相同的端口下进行后端请求发送。

主要的逻辑代码如下

- 主 `maven` 给定项目基路径、项目构建顺序

```xml
					    <modules>
        				<module>client</module>
        				<module>server</module>
							</modules>
	....
							<plugin>
                <groupId>org.commonjava.maven.plugins</groupId>
                <artifactId>directory-maven-plugin</artifactId>
                <version>0.1</version>
                <executions>
                    <execution>
                        <id>directories</id>
                        <goals>
                            <goal>directory-of</goal>
                        </goals>
                        <phase>initialize</phase>
                        <configuration>
                            <property>leasing.basedir</property>
                            <project>
                                <groupId>com.cvicse</groupId>
                                <artifactId>leasing-frontend</artifactId>
                            </project>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

`directory-maven-plugin` 该插件给出了项目的基路径 `leasing.basedir` ，并且能够直接由子maven继承并且进行引用。

通过 `module` 的顺序，来指定：先进行 client 子 maven 的构建，之后进行 server 子 maven 的构建

- server 子 maven 设计

```xml
		<plugin>
				<artifactId>maven-resources-plugin</artifactId>
				<version>3.1.0</version>
				<executions>
					<execution>
						<id>position-react-build</id>
						<goals>
							<goal>copy-resources</goal>
						</goals>
						<phase>prepare-package</phase>
						<configuration>
							<outputDirectory>${project.basedir}/target/classes/public</outputDirectory>
							<resources>
								<resource>
									<!--静态资源源目录-->
									<directory>${leasing.basedir}/client/dist</directory>
									<filtering>false</filtering>
								</resource>
							</resources>
						</configuration>
					</execution>
				</executions>
			</plugin>
```

在 `prepare-package` 阶段进行资源内容的拷贝，由 `maven-resources-plugin` 该插件进行资源复制操作。其中资源源路径为 `${leasing.basedir}/client/dist` ，目标路径为`${project.basedir}/target/classes/public`

- client 子 maven 设计

```xml
 <plugin>
        <groupId>com.github.eirslett</groupId>
        <artifactId>frontend-maven-plugin</artifactId>
        <version>1.7.6</version>
        <configuration>
          <workingDirectory>${project.basedir}</workingDirectory>
          <installDirectory>${project.basedir}</installDirectory>
          <nodeVersion>v8.9.4</nodeVersion>
          <npmVersion>5.6.0</npmVersion>
        </configuration>
        <executions>
          <execution>
            <phase>prepare-package</phase>
            <id>install node and npm</id>
            <goals>
              <goal>install-node-and-npm</goal>
            </goals>
          </execution>
          <execution>
            <id>npm install</id>
            <goals>
              <goal>npm</goal>
            </goals>
            <!-- optional: default phase is "generate-resources" -->
            <phase>prepare-package</phase>
            <configuration>
              <!-- optional: The default argument is actually
               "install", so unless you need to run some other npm command,
               you can remove this whole <configuration> section.
               -->
              <arguments>install</arguments>
            </configuration>
          </execution>
          <execution>
            <id>npm run build</id>
            <phase>prepare-package</phase>
            <goals>
              <goal>npm</goal>
            </goals>
            <configuration>
              <arguments>run build</arguments>
            </configuration>
          </execution>
        </executions>
      </plugin>
```

`frontend-maven-plugin` 该插件实现了在 maven 构建过程中执行 `npm` 相关的指令。在如上的设计中，以 `npm install` 和 `npm run build ` 分别进行依赖的安装和构建，并且在 `prepare-package`阶段执行这些指令

### 4.微服务部署实现

当前版本 ： https://github.com/nemoworks/leasing/releases/tag/0.0.3-20190803

具体部署步骤如下

1. 本地打包、资源上传 `mvn clean package -PbuildDcoker && bash /bash/scp.sh -h <ip>`

前一条指令作为maven 的构建，其中设定环境变量为 `buildDocker` ; 该过程可以为每一个 (当前一共为5个)微服务生成一个 jar 包。

后一条指令进行资源的远程拷贝，具体的bash指令 如下

```sh
# h is the host name

while getopts h: option
do
case "${option}"
in
h) HOST=${OPTARG};;
esac
done


scp leasing-activiti/target/leasing-activiti-0.0.1-SNAPSHOT.jar root@$HOST:/root/cvicse/leasing-activiti-0.0.1-SNAPSHOT.jar
scp leasing-api-gateway/target/leasing-api-gateway-0.0.1-SNAPSHOT.jar root@$HOST:/root/cvicse/leasing-api-gateway-0.0.1-SNAPSHOT.jar
scp leasing-backend/target/leasing-backend-0.0.1-SNAPSHOT.jar root@$HOST:/root/cvicse/leasing-backend-0.0.1-SNAPSHOT.jar
scp leasing-eureka-server/target/leasing-eureka-server-0.0.1-SNAPSHOT.jar root@$HOST:/root/cvicse/leasing-eureka-server-0.0.1-SNAPSHOT.jar
scp leasing-frontend/server/target/leasing-frontend-server-0.0.1-SNAPSHOT.jar root@$HOST:/root/cvicse/leasing-frontend-server-0.0.1-SNAPSHOT.jar
scp bash/buildDocker.sh root@$HOST:/root/cvicse/buildDocker.sh
scp docker-compose.yml root@$HOST:/root/cvicse/docker-compose.yml
scp setup.sh root@$HOST:/root/cvicse/setup.sh
scp docker/Dockerfile root@$HOST:/root/cvicse/Dockerfile
scp docker/dockerize.tar.gz root@$HOST:/root/cvicse/dockerize.tar.gz
```

如上指令将所有的jar包，远端部署需要的 docker 文件和脚本文件均复制到远端的 `/root/cvicse` 目录下。

2. 远端镜像构建与容器创建

在第一步正常执行之后，通过ssh 远程连接，进入 `/root/cvicse` 目录下，进行 `bash setup.sh`来进行镜像的构建。具体如下

```bash
docker-compose down
docker rmi $(docker images | grep "leasing/" | awk '{print $3}')
docker rmi $(docker images | grep "none" | awk '{print $3}')
docker network create backend
docker network create proxy
docker-compose up --build -d && docker-compose logs -f
```

- 1 - 3 行进行了原有环境中镜像的清理工作
- 4，5 行新建了两个docker network : `backend` 和 `proxy`。
  - `backend` 网络作为微服务的内网环境，仅由 `leasing-api-gateway` 来对外暴露端口
  - `proxy` 供 `nginx`镜像反向代理使用。通过 `nginx` 的动态域名添加和证书添加，能够让同样在 `proxy` 网络下的服务自动进行域名解析和证书的支持。
- 第 6 行进行镜像的构建，具体构建方式参见 `1. Docker镜像部署`

