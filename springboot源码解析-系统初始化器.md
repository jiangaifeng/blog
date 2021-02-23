---
title: 从源码解析SpringBoot-系统初始化器
date: 2021-2-14 15:28:19
categories: 
- java
tags:
- 源码
- SpringBoot
---

[toc]

# 系统初始化器
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

//F:\github-jiangaifeng\java-demo-lib\springboot\src\main\resources\META-INF\spring.factories
org.springframework.context.ApplicationContextInitializer = cn.jiangaifeng.initializer.FirstInitializer
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

## 1.3 系统初始化器作用
看看springboot是怎么用的
### org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer

```
public void initialize(ConfigurableApplicationContext context) {
    context.addBeanFactoryPostProcessor(new ConfigurationWarningsApplicationContextInitializer.ConfigurationWarningsPostProcessor(this.getChecks()));
}
```
### org.springframework.boot.context.ContextIdApplicationContextInitializer

```
 public void initialize(ConfigurableApplicationContext applicationContext) {
        ContextIdApplicationContextInitializer.ContextId contextId = this.getContextId(applicationContext);
        applicationContext.setId(contextId.getId());
        applicationContext.getBeanFactory().registerSingleton(ContextIdApplicationContextInitializer.ContextId.class.getName(), contextId);
    }
```

### org.springframework.boot.context.config.DelegatingApplicationContextInitializer

```
public void initialize(ConfigurableApplicationContext context) {
        ConfigurableEnvironment environment = context.getEnvironment();
        List<Class<?>> initializerClasses = this.getInitializerClasses(environment);
        if (!initializerClasses.isEmpty()) {
            this.applyInitializerClasses(context, initializerClasses);
        }

    }
```

### org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer

```
public void initialize(ConfigurableApplicationContext applicationContext) {
        applicationContext.addApplicationListener(this);
    }

    public void onApplicationEvent(WebServerInitializedEvent event) {
        String propertyName = "local." + this.getName(event.getApplicationContext()) + ".port";
        this.setPortProperty((ApplicationContext)event.getApplicationContext(), propertyName, event.getWebServer().getPort());
    }
```


### org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer

```
public void initialize(ConfigurableApplicationContext applicationContext) {
        applicationContext.addBeanFactoryPostProcessor(new SharedMetadataReaderFactoryContextInitializer.CachingMetadataReaderFactoryPostProcessor());
    }
```


### org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

```
public void initialize(ConfigurableApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        applicationContext.addApplicationListener(new ConditionEvaluationReportLoggingListener.ConditionEvaluationReportListener());
        if (applicationContext instanceof GenericApplicationContext) {
            this.report = ConditionEvaluationReport.get(this.applicationContext.getBeanFactory());
        }

    }
```

## 1.4 源码剖析
### 1.4.1 总体流程
```
//1.SpringApplication构造函数中配置系统初始化器
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        //省略...
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //省略...
    }
    
//2.run方法中调用initializers的initialize方法
public ConfigurableApplicationContext run(String... args) {
        //省略...
            //在刷新方法之前，遍历调用initializers的initialize方法
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
        //省略...  
}
```

### 1.4.2 通用工厂加载机制

```
//加载ApplicationContextInitializer的实现类
//type参数是ApplicationContextInitializer
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
        ClassLoader classLoader = this.getClassLoader();
        //加载解析class路径下的META-INF/spring.factories
        //筛选出ApplicationContextInitializer实现并返回到names中
        Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        //反射出实例
        List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
        AnnotationAwareOrderComparator.sort(instances);
        return instances;
}
```

### 1.4.3 其他一些有意思的代码

```
//遍历class路径下的META-INF/spring.factories 
Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");

while(urls.hasMoreElements()) {
    //file:/F:/github-jiangaifeng/java-demo-lib/springboot/target/classes/META-INF/spring.factories
    //jar:file:/C:/Users/jiangaifeng/.m2/repository/org/springframework/boot/spring-boot/2.1.0.RELEASE/spring-boot-2.1.0.RELEASE.jar!/META-INF/spring.factories
    //jar:file:/C:/Users/jiangaifeng/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/2.1.0.RELEASE/spring-boot-autoconfigure-2.1.0.RELEASE.jar!/META-INF/spring.factories
    //jar:file:/C:/Users/jiangaifeng/.m2/repository/org/springframework/spring-beans/5.1.2.RELEASE/spring-beans-5.1.2.RELEASE.jar!/META-INF/spring.factories
    URL url = (URL)urls.nextElement();
    UrlResource resource = new UrlResource(url);
    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
    Iterator var6 = properties.entrySet().iterator();

    while(var6.hasNext()) {
        Entry<?, ?> entry = (Entry)var6.next();
        String factoryClassName = ((String)entry.getKey()).trim();
        String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
        int var10 = var9.length;

        for(int var11 = 0; var11 < var10; ++var11) {
            String factoryName = var9[var11];
            result.add(factoryClassName, factoryName.trim());
        }
    }
}
```


```
//反射
 try {
    Class<?> instanceClass = ClassUtils.forName(name, classLoader);
    Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
    T instance = BeanUtils.instantiateClass(constructor, args);
    instances.add(instance);
    } catch (Throwable var12) {
        throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, var12);
    }
```
