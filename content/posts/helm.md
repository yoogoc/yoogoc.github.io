---
title: helm
tags:
- k8s
date: 2020-07-23 13:06:00
---

# helm 本身

## install

```shell
# 不推荐，慢如蜗牛
curl https://helm.baltorepo.com/organization/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
# 快如闪电
sudo snap install helm --classic

helm repo add apphub https://apphub.aliyuncs.com
```



## 获取charts渲染后的yml

```
helm create test
cd test
helm install --debug --generate-name --dry-run test .
helm template test .
```

## helm & ns

helm与kubectl一样，默认namespace为default，需要手动指定all

```
helm ls --all-namespaces --all
helm ls -A -a
```



## helm cli & harbor

1. 安装harbor时 `./install.sh --with-chartmuseum` 使其开启chartmuseum功能
2. 安装helm push 插件 `helm plugin install https://github.com/chartmuseum/helm-push`
3. [详细文档](https://goharbor.io/docs/2.0.0/working-with-projects/working-with-images/managing-helm-charts/)
4. 坑点1，helm repo add 自定义库时可能会发生‘503 Service Unavailable’，重启终端即可
5. 坑点2，helm push 一定要注意version的一致性，否则harbor会显示HELM_CHART.NO_DETAIL



# 安装某些组件

## kong

```shell
helm repo add kong https://charts.konghq.com
helm repo update
# 如果下载不成功就clone github.com/kong/charts,并且把require lock相关删掉
helm install kong . \
--set ingressController.installCRDs=false \
--set admin.enabled=true,admin.http.enabled=true \
--set manager.ingress.enabled=true,manager.ingress.hostname=kong.cc \
-n kong --create-namespace
```

```
helm install kong kong/kong --set admin.enabled=true --set admin.http.enabled=true --set postgresql.enabled=true
```

## konga

```shell
helm install konga-pg . -n kong --set ingress.enabled=true,ingress.hosts[0].host=konga.cc,ingress.hosts[0].paths[0]=/
# konga 不支持pg10+
config.node_env=production,config.db_adapter=postgres,config.db_host=pg-serivce.pg,config.db_port=5432,config.db_user=postgres,config.db_password=pgsql@123,config.db_uri=postgresql://pg-serivce.pg:5432/konga_database,config.ssl_key_path=nil,config.ssl_crt_path=nil,config.token_secret=yoogosecret
```

## Rancher

```shell
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.yoogo.vip --set ingress.tls.source=tls-rancher-ingress	--set addLocal="false"

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.yoogo.vip,replicas=1,tls=external
```

## metallb

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
cat > metallb-ns.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
  labels:
    app: metallb
EOF
kubectl apply -f metallb-ns.yml

cat > metallb-config.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.220-192.168.1.229
EOF
kubectl apply -f metallb-config.yaml

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

```



## traefik

```
helm repo add traefik https://helm.traefik.io/traefik
helm install traefik traefik/traefik -n kube-system --set service.loadBalancerIP=192.168.1.220
cat > dashboard.yaml <<EOF
# dashboard.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
spec:
  entryPoints:
    - web
  routes:
    - match: Host(\`traefik.yoogo.vip\`) && (PathPrefix(\`/dashboard\`) || PathPrefix(\`/api\`))
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
EOF

kubectl apply -f dashboard.yaml
```



## nginx-ingress-controller

```
helm install nginx-ingress-controller bitnami/nginx-ingress-controller -n nginx-system --create-namespace --set service.loadBalancerIP=192.168.1.220
```



## redis

```shell
helm install redis bitnami/redis --set global.redis.password=123456,global.storageClass=local-path,master.persistence.size=1Gi,slave.persistence.size=1Gi,master.service.nodePort=32001,master.service.type=NodePort,cluster.enabled=false -n redis --create-namespace
```

# pg & debezium

```shell
cat > extended.conf <<EOF
# extended.conf

wal_level = logical
max_wal_senders = 1
max_replication_slots = 1
EOF

kubectl create configmap --namespace postgres --from-file=extended.conf postgresql-config

helm install postgres bitnami/postgresql --set image.repository=debezium/postgres,image.tag=12,global.postgresql.postgresqlPassword=postgres,global.storageClass=local-path,service.type=NodePort,service.nodePort=35432,persistence.size=1Gi,extendedConfConfigMap=postgresql-config -n postgres --create-namespace
```

## chartmuseum

```shell
helm install chart apphub/chartmuseum --set persistence.enabled=true,persistence.storageClass=fast,persistence.size=2Gi,ingress.enabled=true,ingress.hosts[0].name='charts.yoogo.cc' -n env --create-namespace
```

## kafka

```shell
helm upgrade -i kafka --namespace kafka bitnami/kafka --create-namespace \
--set external.enabled=true \
--set global.storageClass=local-path,persistence.size=50Gi \
--set zookeeper.persistence.size=10Gi \
--set livenessProbe.initialDelaySeconds=100 \
--set livenessProbe.timeoutSeconds=100,readinessProbe.initialDelaySeconds=100 \
--set readinessProbe.timeoutSeconds=100 \
--set externalAccess.enabled=true \
--set externalAccess.service.type=NodePort \
--set externalAccess.service.nodePorts[0]=29094 \
--set externalAccess.service.useHostIPs=true
```

## jenkins

```shell
helm install jenkins jenkins/jenkins -n jenkins --create-namespace --set persistence.storageClass=fast,persistence.size=1Gi,master.ingress.enabled=true,master.ingress.hostName=jenkins.yoogo.cc,master.ingress.path=/
```

## mysql

```shell
helm install mysql bitnami/mysql -n mysql --create-namespace --set global.storageClass=local-path,auth.rootPassword=mysql
```

## nacos

```shell
helm install nacos . -n nacos --create-namespace \
--set mysql.persistence.enabled=true \
--set global.mode=cluster,persistence.enabled=true \
--set ingress.enabled=true \
--set ingress.hosts[0].host='nacos.yoogo.cc',ingress.hosts[0].paths[0]='/' \
--set service.rpcPort=37848 \
--set resources.requests.cpu=100m,resources.requests.memory=128Mi
```

## longhorn

```shell
helm install longhorn longhorn/longhorn --namespace longhorn-system --set ingress.enabled=true,ingress.host=longhorn.yoogo.cc,csi.kubeletRootDir=/var/lib/kubelet
```

## kafdrop

```shell
helm upgrade -i kafdrop chart --set kafka.brokerConnect=kafka-headless:9092,server.servlet.contextPath="/",cmdArgs="--message.format=AVRO --schemaregistry.connect=http://localhost:8080",jvm.opts="-Xms32M -Xmx64M",ingress.enabled=true,ingress.hosts[0]='kafdrop.yoogo.cc' -n kafka
```



## es

```
helm install elasticsearch apphub/elasticsearch -n elasticsearch --create-namespace --set global.storageClass=longhorn,master.service.nodePort=30003,master.persistence.size=2Gi,master.livenessProbe.enabled=false,master.readinessProbe.enabled=false,coordinating.replicas=1,coordinating.readinessProbe.enabled=false,coordinating.livenessProbe.enabled=false,data.replicas=1,data.persistence.size=2Gi,data.livenessProbe.enabled=false,data.readinessProbe.enabled=false,global.kibanaEnabled=true,kibana.global.storageClass=longhorn,kibana.persistence.size=2Gi,kibana.livenessProbe.enabled=fasle,kibana.readinessProbe.enabled=false,kibana.ingress.enabled=true,kibana.ingress.hosts[0].name=kibana.yoogo.cc
```

## gitlab
不建议gitlab放在k8s上
```shell
helm install gitlab gitlab/gitlab -n gitlab --create-namespace \
--set global.hosts.domain=yoogo.cn \
--set global.ingress.configureCertmanager=false,certmanager.install=false \
--set prometheus.install=false,global.grafana.enabled=false \
--set nginx-ingress.enabled=false \
--set global.ingress.tls.enabled=false,global.hosts.https=false \
--set global.ingress.class=nginx
```

