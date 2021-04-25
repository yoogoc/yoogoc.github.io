---
title: swagger2 for spring boot
tags:
- java
date: 2020-01-31 14:10:01
---

## swagger2

swagger2作为业界优秀的轮子,其通用性和可拓展性都是首屈一指的(java界的只用过这个)

## 没有对比就没有伤害

公司用ruby,现在用的文档工具是apipie, 这玩意无力吐槽,只能生成简单的文档页面,其实这个还好,大不了就是麻烦一点,但是重点:

1. ruby作为一门资深的动态语言,方法的入参木有列表啊,随便来啊,还要手撸一边文档才能生成文档,ruby写起来很爽,但是维护起来要命,文档年久失修或者某一次加/减了参数但没有更新文档,那接下来的人根本没法快速定位,出来混,总是要还的.
2. 极大降低性能,众所周知,ruby是一个很神奇的语言,ruby确实能写出性能不低的代码,但是大部分人只能写出普通的代码(include me),因此稍微大一点的公司用ruby随着时间的推移系统会越来越慢,可能会是指数级的慢
3. 可拓展性差,ruby可以做到拓展性强,但是一大堆money patch玩不来玩不来.

## 我现在认为的干货

1. `ApiInfoBuilder().contact()` String类型参数已经过期,推荐使用Contact类型
2. `ApiInfoBuilder().securitySchemes()` 设置请求头信息
3. `ApiInfoBuilder().securityContexts()`:

```java
private List<SecurityContext> securityContexts() {
  List<SecurityContext> contexts = new ArrayList<>(1);
  SecurityContext securityContext = SecurityContext.builder()
    .securityReferences(defaultAuth())
    .forPaths(PathSelectors.regex("^((?!login).)*$"))
    .build();
  contexts.add(securityContext);
  return contexts;
}
private List<SecurityReference> defaultAuth() {
  List<SecurityReference> result = new ArrayList<>();
  AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
  AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
  authorizationScopes[0] = authorizationScope;
  result.add(new SecurityReference(tokenHeader, authorizationScopes));
  return result;
}
```
