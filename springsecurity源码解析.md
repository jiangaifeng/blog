---
title: 从源码解析SpringSecurity
date: 2021-2-28 15:28:19
categories: 
- java
tags:
- 源码
- SpringBoot
- SpringSecurity
---



# 构建SpringSecurity配置类

webSecurityExpressionHandler
依赖springSecurityFilterChain

```
public Filter springSecurityFilterChain() throws Exception {
        boolean hasConfigurers = this.webSecurityConfigurers != null && !this.webSecurityConfigurers.isEmpty();
        if (!hasConfigurers) {
            WebSecurityConfigurerAdapter adapter = (WebSecurityConfigurerAdapter)this.objectObjectPostProcessor.postProcess(new WebSecurityConfigurerAdapter() {
            });
            this.webSecurity.apply(adapter);
        }

        //private WebSecurity webSecurity;
        return (Filter)this.webSecurity.build();
    }
```

```
private void init() throws Exception {
        //SecurityConfig,这是我自定义的一个config
        Collection<SecurityConfigurer<O, B>> configurers = this.getConfigurers();
        Iterator var2 = configurers.iterator();

        SecurityConfigurer configurer;
        while(var2.hasNext()) {
            configurer = (SecurityConfigurer)var2.next();
            configurer.init(this);
        }

        var2 = this.configurersAddedInInitializing.iterator();

        while(var2.hasNext()) {
            configurer = (SecurityConfigurer)var2.next();
            //调用init
            configurer.init(this);
        }
}
```

```
public void init(final WebSecurity web) throws Exception {
        //
        final HttpSecurity http = this.getHttp();
        web.addSecurityFilterChainBuilder(http).postBuildAction(new Runnable() {
            public void run() {
                FilterSecurityInterceptor securityInterceptor = (FilterSecurityInterceptor)http.getSharedObject(FilterSecurityInterceptor.class);
                web.securityInterceptor(securityInterceptor);
            }
        });
    }
```

```
protected final HttpSecurity getHttp() throws Exception {
        if (this.http != null) {
            return this.http;
        } else {
            DefaultAuthenticationEventPublisher eventPublisher = (DefaultAuthenticationEventPublisher)this.objectPostProcessor.postProcess(new DefaultAuthenticationEventPublisher());
            this.localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);
            AuthenticationManager authenticationManager = this.authenticationManager();
            this.authenticationBuilder.parentAuthenticationManager(authenticationManager);
            Map<Class<? extends Object>, Object> sharedObjects = this.createSharedObjects();
            this.http = new HttpSecurity(this.objectPostProcessor, this.authenticationBuilder, sharedObjects);
            if (!this.disableDefaults) {
                ((HttpSecurity)((DefaultLoginPageConfigurer)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)((HttpSecurity)this.http.csrf().and()).addFilter(new WebAsyncManagerIntegrationFilter()).exceptionHandling().and()).headers().and()).sessionManagement().and()).securityContext().and()).requestCache().and()).anonymous().and()).servletApi().and()).apply(new DefaultLoginPageConfigurer())).and()).logout();
                ClassLoader classLoader = this.context.getClassLoader();
                List<AbstractHttpConfigurer> defaultHttpConfigurers = SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);
                Iterator var6 = defaultHttpConfigurers.iterator();

                while(var6.hasNext()) {
                    AbstractHttpConfigurer configurer = (AbstractHttpConfigurer)var6.next();
                    this.http.apply(configurer);
                }
            }

            //调用security的configure实现
            this.configure(this.http);
            return this.http;
        }
    }
```

```
public FormLoginConfigurer<HttpSecurity> formLogin() throws Exception {
        return (FormLoginConfigurer)this.getOrApply(new FormLoginConfigurer());
    }
```

```
public FormLoginConfigurer() {
        //UsernamePasswordAuthenticationFilter
        super(new UsernamePasswordAuthenticationFilter(), (String)null);
        this.usernameParameter("username");
        this.passwordParameter("password");
    }
```

# 表单认证过滤逻辑

```
AbstractAuthenticationProcessingFilter.class
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest)req;
        HttpServletResponse response = (HttpServletResponse)res;
        if (!this.requiresAuthentication(request, response)) {
            chain.doFilter(request, response);
        } else {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Request is to process authentication");
            }

            Authentication authResult;
            try {
                authResult = this.attemptAuthentication(request, response);
                if (authResult == null) {
                    return;
                }

                this.sessionStrategy.onAuthentication(authResult, request, response);
            } catch (InternalAuthenticationServiceException var8) {
                this.logger.error("An internal error occurred while trying to authenticate the user.", var8);
                this.unsuccessfulAuthentication(request, response, var8);
                return;
            } catch (AuthenticationException var9) {
                this.unsuccessfulAuthentication(request, response, var9);
                return;
            }

            if (this.continueChainBeforeSuccessfulAuthentication) {
                chain.doFilter(request, response);
            }

            this.successfulAuthentication(request, response, chain, authResult);
        }
    }
```

在哪里调用的？
DelegatingFilterProxy.class 
```
 protected void invokeDelegate(Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        //filterChain
        //characterEncodingFilter
        //hiddenHttpMethodFilter
        //formContentFilter
        //requestContextFilter
        //springSecurityFilterChain
        //Tomcat WebSocket (JSR356) Filter
        delegate.doFilter(request, response, filterChain);
    }
```

```
private void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        FirewalledRequest fwRequest = this.firewall.getFirewalledRequest((HttpServletRequest)request);
        HttpServletResponse fwResponse = this.firewall.getFirewalledResponse((HttpServletResponse)response);
        //WebAsyncManagerIntegrationFilter
        //org.springframework.security.web.context.SecurityContextPersistenceFilter@23f3da8b
        //org.springframework.security.web.header.HeaderWriterFilter@7640a5b1
        //org.springframework.security.web.csrf.CsrfFilter@7724704f
        //org.springframework.security.web.authentication.logout.LogoutFilter@15b642b9
        //org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@518bfd90
        //org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter@1fb2d5e
        //DefaultLogoutPageGeneratingFilter
        //org.springframework.security.web.savedrequest.RequestCacheAwareFilter@5634d0f4
        //org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@7d44a19
        //AnonymousAuthenticationFilter
        //org.springframework.security.web.session.SessionManagementFilter@642f9a77
        //org.springframework.security.web.access.ExceptionTranslationFilter@4b97c4ad
        //FilterSecurityInterceptor
        List<Filter> filters = this.getFilters((HttpServletRequest)fwRequest);
        if (filters != null && filters.size() != 0) {
            FilterChainProxy.VirtualFilterChain vfc = new FilterChainProxy.VirtualFilterChain(fwRequest, chain, filters);
            vfc.doFilter(fwRequest, fwResponse);
        } else {
            if (logger.isDebugEnabled()) {
                logger.debug(UrlUtils.buildRequestUrl(fwRequest) + (filters == null ? " has no matching filters" : " has an empty filter list"));
            }

            fwRequest.reset();
            chain.doFilter(fwRequest, fwResponse);
        }
    }
```