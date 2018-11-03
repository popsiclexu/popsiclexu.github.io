---
layout: post
title: Spring Boot——配置不同环境的属性文件
subtitle: 应用profile改变环境配置
date: 2018-09-09T00:00:00.000Z
author: Xu Zhenxue
header-img: img/post-bg-universe.jpg
catalog: true
tags:
  - Spring Boot
  - Java
catalog: true
---
> 滚动阅读全文

#### 前言
在应用开发过程中，不同开发环境中，应用的配置各不相同，比如开发环境、测试环境和生产环境中的数据库连接等信息。当我们使用单个配置文件配置不同环境中的参数时，需要频繁的改变配置文件较为麻烦。在Spring中我们可以给不同环境配置不同配置文件，运行时只需配置profile参数便可切换配置文件 

#### 一、不同环境配置文件
可以将一些固定不变的配置信息设置在该文件中，例如：mybatis的相关信息。其他根据环境变化而改变的配置放在其他文件中。

我们可以通过改变<code>application.yml</code>(也可以是<code>.properties</code>文件) 文件中的<code>spring.profiles.active</code>的值来改变程序运行环境，并加载相应配置文件。 
1. 默认配置文件--  application.yml  

```
mybatis:
  type-aliases-package: com.facets.core.entity
spring: 
  profiles:
    active:
    - dev
```

2. 开发环境配置文件-- application-dev.yml  

配置开发环境配置信息

```
# 数据源信息
spring: 
  datasource:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://192.168.102.60:5432/dev
    username: postgres
    data-password: 123456
#其他信息
com: 
  xzx: 
    username: xzx
    say: This is dev-environment!
```
3. 生产环境配置文件-- application-prod.yml  

配置生产环境配置信息


```
# 数据源信息
spring: 
  datasource:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://192.168.102.60:5432/dev
    username: postgres
    data-password: 123456
# 其他源信息
com: 
  xzx: 
    username: xzx
    say: This is prod-environment!
```
#### 二、加载配置文件
我们定义一个User实体类，用来接受配置文件的值。
1. 定义实体类

User.java
```
package com.facets.core.entity;

public class User {
	private String username;
	private String say;
	
	public String getUsername() {
		return username;
	}
	public void setUsername(String username) {
		this.username = username;
	}
	public String getSay() {
		return say;
	}
	public void setSay(String say) {
		this.say = say;
	}
	
}

```
2. 定义配置类 
通过配置类加载属性信息，并为User赋值

UserConfig.java
```
package com.facets.core.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.facets.core.entity.User;

@Configuration
public class UserConfig {
	@Bean
	// 加载配置信息中的username和say属性赋值给User对象
	@ConfigurationProperties(prefix="com.xzx")  
	public User getUser() {
		return new User();
	}
}

```
也可以将上述两种方法合并为一个类

UserPropertityConfig.java

```
package com.facets.core.entity;

@Component
@ConfigurationProperties(prefix="com.xzx")  
public class User {
	private String username;
	private String say;
	
	public String getUsername() {
		return username;
	}
	public void setUsername(String username) {
		this.username = username;
	}
	public String getSay() {
		return say;
	}
	public void setSay(String say) {
		this.say = say;
	}
	
}
```


#### 三、 测试

FacetsApplication.java
```
package com.facets.core;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.facets.core.entity.User;

@SpringBootApplication
@RestController
public class FacetsApplication {

	@Autowired
	private User user;
	
	@RequestMapping("/test")
	public User getUser() {
		return user;
	}
	public static void main(String[] args) {
		SpringApplication.run(FacetsApplication.class, args);
	}
}

```

设置<code>application.xml</code>中的spring.profiles.active为dev，得到如下输出

```
{"username":"xzx","say":"This is dev-environment!"}
```
设置<code>application.xml</code>中的spring.profiles.active为prod，得到如下输出


```
{"username":"xzx","say":"This is prod-environment!"}
```

### 后续
在这篇博客中介绍了如何使用不同配置文件，后面将会介绍Spring Boot读取属性文件内容的多种方法。