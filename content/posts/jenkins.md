---
title: jenkins
tags:
- devops
date: 2021-04-11 21:50:00
---

# helm 安装

```sh
helm upgrade -i jenkins jenkins/jenkins -n jenkins \
--set controller.ingress.enabled=true \
--set controller.ingress.path=/ \
--set controller.ingress.hostName=jenkins.yoogo.cc \
--set agent.privileged=true \
--set agent.volumes[0].type=HostPath \
--set agent.volumes[0].hostPath=/var/run/docker.sock \
--set agent.volumes[0].mountPath=/var/run/docker.sock
--set agent.image=registry.cn-hangzhou.aliyuncs.com/yoogo-tools/jenkins-agent \
--set agent.tag=4.7-3
```
