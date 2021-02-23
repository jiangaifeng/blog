---
title: 从源码解析SpringBoot-总目录
date: 2021-2-8 15:28:19
categories: 
- java
tags:
- 源码
- SpringBoot
---

> 学习sb源码也有一段时间了，牛年立一个flag，写一个sb源码的系列文章。顺祝牛年春节快乐。

> 主要素材来源于慕课网上的视频教程[《全方位深入解析最新版SpringBoot源码》](https://coding.imooc.com/learn/list/404.html/)

# 系统初始化器
## 1.1 系统初始化器demo
## 1.2 系统初始化器用法
### 1.2.1 实现方式一
### 1.2.2 实现方式二
### 1.2.3 实现方式三
## 1.3 系统初始化器作用
### org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
### org.springframework.boot.context.ContextIdApplicationContextInitializer
### org.springframework.boot.context.config.DelegatingApplicationContextInitializer
### org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
### org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
### org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
## 1.4 源码剖析
### 1.4.1 总体流程
### 1.4.2 通用工厂加载机制
### 1.4.3 其他一些有意思的代码

# bean的生命之源
## 2.1 常用bean配置方式demo
### 2.1.1 XML配置方式
### 2.1.2 注解配置方式
## 2.2 关键源码剖析
### 2.2.1 ConfigationClassPostProcessor是bean之生命之源
### 2.2.2 怎么才算是配置类？
### 2.2.3 配置类解析之路漫漫
#### 2.2.3.1 解析application概览
#### 2.3.2.2 application处理ComponentScan注解
#### 2.2.3.3 解析TestController
#### 2.2.3.4 解析BeanConfigation
#### 2.2.3.5 application处理Import注解
#### 2.2.3.6 处理AutoConfigurationImportSelector
#### 2.2.3.7 ServletWebServerFactoryAutoConfiguration
#### 2.2.3.8 处理AutoConfigurationPackages$Registrar




