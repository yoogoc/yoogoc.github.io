---
title: first envoy(待完善)
tags:
- cloudnative
date: 2020-12-05 23:07:00
---

## 为什么用

’为什么用‘基本等价于：他比别人好在哪里；解决了哪些刚需及主要矛盾；以及他的设计思想对我们自身现有及未来的技术栈有什么影响；会引入哪些问题，是否在可接受范围内；容错率的高低；运维成本如何······

（这大概也是团队每引入一个新技术所思考的）

### 核心思想

> The network should be transparent to applications. When network and application problems do occur it should be easy to determine the source of the problem.

翻译：我们开发的应用应该是对网络无感知的，当发生问题时可以简单并且准确的定位究竟是网络的问题还是应用本身的问题

注释：事实上，这个思想与[12factor](https://12factor.net/zh_cn/)中的’IV. 后端服务‘是不谋而合的，应用程序应该尽可能的无状态化

（说很简单，做很难）

## 运行时

1. sidecar模式：每启用一个应用程序就会启动一个对应的envoy进程，与应用程序通信使用localhost，优点：应用无感知，缺点：部署复杂度提高。
2. 可以拦截l3/l4层，因此可以拦截tcp，udp请求，envoy正是通过拦截请求的方式来实现的
3. 可以拦截http请求



（待完善）····



## 概念

1. host，具有稳定访问路径的主机
2. Downstream，发送请求的envoy的host
3. Upstream，接收请求的evnoy的host
4. Listener，是可以连接Downstream的网络地址（port，unix socket···）
5. Cluster，一组同一服务注册的Upstream
6. Mesh，一组提供同一网络拓扑的host
7. Runtime configuration，与Envoy一起部署的带外实时配置系统



## 基本配置

### 静态配置（简单分析下给出的示例）

```yaml
static_resources:

  listeners: # 监听器设置
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0 # 监听地址
        port_value: 10000 # 监听端口， 也就是说，外部可以通过 ip:10000的形式访问
    filter_chains: # 过滤链
    - filters: # 一组过滤器链，按序执行
      - name: envoy.filters.network.http_connection_manager # 名称，envoy支持的
        typed_config: #对应过滤器的配置,pb对象
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http # 你所需要的状态前缀，可在http://你自己的admin-ui/stats下看到
          access_log: #http访问日志
          - name: envoy.access_loggers.file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: /dev/stdout
          http_filters: # http过滤链，按序执行
          - name: envoy.filters.http.router
          route_config: #静态路由配置
            name: local_route # 随便给你的路由起个名字
            virtual_hosts: # 组成路由表的虚拟主机服务数组
            - name: local_service # 随便给你的虚拟主机起个名字，仅在统计信息时使用
              domains: ["*"] # 匹配域列表，支持通配符，具体看官网文档
              routes: # 路由表
              - match: # 匹配条件，具体支持哪些条件看官网文档
                  prefix: "/"
                route: # 路由信息
                  host_rewrite_literal: www.envoyproxy.io # 请求过程中重写http host header
                  cluster: service_envoyproxy_io # 需要路由到的上游集群（envoy概念中的集群）

  clusters:
  - name: service_envoyproxy_io # 要和route中的cluster相符
    connect_timeout: 30s # 连接超时
    type: LOGICAL_DNS # 服务发现类型，此处的LOGICAL_DNS具体理解要看官网
    # Comment out the following line to test on v6 networks
    dns_lookup_family: V4_ONLY # dns解析策略，AUTO/V4_ONLY/V6_ONLY
    load_assignment: # 加载策略
      cluster_name: service_envoyproxy_io
      endpoints: # 字面意思，翻译成中文感觉会损失一部分意思，还是直接读英文比较好
      - lb_endpoints:
        - endpoint:
            address: #目标地址
              socket_address: # 套接字地址
                address: www.envoyproxy.io #真正地址
                port_value: 443 # 端口
    transport_socket: # 交换套接字？ 主要用于TLS
      name: envoy.transport_sockets.tls # 固定字符串
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: www.envoyproxy.io #创建TLS后端连接时使用的SNI字符串
```

上述文档实现的东西很简单：

访问0.0.0.0:10000会看到www.envoyproxy.io的页面

看起来配置着实过于复杂

### 动态配置

（待续···）
