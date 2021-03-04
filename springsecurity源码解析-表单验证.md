---
title: SpringSecurity源码解析-表单验证
date: 2021-3-4 15:28:19
categories: 
- java
tags:
- 源码
- SpringBoot
- SpringSecurity
---



# 在写这篇文章之前的疑问

- spring security中有哪些过滤器？
- 过滤器、拦截器、切片的区别是什么？
- 登录成功的过滤器流程
- 登录失败的过滤器流程
- 无认证信息接口的过滤器流程
- 异常的流程
- 如何扩展表单验证，自定义一个手机号短信验证码的登录方式