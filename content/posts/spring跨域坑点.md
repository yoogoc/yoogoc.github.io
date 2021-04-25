---
title: spring 跨域坑点
tags:
- java
date: 2020-02-12 22:35
---

### 1. 正常的Spring boot项目

基本的spring boot项目只需要写个配置类，实现`WebMvcConfigurer`接口的`addCorsMappings`方法(或者直接注册CorsFilter bean)

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
            .allowCredentials(true)
            .allowedHeaders("*")
            .allowedOrigins("*")
            .allowedMethods("*");
}
```

这样就会给所有路由的请求加上跨域所需的headers

### 2. Spring Boot + Shiro

Spring Boot整合Shiro之后，默认所有请求会先经过shiro的监听器，所以上面的全局方法已经不管用了。（怎么整合shiro就不说了）

这时候就得祭出第二招，cors监听器。实现一个监听器，放在shiro的监听器之前，这样就可以保证在所有请求到来的时候，都会给response加上跨域headers。

```java
@Bean
public FilterRegistrationBean corsFilter() {
    final UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    final CorsConfiguration config = new CorsConfiguration();
    // 允许cookies跨域
    config.setAllowCredentials(true);
    // #允许向该服务器提交请求的URI，*表示全部允许，在SpringMVC中，如果设成*，会自动转成当前请求头中的Origin
    config.addAllowedOrigin("*");
    // #允许访问的头信息,*表示全部
    config.addAllowedHeader("*");
    // 预检请求的缓存时间（秒），即在这个时间段里，对于相同的跨域请求不会再预检了
    config.setMaxAge(18000L);
    // 允许提交请求的方法，*表示全部允许
    config.addAllowedMethod("OPTIONS");
    config.addAllowedMethod("HEAD");
    config.addAllowedMethod("GET");
    config.addAllowedMethod("PUT");
    config.addAllowedMethod("POST");
    config.addAllowedMethod("DELETE");
    config.addAllowedMethod("PATCH");
    source.registerCorsConfiguration("/**", config);

    FilterRegistrationBean bean = new FilterRegistrationBean(new CorsFilter(source));
    // 设置监听器的优先级
    bean.setOrder(0);

    return bean;
}
```
