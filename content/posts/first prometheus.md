---
title: first prometheus(待完善)
tags:
- k8s
date: 2020-11-30 23:18:00
---

## 核心组件

1. Prometheus Server， 服务本身，主要用于抓取数据和存储时序数据，另外还提供查询和 Alert Rule 配置管理。
2. client libraries，客户端资源库，用于对接 Prometheus Server, 可以查询和上报数据。
3. push gateway ，推送网关，用于批量，短期的监控数据的汇总节点，主要用于业务数据汇报等。
4. 各种汇报数据的 exporters ，例如汇报机器数据的 node_exporter, 汇报 MongoDB 信息的 MongoDB exporter 等等。
5. 用于告警通知管理的 alertmanager 。

官网架构图：

![架构](/images/prometheus-architecture.png)

## helm安装

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm upgrade --install prometheus prometheus-community/prometheus \
-n prometheus --create-namespace \
--set alertmanager.ingress.enabled=true \
--set alertmanager.ingress.hosts[0]=alertmanager.yoogo.cn \
--set pushgateway.ingress.enabled=true \
--set pushgateway.ingress.hosts[0]=pushgateway.yoogo.cn \
--set server.ingress.enabled=true,server.ingress.hosts[0]=prometheus.yoogo.cn

helm install grafana bitnami/grafana -n prometheus \
--set global.storageClass=local-path \
--set ingress.enabled=true,ingress.hosts[0].name=grafana.yoogo.cn


```

