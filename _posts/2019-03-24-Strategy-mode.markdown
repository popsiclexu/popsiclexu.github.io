---
layout: post
title: 利用反射+策略模式优化过多的if else 代码
subtitle: Reducing "if else" code with Java Reflect and Strategy design mode 
date: 2019-03-24T00:00:00.000Z
author: Xu Zhenxue
header-style: text
catalog: true
tags:
  - Java
  - Spring Boot
  - 设计模式
---
> 滚动阅读全文

## 前言
最近刚看完《设计模式之禅》，在写代码前总是想着能不能尝试用上一些设计模式。前几天看到一篇公众号推文利用策略模式来优化过多的if else代码，正好符合目前我面临的一个场景，作者使用一个枚举类来维护所有的策略，这样的话，没增加一个策略，都要去枚举类里增加相应的枚举常量，不太符合“开闭原则”。同时，随着策略的增加，这个枚举类源码的理解性也会变得越来差，也比较难维护。我通过注解+反射和策略模式做了重构，这样一来增加一个策略之后，只需要放到被扫描的包下，就会被自动识别和使用，比枚举的方式更具扩展性，也更加的符合“开闭原则”。设计模式相关的资料网上很多，我就不再介绍什么是策略模式,直接介绍如何实现。

## 场景描述
平时大家可能都会写这样的代码,如果条件过多的话逻辑就比较混乱，也容易出错。比如我面临的场景：需要访问数十个API接口，对每个API接口返回的数据做不同的处理，如果按如下方式实现的话，我就要写数十个if else，增加一个API时又要来增加一个if else 即难以维护，阅读性也很差。
```java
if(a){
    
}else if(b){
    
}else if(c){
    
}else{
    
}
```

## 策略模式优化
优化的大致思想为：首先把每个if else里逻辑抽取到各实现类中作为策略，并使用自定义注解给每个策略命名；然后，通过反射机制扫描使用该注解的所有类，放入锦囊（Map）中；最后，根据策略名(key)，取出相应的策略执行相应的逻辑。
#### 策略实现
1. 注解实现

```java
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;


@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface DataSourceType {
    // 策略名，在我的场景中是API的类型
	String value() default "";
}

```
2. 策略抽象类

```java
import java.util.Map;

public interface DataSourceStrategy {
    // 每个策略是逻辑实现
	Map<String,Object> connect(Map<String,String> params);
	

}
```

3. 策略实现类

```java
import com.uniplore.restdata.core.datasource.DataSourceStrategy;
import com.uniplore.restdata.core.datasource.DataSourceType;
import org.springframework.stereotype.Service;

import java.util.Map;


@Service
@DataSourceType("aliPay")
public class AliPayDS implements DataSourceStrategy {
	@Override
	public Map<String, Object> connect(Map<String, String> params) {
		System.out.println("Ali API");
		return null;
	}

}
```

#### 反射实现锦囊
利用Java反射机制扫描指定包下的所有策略类，将策略名与策略类映射放到容器(ALL_DATA_SOURCE)中，该容器也可理解为收纳策略的锦囊。我使用的是SpringBoot，所以在该类上实现ApplicationRunner接口，在SpringBoot启动完成后会自动执行run方法。
```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

import java.io.File;
import java.net.URL;
import java.util.HashMap;

@Component
public class AllDataSourceType implements ApplicationRunner {
    // 策略容器
	private final HashMap<String,Class> ALL_DATA_SOURCE = new HashMap<>(100);

	public  Class getDS(String dsType){
		return ALL_DATA_SOURCE.get(dsType);
	}

	public  void initAllDS(String packageName){
		URL url = this.getClass().getClassLoader().getResource(packageName.replace(".","/"));
		File dir = new File(url.getFile());
		for (File file : dir.listFiles()){
			if (file.isDirectory()){
			    // 递归扫描子目录
				initAllDS(packageName);
			}else {
				String clazzName = packageName + "." + file.getName().replace(".class","");
				try {
					Class<?> clazz = Class.forName(clazzName);
					if (clazz.isAnnotationPresent(DataSourceType.class)){
					    // 获取策略名
						DataSourceType annotation = clazz.getAnnotation(DataSourceType.class);
						// 将扫描到的策略放入容器
						ALL_DATA_SOURCE.put(annotation.value(),clazz);
					}
				} catch (ClassNotFoundException e) {
					e.printStackTrace();
					continue;
				}
			}
		}

	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		this.initAllDS("com.uniplore.restdata.core.datasource.impl");
	}
}
```
#### 获取策略实例
因为每个策略都加上了@Service注解，故在Spring启动时已经实例化放到IOC容器中，我们只需要通过ApplicationContext获取到相应实例即可。
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;

@Component
public class DataSourceContext {
	@Autowired
	private ApplicationContext applicationContext;

    // 注入锦囊
	@Autowired
	private AllDataSourceType allDataSourceType;
	
	public DataSourceStrategy getStrategyInstance(String dsType) throws ClassNotFoundException {
		Class<?> clazz = allDataSourceType.getDS(dsType);
		// 获取策略实例
		DataSourceStrategy strategyInstance = ((DataSourceStrategy) applicationContext.getBean(clazz));
		return strategyInstance;
	}
}

```
#### 使用策略

```java
import com.uniplore.restdata.core.datasource.DataSourceContext;
import com.uniplore.restdata.core.datasource.DataSourceStrategy;
import com.uniplore.restdata.core.model.ReturnT;
import com.uniplore.restdata.service.DataSourceService;
import lombok.extern.log4j.Log4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
@Log4j
public class DataSourceServiceImpl implements DataSourceService {

	@Autowired
	private DataSourceContext dataSourceContext;


	@Override
	public ReturnT<Map<String,Object>> connect(String dsType, Map<String, String> params) {
		Map<String,Object> resultMap;
		try {
		    // 获取策略实例
			DataSourceStrategy strategyInstance = dataSourceContext.getStrategyInstance(dsType);
			// 使用策略
			resultMap = strategyInstance.connect(params);
		} catch (ClassNotFoundException e) {
//			log.error(e.getStackTrace());
			return new ReturnT<>(ReturnT.FAIL_CODE, "Exception:" + e.getMessage());
		}
		return new ReturnT<>(resultMap);
	}
}

```
#### 优化前后对比 
1. 优化前，我们的业务逻辑是下面这样，随着API数量的增加，if else也要随着增加

```java
public void testNoStrategy(String dsType){
		if (dsType.equals("BaZhuaYu")){
			BaZhuaYuDS baZhuaYuDS = new BaZhuaYuDS();
			baZhuaYuDS.connect(null);
		}else if (dsType.equals("AliPay")){
			AliPayDS aliPayDS = new AliPayDS();
			aliPayDS.connect(null);
		}else if (dsType.equals("")){

		}
	}
```

2. 优化后，无论增加多少个API我们都不用改变使用场景的源码，只需要实现相应的策略就行

```java
public void testStrategy(String dsType){
		System.out.println(dataSourceService.connect(dsType, null));
	}

```
