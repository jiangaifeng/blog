---
title: 从源码解析SpringBoot-bean的生命之源
date: 2021-2-20 15:28:19
categories: 
- java
tags:
- 源码
- SpringBoot
---

[toc]

# bean的生命之源
## 2.1 常用bean配置方式demo
### 2.1.1 XML配置方式

```
<!--无参构造方式-->
    <!--<bean id="student" class="cn.jiangaifeng.bean.Student">-->
        <!--<property name="name" value="zhangsan"></property>-->
        <!--<property name="age" value="12"></property>-->
        <!--<property name="classList">-->
            <!--<list>-->
                <!--<value>math</value>-->
                <!--<value>english</value>-->
            <!--</list>-->
        <!--</property>-->
    <!--</bean>-->

    <!--有参构造方式-->
    <bean id="student" class="cn.jiangaifeng.bean.Student">
        <constructor-arg index="0" value="zhangsan"></constructor-arg>
        <constructor-arg index="1" value="11"></constructor-arg>
        <property name="classList">
            <list>
                <value>math</value>
                <value>english</value>
            </list>
        </property>
    </bean>

    <!--静态工厂-->
    <bean id="dog" class="cn.jiangaifeng.ioc.xml.AnimalFactory" factory-method="getAnimal">
        <constructor-arg value="dog"></constructor-arg>
    </bean>

    <!--动态工厂-->
    <!--<bean name="animalFactory" class="cn.jiangaifeng.ioc.xml.AnimalFactory"/>-->

    <!--<bean id="dog" factory-bean="animalFactory" factory-method="getAnimal2">-->
        <!--<constructor-arg value="dog"/>-->
    <!--</bean>-->

    <!--<bean id="cat" factory-bean="animalFactory" factory-method="getAnimal2">-->
        <!--<constructor-arg value="cat"/>-->
    <!--</bean>-->
```

### 2.1.2 注解配置方式


```
//使用@Component

//配置类中使用@Bean
@Configuration
public class BeanConfigation {
    @Bean("User")
    User getUser(){
        return new User();
    }
}

//实现FactoryBean
@Component
public class MyCat implements FactoryBean<Animal> {
    @Override
    public Animal getObject() throws Exception {
        return new Cat();
    }

    @Override
    public Class<?> getObjectType() {
        return Animal.class;
    }
}

//实现BeanDefinitionRegistryPostProcessor
@Component
public class MyBeanRegister implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition();
        rootBeanDefinition.setBeanClass(Monkey.class);
        beanDefinitionRegistry.registerBeanDefinition("monkey", rootBeanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}

//实现ImportBeanDefinitionRegistrar
public class MyBeanImport implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition();
        rootBeanDefinition.setBeanClass(Bird.class);
        beanDefinitionRegistry.registerBeanDefinition("bird", rootBeanDefinition);
    }
}

@Import(MyBeanImport.class)

```

## 2.2 关键源码剖析

### 2.2.1 ConfigationClassPostProcessor是bean之生命之源

ConfigationClassPostProcessor这一特殊的BeanFactoryPostProcessors是bean之源头

```
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    //省略...
    
    currentRegistryProcessors = new ArrayList();
    //org.springframework.context.annotation.internalConfigurationAnnotationProcessor
    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    String[] var16 = postProcessorNames;
    var9 = postProcessorNames.length;

    int var10;
    String ppName;
    for(var10 = 0; var10 < var9; ++var10) {
        ppName = var16[var10];
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
        }
    }

    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    //ConfigationClassPostProcessor=》processConfigBeanDefinitions
    //筛选出application配置类
    //parse application配置类
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();
    
    //省略...
}
```

### 2.2.2 怎么才算是配置类？

有@Configation注解
```
public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
    return metadata.isAnnotated(Configuration.class.getName());
}
```


有@Import注解
有@Component注解
有@ImportResource注解
有@ComponentScan注解
```
public static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
    if (metadata.isInterface()) {
        return false;
    } else {
        Iterator var1 = candidateIndicators.iterator();

        while(var1.hasNext()) {
            String indicator = (String)var1.next();
            if (metadata.isAnnotated(indicator)) {
                return true;
            }
        }

        try {
            return metadata.hasAnnotatedMethods(Bean.class.getName());
        } catch (Throwable var3) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + var3);
            }

            return false;
        }
    }
}
```

application是天选之子


### 2.2.3 配置类解析之路漫漫

#### 2.2.3.1 解析application概览

```
@Nullable
    protected final ConfigurationClassParser.SourceClass doProcessConfigurationClass(ConfigurationClass configClass, ConfigurationClassParser.SourceClass sourceClass) throws IOException {
        //1. 有Component注解，需要处理内部类
        //application不包含内部类，此处不处理
        if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
            this.processMemberClasses(configClass, sourceClass);
        }

        //2. 处理PropertySources注解
        //application不包含，此处不处理
        Iterator var3 = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), PropertySources.class, PropertySource.class).iterator();

        AnnotationAttributes importResource;
        while(var3.hasNext()) {
            importResource = (AnnotationAttributes)var3.next();
            if (this.environment instanceof ConfigurableEnvironment) {
                this.processPropertySource(importResource);
            } else {
                this.logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() + "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }

        //3. 处理ComponentScan注解  @ComponentScan("cn.jiangaifeng")
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            Iterator var13 = componentScans.iterator();

            while(var13.hasNext()) {
                AnnotationAttributes componentScan = (AnnotationAttributes)var13.next();
                //解析@Component 
                Set<BeanDefinitionHolder> scannedBeanDefinitions = this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                Iterator var7 = scannedBeanDefinitions.iterator();

                while(var7.hasNext()) {
                    BeanDefinitionHolder holder = (BeanDefinitionHolder)var7.next();
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }

                    //检查是否是Configation配置类，继续解析
                    //判断依据是2.2.2，有Full和Lite两种
                    //Full和Lite都会在此继续解析下去
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        this.parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        //4. 处理Imports注解
        this.processImports(configClass, sourceClass, this.getImports(sourceClass), true);
        
        //5.处理Import Resource注解
        importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            String[] var19 = resources;
            int var21 = resources.length;

            for(int var22 = 0; var22 < var21; ++var22) {
                String resource = var19[var22];
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }

        //6. 处理Bean Method注解
        Set<MethodMetadata> beanMethods = this.retrieveBeanMethodMetadata(sourceClass);
        Iterator var17 = beanMethods.iterator();

        while(var17.hasNext()) {
            MethodMetadata methodMetadata = (MethodMetadata)var17.next();
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        //7. 处理接口
        this.processInterfaces(configClass, sourceClass);
        
        //8. 处理父类
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);
                return sourceClass.getSuperClass();
            }
        }

        return null;
    }
```

#### 2.3.2.2 application处理ComponentScan注解

```
//3. 处理ComponentScan注解  @ComponentScan("cn.jiangaifeng")
//ComponentScanAnnotationParser
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry, componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
        Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
        boolean useInheritedGenerator = BeanNameGenerator.class == generatorClass;
        scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator : (BeanNameGenerator)BeanUtils.instantiateClass(generatorClass));
        ScopedProxyMode scopedProxyMode = (ScopedProxyMode)componentScan.getEnum("scopedProxy");
        if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
            scanner.setScopedProxyMode(scopedProxyMode);
        } else {
            Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
            scanner.setScopeMetadataResolver((ScopeMetadataResolver)BeanUtils.instantiateClass(resolverClass));
        }

        scanner.setResourcePattern(componentScan.getString("resourcePattern"));
        
        //设置includeFilters
        //跳过
        AnnotationAttributes[] var15 = componentScan.getAnnotationArray("includeFilters");
        int var8 = var15.length;

        int var9;
        AnnotationAttributes filter;
        Iterator var11;
        TypeFilter typeFilter;
        for(var9 = 0; var9 < var8; ++var9) {
            filter = var15[var9];
            var11 = this.typeFiltersFor(filter).iterator();

            while(var11.hasNext()) {
                typeFilter = (TypeFilter)var11.next();
                scanner.addIncludeFilter(typeFilter);
            }
        }

        //设置excludeFilters
        //跳过
        var15 = componentScan.getAnnotationArray("excludeFilters");
        var8 = var15.length;

        for(var9 = 0; var9 < var8; ++var9) {
            filter = var15[var9];
            var11 = this.typeFiltersFor(filter).iterator();

            while(var11.hasNext()) {
                typeFilter = (TypeFilter)var11.next();
                scanner.addExcludeFilter(typeFilter);
            }
        }

        //设置lazyInit
        //跳过
        boolean lazyInit = componentScan.getBoolean("lazyInit");
        if (lazyInit) {
            scanner.getBeanDefinitionDefaults().setLazyInit(true);
        }

        Set<String> basePackages = new LinkedHashSet();
        String[] basePackagesArray = componentScan.getStringArray("basePackages");
        String[] var19 = basePackagesArray;
        int var21 = basePackagesArray.length;

        int var22;
        for(var22 = 0; var22 < var21; ++var22) {
            String pkg = var19[var22];
            String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg), ",; \t\n");
            Collections.addAll(basePackages, tokenized);
        }

        //设置basePackageClasses
        Class[] var20 = componentScan.getClassArray("basePackageClasses");
        var21 = var20.length;

        for(var22 = 0; var22 < var21; ++var22) {
            Class<?> clazz = var20[var22];
            basePackages.add(ClassUtils.getPackageName(clazz));
        }

        if (basePackages.isEmpty()) {
            basePackages.add(ClassUtils.getPackageName(declaringClass));
        }

        scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
            protected boolean matchClassName(String className) {
                return declaringClass.equals(className);
            }
        });
        
        //执行包扫描
        return scanner.doScan(StringUtils.toStringArray(basePackages));
    }
```


```
//3. 处理ComponentScan注解  @ComponentScan("cn.jiangaifeng")
//ComponentScanAnnotationParser
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet();
        String[] var3 = basePackages;
        int var4 = basePackages.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String basePackage = var3[var5];
            //扫描包下的候选bean（component&managedbean）
            Set<BeanDefinition> candidates = this.findCandidateComponents(basePackage);
            Iterator var8 = candidates.iterator();

            while(var8.hasNext()) {
                BeanDefinition candidate = (BeanDefinition)var8.next();
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                candidate.setScope(scopeMetadata.getScopeName());
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                if (candidate instanceof AbstractBeanDefinition) {
                    this.postProcessBeanDefinition((AbstractBeanDefinition)candidate, beanName);
                }

                if (candidate instanceof AnnotatedBeanDefinition) {
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition)candidate);
                }

                if (this.checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    this.registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }

        return beanDefinitions;
    }
```


```
//ClassPathScanningCandidateComponentProvider.class
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
        LinkedHashSet candidates = new LinkedHashSet();

        try {
            //classpath*:cn/jiangaifeng/**/*.class
            String packageSearchPath = "classpath*:" + this.resolveBasePackage(basePackage) + '/' + this.resourcePattern;
            //加载所有的resource
            Resource[] resources = this.getResourcePatternResolver().getResources(packageSearchPath);
            boolean traceEnabled = this.logger.isTraceEnabled();
            boolean debugEnabled = this.logger.isDebugEnabled();
            Resource[] var7 = resources;
            int var8 = resources.length;

            for(int var9 = 0; var9 < var8; ++var9) {
                Resource resource = var7[var9];
                if (traceEnabled) {
                    this.logger.trace("Scanning " + resource);
                }

                if (resource.isReadable()) {
                    try {
                        MetadataReader metadataReader = this.getMetadataReaderFactory().getMetadataReader(resource);
                        //过滤Compent或者ManagedBean，细节不深究了
                        //通过Condition注解过滤
                        //通过include和exclude注解过滤
                        if (this.isCandidateComponent(metadataReader)) {
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                            sbd.setResource(resource);
                            sbd.setSource(resource);
                            //过滤
                            if (this.isCandidateComponent((AnnotatedBeanDefinition)sbd)) {
                                if (debugEnabled) {
                                    this.logger.debug("Identified candidate component class: " + resource);
                                }

                                candidates.add(sbd);
                            } else if (debugEnabled) {
                                this.logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        } else if (traceEnabled) {
                            this.logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    } catch (Throwable var13) {
                        throw new BeanDefinitionStoreException("Failed to read candidate component class: " + resource, var13);
                    }
                } else if (traceEnabled) {
                    this.logger.trace("Ignored because not readable: " + resource);
                }
            }

            return candidates;
        } catch (IOException var14) {
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", var14);
        }
    }
```




#### 2.2.3.3 解析TestController

```
//ConfigurationClassParser.class
//可以看到又回到这个解析方法了
 @Nullable
    protected final ConfigurationClassParser.SourceClass doProcessConfigurationClass(ConfigurationClass configClass, ConfigurationClassParser.SourceClass sourceClass) throws IOException {
        //没有内部类，跳过
        if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
            this.processMemberClasses(configClass, sourceClass);
        }

        //没有PropertySource，跳过
        Iterator var3 = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), PropertySources.class, PropertySource.class).iterator();

        AnnotationAttributes importResource;
        while(var3.hasNext()) {
            importResource = (AnnotationAttributes)var3.next();
            if (this.environment instanceof ConfigurableEnvironment) {
                this.processPropertySource(importResource);
            } else {
                this.logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() + "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }

        //没有ComponentScans，跳过
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            Iterator var13 = componentScans.iterator();

            while(var13.hasNext()) {
                AnnotationAttributes componentScan = (AnnotationAttributes)var13.next();
                Set<BeanDefinitionHolder> scannedBeanDefinitions = this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                Iterator var7 = scannedBeanDefinitions.iterator();

                while(var7.hasNext()) {
                    BeanDefinitionHolder holder = (BeanDefinitionHolder)var7.next();
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }

                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        this.parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        //没有Imports，跳过
        this.processImports(configClass, sourceClass, this.getImports(sourceClass), true);
        
        //没有ImportResource，跳过
        importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            String[] var19 = resources;
            int var21 = resources.length;

            for(int var22 = 0; var22 < var21; ++var22) {
                String resource = var19[var22];
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }

        //没有BeanMethod，跳过
        Set<MethodMetadata> beanMethods = this.retrieveBeanMethodMetadata(sourceClass);
        Iterator var17 = beanMethods.iterator();

        while(var17.hasNext()) {
            MethodMetadata methodMetadata = (MethodMetadata)var17.next();
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        //处理接口，跳过
        this.processInterfaces(configClass, sourceClass);
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);
                return sourceClass.getSuperClass();
            }
        }

        return null;
    }
```

#### 2.2.3.4 解析BeanConfigation

```
//ConfigurationClassParser.class
//同样还是回到这个解析方法
 @Nullable
    protected final ConfigurationClassParser.SourceClass doProcessConfigurationClass(ConfigurationClass configClass, ConfigurationClassParser.SourceClass sourceClass) throws IOException {
        if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
            this.processMemberClasses(configClass, sourceClass);
        }

        Iterator var3 = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), PropertySources.class, PropertySource.class).iterator();

        AnnotationAttributes importResource;
        while(var3.hasNext()) {
            importResource = (AnnotationAttributes)var3.next();
            if (this.environment instanceof ConfigurableEnvironment) {
                this.processPropertySource(importResource);
            } else {
                this.logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() + "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }

        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            Iterator var13 = componentScans.iterator();

            while(var13.hasNext()) {
                AnnotationAttributes componentScan = (AnnotationAttributes)var13.next();
                Set<BeanDefinitionHolder> scannedBeanDefinitions = this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                Iterator var7 = scannedBeanDefinitions.iterator();

                while(var7.hasNext()) {
                    BeanDefinitionHolder holder = (BeanDefinitionHolder)var7.next();
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }

                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        this.parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        this.processImports(configClass, sourceClass, this.getImports(sourceClass), true);
        importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            String[] var19 = resources;
            int var21 = resources.length;

            for(int var22 = 0; var22 < var21; ++var22) {
                String resource = var19[var22];
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }
        
        //获得Bean Method
        Set<MethodMetadata> beanMethods = this.retrieveBeanMethodMetadata(sourceClass);
        Iterator var17 = beanMethods.iterator();

        while(var17.hasNext()) {
            MethodMetadata methodMetadata = (MethodMetadata)var17.next();
            //添加到private final Set<BeanMethod> beanMethods = new LinkedHashSet();中去
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        this.processInterfaces(configClass, sourceClass);
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);
                return sourceClass.getSuperClass();
            }
        }

        return null;
    }
```

#### 2.2.3.5 application处理Import注解

```
//ConfigurationClassParser.class
private void collectImports(ConfigurationClassParser.SourceClass sourceClass, Set<ConfigurationClassParser.SourceClass> imports, Set<ConfigurationClassParser.SourceClass> visited) throws IOException {
        //visited为了便于遍历父级注解
        if (visited.add(sourceClass)) {
            Iterator var4 = sourceClass.getAnnotations().iterator();

            while(var4.hasNext()) {
                ConfigurationClassParser.SourceClass annotation = (ConfigurationClassParser.SourceClass)var4.next();
                //org.springframework.boot.autoconfigure.AutoConfigurationPackages$Registrar
                //org.springframework.boot.autoconfigure.AutoConfigurationImportSelector
                String annName = annotation.getMetadata().getClassName();
                if (!annName.startsWith("java") && !annName.equals(Import.class.getName())) {
                    this.collectImports(annotation, imports, visited);
                }
            }

            imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
        }

    }
```

#### 2.2.3.6 处理AutoConfigurationImportSelector
springboot利器：自动配置

```
private void processImports(ConfigurationClass configClass, ConfigurationClassParser.SourceClass currentSourceClass, Collection<ConfigurationClassParser.SourceClass> importCandidates, boolean checkForCircularImports) {
        if (!importCandidates.isEmpty()) {
            if (checkForCircularImports && this.isChainedImportOnStack(configClass)) {
                this.problemReporter.error(new ConfigurationClassParser.CircularImportProblem(configClass, this.importStack));
            } else {
                this.importStack.push(configClass);

                try {
                    Iterator var5 = importCandidates.iterator();

                    while(var5.hasNext()) {
                        ConfigurationClassParser.SourceClass candidate = (ConfigurationClassParser.SourceClass)var5.next();
                        Class candidateClass;
                        if (candidate.isAssignable(ImportSelector.class)) {
                            //实现了ImportSelector
                            candidateClass = candidate.loadClass();
                            ImportSelector selector = (ImportSelector)BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                            ParserStrategyUtils.invokeAwareMethods(selector, this.environment, this.resourceLoader, this.registry);
                            //实现了DeferredImportSelector，执行handle
                            if (selector instanceof DeferredImportSelector) {
                                //添加到this.deferredImportSelectors中去
                                this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector)selector);
                            } else {
                                String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                                Collection<ConfigurationClassParser.SourceClass> importSourceClasses = this.asSourceClasses(importClassNames);
                                this.processImports(configClass, currentSourceClass, importSourceClasses, false);
                            }
                        } else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                            candidateClass = candidate.loadClass();
                            ImportBeanDefinitionRegistrar registrar = (ImportBeanDefinitionRegistrar)BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                            ParserStrategyUtils.invokeAwareMethods(registrar, this.environment, this.resourceLoader, this.registry);
                            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                        } else {
                            this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                            this.processConfigurationClass(candidate.asConfigClass(configClass));
                        }
                    }
                } catch (BeanDefinitionStoreException var15) {
                    throw var15;
                } catch (Throwable var16) {
                    throw new BeanDefinitionStoreException("Failed to process import candidates for configuration class [" + configClass.getMetadata().getClassName() + "]", var16);
                } finally {
                    this.importStack.pop();
                }
            }

        }
    }
```


```
//ConfigurationClassParser.class
//DeferredImportSelectorHandler
//private List<ConfigurationClassParser.DeferredImportSelectorHolder> deferredImportSelectors;
 public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
            //构造importSelector的holder
            ConfigurationClassParser.DeferredImportSelectorHolder holder = new ConfigurationClassParser.DeferredImportSelectorHolder(configClass, importSelector);
            if (this.deferredImportSelectors == null) {
                ConfigurationClassParser.DeferredImportSelectorGroupingHandler handler = ConfigurationClassParser.this.new DeferredImportSelectorGroupingHandler();
                handler.register(holder);
                handler.processGroupImports();
            } else {
                //执行此步骤，添加到deferredImportSelectors中去
                this.deferredImportSelectors.add(holder);
            }

        }
```


```
//ConfigurationClassParser.class
//在执行完application的配置文件解析之后，最后执行this.deferredImportSelectorHandler.process();
public void parse(Set<BeanDefinitionHolder> configCandidates) {
        Iterator var2 = configCandidates.iterator();

        while(var2.hasNext()) {
            BeanDefinitionHolder holder = (BeanDefinitionHolder)var2.next();
            BeanDefinition bd = holder.getBeanDefinition();

            try {
                if (bd instanceof AnnotatedBeanDefinition) {
                    this.parse(((AnnotatedBeanDefinition)bd).getMetadata(), holder.getBeanName());
                } else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition)bd).hasBeanClass()) {
                    this.parse(((AbstractBeanDefinition)bd).getBeanClass(), holder.getBeanName());
                } else {
                    this.parse(bd.getBeanClassName(), holder.getBeanName());
                }
            } catch (BeanDefinitionStoreException var6) {
                throw var6;
            } catch (Throwable var7) {
                throw new BeanDefinitionStoreException("Failed to parse configuration class [" + bd.getBeanClassName() + "]", var7);
            }
        }

        //在此执行
        this.deferredImportSelectorHandler.process();
    }
```


```
//ConfigurationClassParser.class
//DeferredImportSelectorHandler
//private List<ConfigurationClassParser.DeferredImportSelectorHolder> deferredImportSelectors;
//与上文的hander遥相呼应
public void process() {
            //这里放的就是上文import的AutoConfigationImportSelector
            List<ConfigurationClassParser.DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
            this.deferredImportSelectors = null;

            try {
                if (deferredImports != null) {
                    ConfigurationClassParser.DeferredImportSelectorGroupingHandler handler = ConfigurationClassParser.this.new DeferredImportSelectorGroupingHandler();
                    deferredImports.sort(ConfigurationClassParser.DEFERRED_IMPORT_COMPARATOR);
                    //register
                    deferredImports.forEach(handler::register);
                    handler.processGroupImports();
                }
            } finally {
                this.deferredImportSelectors = new ArrayList();
            }

        }
```

```
//this.groupings
//this.configurationClasses
//deferredImports.forEach(handler::register);
public void register(ConfigurationClassParser.DeferredImportSelectorHolder deferredImport) {
            //org.springframework.boot.autoconfigure.AutoConfigurationImportSelector$AutoConfigurationGroup
            Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();
            //创建Group实例
            ConfigurationClassParser.DeferredImportSelectorGrouping grouping = (ConfigurationClassParser.DeferredImportSelectorGrouping)this.groupings.computeIfAbsent(group != null ? group : deferredImport, (key) -> {
                return new ConfigurationClassParser.DeferredImportSelectorGrouping(this.createGroup(group));
            });
            //Group里面添加了deferredImport(AutoConfigationImportSelector)
            grouping.add(deferredImport);
            //添加到this.configurationClasses里面去
            this.configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(), deferredImport.getConfigurationClass());
        }
```

```
//handler.processGroupImports();
public void processGroupImports() {
            Iterator var1 = this.groupings.values().iterator();

            while(var1.hasNext()) {
                ConfigurationClassParser.DeferredImportSelectorGrouping grouping = (ConfigurationClassParser.DeferredImportSelectorGrouping)var1.next();
                //getImports()获取所有的自动配置类
                grouping.getImports().forEach((entry) -> {
                    ConfigurationClass configurationClass = (ConfigurationClass)this.configurationClasses.get(entry.getMetadata());

                    try {
                        //processImports
                        ConfigurationClassParser.this.processImports(configurationClass, ConfigurationClassParser.this.asSourceClass(configurationClass), ConfigurationClassParser.this.asSourceClasses(entry.getImportClassName()), false);
                    } catch (BeanDefinitionStoreException var4) {
                        throw var4;
                    } catch (Throwable var5) {
                        throw new BeanDefinitionStoreException("Failed to process import candidates for configuration class [" + configurationClass.getMetadata().getClassName() + "]", var5);
                    }
                });
            }

        }
```


```
protected AutoConfigurationImportSelector.AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {
        //自动配置是否开启
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            //exclude
            //excludeName
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            //
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            configurations = this.removeDuplicates(configurations);
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.filter(configurations, autoConfigurationMetadata);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return new AutoConfigurationImportSelector.AutoConfigurationEntry(configurations, exclusions);
        }
    }
```


```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        //org.springframework.boot.autoconfigure.EnableAutoConfiguration
        //加载了所有的自动配置类
        /*
        org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration
        org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration
        org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
        org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration
        org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration
        org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration
        org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration
        org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration
        org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration
        org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration
        org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration
        org.springframework.boot.autoconfigure.task.TaskSchedulingAutoConfiguration
        org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration
        org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration
        org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
        org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration
         */
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
```

#### 2.2.3.7  ServletWebServerFactoryAutoConfiguration
看一下webserverfactory自动配置类是怎么工作的


```
private void processImports(ConfigurationClass configClass, ConfigurationClassParser.SourceClass currentSourceClass, Collection<ConfigurationClassParser.SourceClass> importCandidates, boolean checkForCircularImports) {
        if (!importCandidates.isEmpty()) {
            if (checkForCircularImports && this.isChainedImportOnStack(configClass)) {
                this.problemReporter.error(new ConfigurationClassParser.CircularImportProblem(configClass, this.importStack));
            } else {
                this.importStack.push(configClass);

                try {
                    Iterator var5 = importCandidates.iterator();

                    while(var5.hasNext()) {
                        ConfigurationClassParser.SourceClass candidate = (ConfigurationClassParser.SourceClass)var5.next();
                        Class candidateClass;
                        //org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
                        if (candidate.isAssignable(ImportSelector.class)) {
                            candidateClass = candidate.loadClass();
                            ImportSelector selector = (ImportSelector)BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                            ParserStrategyUtils.invokeAwareMethods(selector, this.environment, this.resourceLoader, this.registry);
                            if (selector instanceof DeferredImportSelector) {
                                this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector)selector);
                            } else {
                                String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                                Collection<ConfigurationClassParser.SourceClass> importSourceClasses = this.asSourceClasses(importClassNames);
                                this.processImports(configClass, currentSourceClass, importSourceClasses, false);
                            }
                        } else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                            candidateClass = candidate.loadClass();
                            ImportBeanDefinitionRegistrar registrar = (ImportBeanDefinitionRegistrar)BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                            ParserStrategyUtils.invokeAwareMethods(registrar, this.environment, this.resourceLoader, this.registry);
                            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                        } else {
                            //走这儿
                            this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                            this.processConfigurationClass(candidate.asConfigClass(configClass));
                        }
                    }
                } catch (BeanDefinitionStoreException var15) {
                    throw var15;
                } catch (Throwable var16) {
                    throw new BeanDefinitionStoreException("Failed to process import candidates for configuration class [" + configClass.getMetadata().getClassName() + "]", var16);
                } finally {
                    this.importStack.pop();
                }
            }

        }
    }
```


```
 @Nullable
    protected final ConfigurationClassParser.SourceClass doProcessConfigurationClass(ConfigurationClass configClass, ConfigurationClassParser.SourceClass sourceClass) throws IOException {
        if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
            //有内部类
            //
            this.processMemberClasses(configClass, sourceClass);
        }

        //没有PropertySources
        Iterator var3 = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), PropertySources.class, PropertySource.class).iterator();

        AnnotationAttributes importResource;
        while(var3.hasNext()) {
            importResource = (AnnotationAttributes)var3.next();
            if (this.environment instanceof ConfigurableEnvironment) {
                this.processPropertySource(importResource);
            } else {
                this.logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() + "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }

        //没有componentScans
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            Iterator var13 = componentScans.iterator();

            while(var13.hasNext()) {
                AnnotationAttributes componentScan = (AnnotationAttributes)var13.next();
                Set<BeanDefinitionHolder> scannedBeanDefinitions = this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                Iterator var7 = scannedBeanDefinitions.iterator();

                while(var7.hasNext()) {
                    BeanDefinitionHolder holder = (BeanDefinitionHolder)var7.next();
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }

                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        this.parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        //有Imports
        //org.springframework.boot.context.properties.EnableConfigurationPropertiesImportSelector
        //org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration$BeanPostProcessorsRegistrar
        //org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryConfiguration$EmbeddedTomcat
        //org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryConfiguration$EmbeddedJetty
        //org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryConfiguration$EmbeddedUndertow
        this.processImports(configClass, sourceClass, this.getImports(sourceClass), true);
        
        //没有ImportResource
        importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            String[] var19 = resources;
            int var21 = resources.length;

            for(int var22 = 0; var22 < var21; ++var22) {
                String resource = var19[var22];
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }

        //有beanMethods
        Set<MethodMetadata> beanMethods = this.retrieveBeanMethodMetadata(sourceClass);
        Iterator var17 = beanMethods.iterator();

        while(var17.hasNext()) {
            MethodMetadata methodMetadata = (MethodMetadata)var17.next();
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        this.processInterfaces(configClass, sourceClass);
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);
                return sourceClass.getSuperClass();
            }
        }

        return null;
    }
```

#### 2.2.3.8 处理AutoConfigurationPackages$Registrar

```
private void processImports(ConfigurationClass configClass, ConfigurationClassParser.SourceClass currentSourceClass, Collection<ConfigurationClassParser.SourceClass> importCandidates, boolean checkForCircularImports) {
        if (!importCandidates.isEmpty()) {
            if (checkForCircularImports && this.isChainedImportOnStack(configClass)) {
                this.problemReporter.error(new ConfigurationClassParser.CircularImportProblem(configClass, this.importStack));
            } else {
                //进入
                this.importStack.push(configClass);

                try {
                    Iterator var5 = importCandidates.iterator();

                    while(var5.hasNext()) {
                        ConfigurationClassParser.SourceClass candidate = (ConfigurationClassParser.SourceClass)var5.next();
                        Class candidateClass;
                        if (candidate.isAssignable(ImportSelector.class)) {
                            candidateClass = candidate.loadClass();
                            ImportSelector selector = (ImportSelector)BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                            ParserStrategyUtils.invokeAwareMethods(selector, this.environment, this.resourceLoader, this.registry);
                            if (selector instanceof DeferredImportSelector) {
                                this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector)selector);
                            } else {
                                String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                                Collection<ConfigurationClassParser.SourceClass> importSourceClasses = this.asSourceClasses(importClassNames);
                                this.processImports(configClass, currentSourceClass, importSourceClasses, false);
                            }
                        } else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                            //实现了ImportBeanDefinitionRegistrar，添加到容器中
                            candidateClass = candidate.loadClass();
                            ImportBeanDefinitionRegistrar registrar = (ImportBeanDefinitionRegistrar)BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                            ParserStrategyUtils.invokeAwareMethods(registrar, this.environment, this.resourceLoader, this.registry);
                            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                        } else {
                            this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                            this.processConfigurationClass(candidate.asConfigClass(configClass));
                        }
                    }
                } catch (BeanDefinitionStoreException var15) {
                    throw var15;
                } catch (Throwable var16) {
                    throw new BeanDefinitionStoreException("Failed to process import candidates for configuration class [" + configClass.getMetadata().getClassName() + "]", var16);
                } finally {
                    this.importStack.pop();
                }
            }

        }
    }
```
