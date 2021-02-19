---
title: 从源码解析SpringBoot
date: 2021-2-8 15:28:19
---

> 学习sb源码也有一段时间了，牛年立一个flag，写一个sb源码的系列文章。顺祝牛年春节快乐。

> 主要素材来源于慕课网上的视频教程[《全方位深入解析最新版SpringBoot源码》](https://coding.imooc.com/learn/list/404.html/)


# 一、系统初始化器
## 1.1 系统初始化器demo

```
public class FirstInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    public void initialize(ConfigurableApplicationContext configurableApplicationContext) {
        ConfigurableEnvironment environment = configurableApplicationContext.getEnvironment();
        Map<String,Object> map = new HashMap<String,Object>();
        map.put("key1","value1");
        MapPropertySource mapPropertySource = new MapPropertySource("firstInitializer",map);
        environment.getPropertySources().addLast(mapPropertySource);
        System.out.println("run FirstInitializer");
    }
```
## 1.2 系统初始化器用法
### 1.2.1 实现方式一
- 实现ApplicationContextInitializer接口
- spring.factories内填写实现
- key值为org.springframework.context.ApplicationContextInitializer

### 1.2.2 实现方式二
- 实现实现ApplicationContextInitializer接口
- SpringApplication类初始化后设置进去

### 1.2.3 实现方式三
- 实现ApplicationContextInitializer接口
- application.properties内填写接口实现
- key值为context.initializer.classes

## 系统初始化器作用
## 源码剖析

```
//SpringApplication构造函数中配置系统初始化器
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        //省略...
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //省略...
    }
    
public ConfigurableApplicationContext run(String... args) {
        //省略...
            //在刷新方法之前，遍历调用initializers的initialize方法
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
        //省略...  
}
```











