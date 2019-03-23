---
layout: post
title: Spring Boot——读取属性文件的多种方法
subtitle: Spring中的属性处理
date: 2018-09-09T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Spring Boot
  - Java
catalog: true
---
> 滚动阅读全文

#### 前言
在程序开发中，为了减少程序中的“硬编码”，我们通常会将一些信息放在属性文件中，再读取到程序中。Spring 提供了多种便利的方法帮助我们从属性文件中读取数据。

#### 一、属性文件加载
程序默认加载application.yml(或application.properties)属性文件。如需要加载其它属性文件使用<code style='color:red'>@PropertySource</code>注解:  
例如: 加载user.properties文件

```java
@PropertySource("user.properties")
```
加载完属性文件后我们可以通过以下集中方式获取属性设置的值
#### 二、读取属性值的方式
##### 1、Environment检索属性

```java
@Autowired
	private Environment env;
	@Bean
	@Qualifier("env")
	public User getUserByEnv() {
		User user = new User();
		// 获取属性值
		user.setUsername(env.getProperty("com.xzx.username"));
		user.setSay(env.getProperty("com.xzx.say"));
		return user ;
	}
```
Environment类对属性的操作有多种方法，比如对null属性赋予默认值，按指定数据类型读取属性，检查属性是否存在等；具体可以详细了解Environmentment类

##### 2、@ConfigurationProperties方式
使用@configuration注解可以读取文件中的属性，根据属性key自动为Bean赋值；如下我们注入一个User bean并通过该注解为其属性赋值。

注：User bean的属性名与文件中的属性名一致；
```java
@Bean
	@Qualifier("conPro")
	@ConfigurationProperties(prefix="com.xzx")
	public User getUserByCP() {
		return new User();
	}
```
User.java

```java
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

##### 3、@Value方式

我们还可以使用属性占位符的方式获取属性值；
```java
@Value("${com.xzx.username}")
	private String username;
	
	@Value("${com.xzx.say}")
	private String say;
	
	@Bean
	@Qualifier("value")
	public User getUserByValue() {
		User user = new User();
		user.setUsername(username);
		user.setSay(say);
		return user ;
	}
```

以上三种方式合并文件为

```java
package com.facets.core.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;

import com.facets.core.entity.User;

@Configuration
// 默认加载application属性文件
@PropertySource(value="user.properties")
public class UserConfig {
	@Autowired
	private Environment env;
	@Bean
	@Qualifier("env")
	public User getUserByEnv() {
		User user = new User();
		user.setUsername(env.getProperty("com.xzx.username"));
		user.setSay(env.getProperty("com.xzx.say"));
		return user ;
	}
	
	@Bean
	@Qualifier("conPro")
	@ConfigurationProperties(prefix="com.xzx")
	public User getUserByCP() {
		return new User();
	}
	
	@Value("${com.xzx.username}")
	private String username;
	
	@Value("${com.xzx.say}")
	private String say;
	
	@Bean
	@Qualifier("value")
	public User getUserByValue() {
		User user = new User();
		user.setUsername(username);
		user.setSay(say);
		return user ;
	}
}

```


### 三、测试
属性文件user.properties

```java
com.xzx.username=xzx
com.xzx.say=hello worldFacets
```



FacetsApplication.java
```java
package com.facets.core;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.facets.core.entity.User;

@SpringBootApplication(exclude=DataSourceAutoConfiguration.class)
@RestController
public class FacetsApplication {

	@Autowired
	@Qualifier("env")
	private User user;
	
	@Autowired
	@Qualifier("conPro")
	private User user1;
	
	@Autowired
	@Qualifier("value")
	private User user2;
	
	@RequestMapping("/test")
	public List<User> getUser() {
		List<User> list = new ArrayList<>();
		list.add(user);
		list.add(user1);
		list.add(user2);
		return list;
	}
	public static void main(String[] args) {
		SpringApplication.run(FacetsApplication.class, args);
	}
}

```
输出：

```json
[{"username":"xzx","say":"hello world"},{"username":"xzx","say":"hello world"},{"username":"xzx","say":"hello world"}]
```
