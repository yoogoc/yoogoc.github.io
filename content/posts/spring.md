---
title: spring 相关
tags:
- java
date: 2020-01-31 14:10:01
---
1. 支持的密码encoder `PasswordEncoderFactories`

2. url中  `/`不能有重复,否则会报`org.springframework.security.web.firewall.RequestRejectedException: The request was rejected because the URL was not normalized.`,从而触发`RestAuthenticationEntryPoint`

3. branner :

   ```
    _   _    ___     ___     __ _    ___
    | | | |  / _ \   / _ \   / _` |  / _ \
    | |_| | | (_) | | (_) | | (_| | | (_) |
     \__, |  \___/   \___/   \__, |  \___/
      __/ |                   __/ |
     |___/                   |___/
   ```



4. 重构多文件众多相似代码

   ```java
   // 重构前
   @Api(tags = "OmsOrderSettingController", description = "订单设置管理")
   // 使用正则搜索
   // \@Api\(tags = \"(\w*)\"\, description =
   // \@Api\(value = \".*", description =
   // 替换为
   // \@Api\(tags =
   // 重构后
   @Api(tags = "订单设置管理")
   \if\(\w*)\
   ```

5. 在Springboot应用开发中使用JPA时，通常在主应用程序所在包或者其子包的某个位置定义我们的Entity和Repository，这样基于Springboot的自动配置，无需额外配置，我们定义的Entity和Repository即可被发现和使用。但有时候我们需要定义Entity和Repository不在应用程序所在包及其子包，那么这时候就需要使用@EntityScan和@EnableJpaRepositories了

6. parent为`spring-boot-starter-parent`  ,就不要再dependencyManagement引入xxx了

7. `dependencyManagement`中引入新包,如果相关dependencies中没有使用,则maven不会去下载这个包,从而导致 Not found

