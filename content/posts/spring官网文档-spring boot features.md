---
title: spring官网文档-spring boot features
tags:
- java
date: 2020-05-07 14:08:00
---

# 1. SpringApplication

默认情况下，会显示信息日志消息，包括一些相关的启动细节，比如启动应用程序的用户。如果您需要一个日志级别，而不是INFO，您可以设置它，如日志级别中所述。应用程序版本是使用主应用程序类包中的实现版本确定的。通过设置spring.main.log-startup-info可以关闭启动信息日志。



### 1. Lazy init

`spring.main.lazy-initialization=true`

SpringApplication允许应用程序被延迟初始化。当启用延迟初始化时，bean是根据需要创建的，而不是在应用程序启动期间创建的。因此，启用延迟初始化可以减少应用程序启动所需的时间。在web应用程序中，启用延迟初始化将导致在接收到HTTP请求之前不会初始化许多与web相关的bean。

延迟初始化的一个缺点是，它会延迟应用程序问题的发现。如果错误配置的bean是惰性初始化的，那么在启动期间将不再出现故障，只有在初始化bean时问题才会变得明显。还必须注意确保JVM有足够的内存来容纳应用程序的所有bean，而不仅仅是那些在启动期间初始化的bean。由于这些原因，默认情况下不会启用延迟初始化，建议在启用延迟初始化之前对JVM堆大小进行微调。

可以使用SpringApplicationBuilder上的lazyinitialize方法或SpringApplication上的setlazyinitialize方法以编程方式启用延迟初始化。或者，也可以使用spring.main来启用它。延迟初始化属性如下例所示:

### 2. 自定义Banner

支持的变量：

| Variable                                                     | Description                                                  |
| :----------------------------------------------------------- | ------------------------------------------------------------ |
| `${application.version}`                                     | The version number of your application, as declared in `MANIFEST.MF`. For example, `Implementation-Version: 1.0` is printed as `1.0`. |
| `${application.formatted-version}`                           | The version number of your application, as declared in `MANIFEST.MF` and formatted for display (surrounded with brackets and prefixed with `v`). For example `(v1.0)`. |
| `${spring-boot.version}`                                     | The Spring Boot version that you are using. For example `2.2.6.RELEASE`. |
| `${spring-boot.formatted-version}`                           | The Spring Boot version that you are using, formatted for display (surrounded with brackets and prefixed with `v`). For example `(v2.2.6.RELEASE)`. |
| `${Ansi.NAME}` (or `${AnsiColor.NAME}`, `${AnsiBackground.NAME}`, `${AnsiStyle.NAME}`) | Where `NAME` is the name of an ANSI escape code. See [`AnsiPropertySource`](https://github.com/spring-projects/spring-boot/tree/v2.2.6.RELEASE/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/ansi/AnsiPropertySource.java) for details. |
| `${application.title}`                                       | The title of your application, as declared in `MANIFEST.MF`. For example `Implementation-Title: MyApp` is printed as `MyApp`. |

> The `SpringApplication.setBanner(…)` method can be used if you want to generate a banner programmatically. Use the `org.springframework.boot.Banner` interface and implement your own `printBanner()` method.
