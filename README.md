# 微服务容器化改造

## 改造前的微服务

- [auth-service](https://github.com/xdevops-caj-lab-cloudnative-tk/PiggyMetrics/tree/master/auth-service)


## 设置环境变量

认证服务使用以下的环境变量作为微服务连接认证服务的client credentials：
- NOTIFICATION_SERVICE_PASSWORD
- STATISTICS_SERVICE_PASSWORD
- ACCOUNT_SERVICE_PASSWORD

Set env vars for bash:

```bash
echo "export NOTIFICATION_SERVICE_PASSWORD=notification" >> $HOME/.bashrc
echo "export STATISTICS_SERVICE_PASSWORD=statistics" >> $HOME/.bashrc
echo "export ACCOUNT_SERVICE_PASSWORD=account" >> $HOME/.bashrc

source $HOME/.bashrc

```

Set env vars for zsh

```bash
echo "export NOTIFICATION_SERVICE_PASSWORD=notification" >> $HOME/.zshrc
echo "export STATISTICS_SERVICE_PASSWORD=statistics" >> $HOME/.zshrc
echo "export ACCOUNT_SERVICE_PASSWORD=account" >> $HOME/.zshrc

source $HOME/.zshrc

```


Verify
```bash
printenv | grep PASSWORD
```


## 每个微服务有自己的代码仓库

将原来微服务属于一个Maven module改为每个微服务有自己的代码仓库。

添加`.gitignore`文件，以忽略一些不需要提交到代码仓库的文件。

在pom.xml中将parent直接改为spring-boot-starter-parent，并引入spring-cloud-dependencies。

```xml
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.3.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
		<java.version>1.8</java.version>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
```

指定groupId:

```xml
<groupId>com.piggymetrics</groupId>
```

## 本地构建运行

```bash
mvn clean install && mvn spring-boot:run
```

此时发现报错，因此程序试图从Spring Cloud Config Server拉取应用配置。

## 去除从Spring Cloud Config Server拉取配置

将`src/main/resources`下的bootstrap.yml改为application.yaml。

并移除其中`spring.cloud.config`的内容。

同时在pom.xml文件中去除对应的依赖：
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```

再次本地构建运行。

发现程序可以启动成功，但是报错无法注册到Eureka Server。

## 去除注册到Eureka Server

修改Spring Boot应用类AuthApplication.java，将`@EnableDiscoveryClient`注解去掉，并去掉对应的`import`语句。

同时在pom.xml文件中去除对应的依赖：
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```
或
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
```

再次本地构建运行。

发现程序可以启动成功，但是报错无法连接到数据库（本例子数据库为MongoDB）。

在解决连接到MongoDB数据库问题之前，先检查一下剩余的Spring Cloud依赖。

## 剩余的Spring Cloud依赖

因为本服务中使用了`spring-cloud-starter-oauth2`和`spring-cloud-starter-sleuth`，因此需要保留这两个Spring Cloud依赖。

如果程序不再依赖任何Sprng Cloud组建，则可以去除全部Spring Cloud依赖，并再次本地构建运行。

## 配置上下文路径

在原来的Spring Cloud Config Server中定义了auth-service的上下文路径为`/uaa`，因此需要在application.yaml中配置上下文路径：

```yaml
server:
  servlet:
    context-path: /uaa
```

再次本地构建运行。

## 在本地运行MongoDB

参见[mongodb](https://github.com/xdevops-caj-lab-cloudnative-tk-msa/mongodb) 来创建MongoDB数据库。

## 配置连接MongoDB数据库

在原来的Spring Cloud Config Server中定义了auth-service的MongoDB数据库连接，因此需要在application.yaml中配置MongoDB数据库连接：

```yaml
spring:
  data:
    mongodb:
      host: auth-mongodb
      username: user
      password: ${MONGODB_PASSWORD}
      database: piggymetrics
      port: 27017
```

其中`auth-mongodb`是MongoDB数据库的服务名。


对MongoDB的连接配置作如下修改，并放到application.yaml中：

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://user:password@auth-mongodb:27017/authdb}
```

说明：
- `spring.data.mongodb.uri`属性指定了MongoDB的连接URI，如果没有指定，则使用默认值`mongodb://user:password@auth-mongodb:27017/authdb`。
- TODO 经测试，在本例中只有采用`spring.data.mongodb.uri`的方式程序才能正常连接上MongoDB数据库，如果采用`spring.data.mongodb.host`、`spring.data.mongodb.username`、`spring.data.mongodb.password`、`spring.data.mongodb.database`、`spring.data.mongodb.port`的方式，则程序无法连接上MongoDB数据库。


再次本地构建运行。

发现程序可以启动成功。

## 本地调试运行

添加一个application-local.yaml文件，用于本地调试。

指定本地调试时的端口：
```yaml
server:
  port: 5000
```

本地调试运行：
```bash
mvn spring-boot:run -Dspring.profiles.active=local
```

## 构建容器镜像

创建Dockerfile文件：

```bash
FROM registry.access.redhat.com/ubi8/openjdk-8

COPY target/*.jar /opt/app.jar

CMD java -jar /opt/app.jar

EXPOSE 8080
```

构建容器镜像：

```bash
podman build -t auth-service .
```

推送容器镜像到镜像仓库：

```bash
podman login quay.io
# replace as your repository
podman tag auth-service:latest quay.io/xxx/auth-service:latest
podman push quay.io/xxx/auth-service:latest
```

## 部署到Kubernetes

在piggymeticrs-config仓库中集中管理多个微服务的Kubernetes YAML。

### 部署MongoDB

TODO 使用Helm部署bitnami/mongodb

### 部署应用

TODO Kubernetes deployment, confimap, secret, service







