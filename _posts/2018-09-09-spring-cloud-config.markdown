---
layout: post
title: Spring Cloud学习笔记(一):Spring Cloud Config
subtitle: Spring Cloud Config配置使用
date: 2018-09-09T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Spring Cloud


---



> 滚动阅读全文 

## 前言
将应用程序的配置信息直接写入代码通常是有问题的，因为每次对配置进行更改时，应用程序都必须重新编译和重新部署。为了避免这种情况，开发人员会将配置信息与应用程序代码完全分离。现已有许多优秀的开源的配置管理解决方案，如Etcd,Consul,ZooKeeper等。Spring Cloud Config提供不同后端支持的通用配置管理解决方案。它可以将Git、Eureka和Consul作为后端进行整合。  

接下来的示例中将包括以下内容：
1. 搭建一个基于文件系统存储的配置服务器
2. 基于Git的配置服务器
3. 读取配置信息
4. 刷新配置信息

## 搭建Speing Cloud 配置服务器

### 1、新建项目并添加依赖
新建一个Spring Boot项目(此处命名为<code style="color:red">cloud-config-server</code>)并引入如下依赖

```
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>
		<dependency>
                       <groupId>org.springframework.cloud</groupId>
                       <artifactId>spring-cloud-starter-config</artifactId>
                </dependency>
```

### 2、修改配置文件
修改<code style="color:red">application.yml</code>文件

```
server:
# 配置服务器监听窗口
  port: 8888
spring:
  profiles:
# 用于存储配置的后端存储库（文件系统）
    active: native
  cloud:
    config:
      server:
        native:
# 配置文件存储位置的路径
          search-locations: classpath:config/
```
### 3、创建Spring Cloud Config引导类

在启动类中增加注解<code style="color:red">@EnableConfigServer</code>,激活配置服务

```
@SpringBootApplication
@EnableConfigServer
public class CloudConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(CloudConfigServerApplication.class, args);
	}
}


```
经过以上简单注解及配置便完成了一个Spring Cloud Config服务

### 4、测试

为了测试我们的配置管理服务是否搭建成功，我们建立两个配置文件<code style="color:red">study-dev.yml</code>和<code style="color:red">study-prod.yml</code>（也可以是.properties文件）放在配置服务器的<code style="color:red">src/main/resource/config</code>目录下。


study-dev.yml
```
test:
  value: dev
  
```
study-dev.yml

```
test:
  value: prod
```
注：应用程序配置文件的命名约定为“<font style="color:red">应用程序名称-环境名称</font>”  ，这样的话我们只需在应用程序的<code style="color:red">application.yml</code>文件中设置<code style="color:red">spring.profile.active</code>的值为所选环境名称，程序会自动加载不同配置文件。例如设置置<code style="color:red">spring.profile.active=dev</code>程序加载的就是study-dev.yml文件



执行<code style="color:red">mvn spring-boot:run</code>启动配置服务。

访问<code style="color:red">localhost:8888/study/default</code>,可以看到<code style="color:red">study.yml</code>文件中的信息以json格式返回。  
访问<code style="color:red">localhost:8888/study/dev</code>,可以看到<code style="color:red">study-dev.yml</code>文件中的信息。  
访问<code style="color:red">localhost:8888/study/prod</code>,可以看到<code style="color:red">study-prod.yml</code>文件中的信息。  


## 基于Git的Spring Cloud配置服务器
我们可以将配置文件放到git服务器中进行管理，只需将配置文件放到git库中，并修改配置服务器的application.yml文件即可

```
server:
# 配置服务器监听窗口
  port: 8888
spring: 
  cloud:
    config:
      server:
        git:
          # 配置Git存储库的URL
          uri: https://github.com/zhenxuexu/config-repo
          # 查找配置的路径,多个文件夹用","分割
          search-locations: study
          username: xxx
          password: xxxx
```

## 读取配置信息
接下来我们要在另一个应用程序中读取我们存储在配置服务器中的配置信息

### 1、新建项目并引入依赖
新建一个Spring boot项目为Cloud-study，添加依赖

```
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-config-client</artifactId>
	</dependency>
```
### 2、修改配置
新建<code style="color:red">bootstrap.yml</code>文件，添加配置信息

```
spring:
  application:
  # 设置应用程序的名称
    name: study
  # 设置激活的环境，dev 将加载study-dev.yml文件
  profiles:
    active: dev
# config server 配置
  cloud:
    config: 
      enabled: true 
      uri: http://localhost:8888
```

注：<code style="color:red">bootstrap.yml</code>文件优先加载与<code style="color:red">application.yml</code>文件，且<code style="color:red">bootstrap.yml</code>文件中的属性值不会被其他配置文件值覆盖，故一般将应用程序名称、应用程序profile和连接到Spring Cloud Config服务器的URI放在<code style="color:red">bootstrap.yml</code>文件中

### 3、获取配置属性

我们通过<code style="color:red">@Value</code>注解获取相关属性

```
@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class CloudStudyApplicationTests {

	@Test
	public void contextLoads() {
	}
	
	@Value("${test.value}")
	private String value;

	@Test
	public void testConfig() {
	    log.info(value);
	}
	
}

```

## 刷新配置
如何在属性变化是动态刷新应用程序？  
Spring Boot Actuator提供了<code style="color:red">@RefreshScope</code>注解，允许开发团队访问/refresh端点，这会强制Spring boot应用程序重新读取应用程序配置。

为了可以刷新配置，我们需要引入actuator依赖

```
	    <dependency>
	    	<groupId>org.springframework.boot</groupId>
	    	<artifactId>spring-boot-starter-actuator</artifactId>
	    </dependency>
```
注解使用，需要给加载变量的类上面加载<code style="color:red">@RefreshScope</code>
```
@RunWith(SpringRunner.class)
@SpringBootTest
@RefreshScope
@Slf4j
public class CloudStudyApplicationTests {

	@Test
	public void contextLoads() {
	}
	
	@Value("${test.value}")
	private String value;

	@Test
	public void testConfig() {
	    log.info(value);
	}
	
}

```
对<code style="color:red">localhost:8080/refresh</code>发送一个POST请求，便会刷新带有<code style="color:red">@RefreshScope</code>注解类中的变量信息。

如果有多个服务，需要多次执行/refresh也是比较繁琐的,github的webhook可以用来只要提交代码就自动调用客户端来更新。



