---
title: ceph install
tags:
- linux
date: 2020-10-01 14:52:00
---

# REQUIREMENTS

1. docker
2. NTP
3. lvm
4. Any modern Linux * 3

# INSTALL CEPHADM

```shell
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release octopus
./cephadm install
```

# BOOTSTRAP A NEW CLUSTER

```shell
mkdir -p /etc/ceph
cephadm bootstrap --mon-ip 192.168.1.31
```

=>

```
URL: https://ceph:8443/
User: admin
Password: 62zgr41d25
```

# ENABLE CEPH CLI

```shell
# 1. 使用adm代理
cephadm shell
# 2. 安装ceph-common
cephadm add-repo --release octopus
cephadm install ceph-common
```

# ADD HOSTS TO THE CLUSTER

```shell
# ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host>*
ssh-copy-id -f -i /etc/ceph/ceph.pub root@host2
# ceph orch host add *newhost*
ceph orch host add host2 192.168.1.32
```



# DEPLOY ADDITIONAL MONITORS (OPTIONAL)¶

add host以后会自动部署mon

# DEPLOY OSDS(IMPORTANT)

```shell
# ceph orch daemon add osd *<host>*:*<device-path>*
ceph orch daemon add osd ceph:/dev/vdc
```

# DEPLOY MDSS(CephFS)

```shell
# ceph orch apply mds *<fs-name>* --placement="*<num-daemons>* [*<host1>* ...]"
```

# DEPLOY RGWS

```shell
# ceph orch apply rgw *<realm-name>* *<zone-name>* --placement="*<num-daemons>* [*<host1>* ...]"
ceph orch apply rgw yoogo cn-guangzhou-1 --placement="1 ceph"

radosgw-admin user create --uid="test" --display-name="test user"
=> "access_key": "LT5J2627727WOD89ULZV"
=> "secret_key": "2PyRDiJezEgjdrw6JUMaAmD4m4XE7TdUKnFBFuL9"

radosgw-admin user create --uid='admin' --display-name='admin' --system
=> {
"user": "admin",
"access_key": "Z9MK5JDAF5Z3HCZNML8A",
"secret_key": "rx4e9K7SZ7wDzwiqwVM1czOBNa6NIPkxzkumntZY"
}
```
#### Gotcha

如果单节点部署必须要关掉自动扩容，否则Cluster Status可能一直warn，而rgw service依赖HEALTH_OK，

多节点部署也可能会出现这个问题，做法：多加几个osd

# ENABLING THE OBJECT GATEWAY MANAGEMENT FRONTEND

```shell
radosgw-admin user create --uid=<user_id> --display-name=<display_name> --system
radosgw-admin user info --uid=<user_id>
ceph dashboard set-rgw-api-access-key <access_key>
ceph dashboard set-rgw-api-secret-key <secret_key>


ceph dashboard set-rgw-api-host <host>
ceph dashboard set-rgw-api-port <port>
ceph dashboard set-rgw-api-scheme <scheme>  # http or https
ceph dashboard set-rgw-api-admin-resource <admin_resource> # 默认可以用
ceph dashboard set-rgw-api-user-id <user_id>
```

# s3cmd

```
s3cmd --configure
=> Access key and Secret key 填 radosgw-admin user info --uid=<user_id> 拿到的相应值
=> Default Region 这个默认US即可
=> S3 Endpoint 输入自己的rgw地址
=> DNS-style bucket+hostname 如果rgw已经有域名则可以设置为%(bucket)s.s3.amazonaws.com或者s3.amazonaws.com/%(bucket)，如果没有的话，只能设置为192.168.1.31/%(bucket)（特指格式，不是照抄就行）
=> 其余没有特殊要求均可默认，http/https自选

```

# RBD

最好用的块存储

1. [命令行创建](https://docs.ceph.com/en/latest/rados/operations/pools/#create-a-pool)或者dashborad创建一个pool
2. rbd初始化`rbd pool init <pool-name>`

3. 创建一个供rbd访问的user

```shell
# ceph auth get-or-create client.{ID} mon 'profile rbd' osd 'profile {profile name} [pool={pool-name}][, profile ...]' mgr 'profile rbd [pool={pool-name}]'
ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'

[client.kubernetes]
	key = AQBYwUBgKY0aDhAAj2Smeemh4p/lU4U4RbaoJw==
```

4. 创建rbd image

```shell
# rbd create --size {megabytes} {pool-name}/{image-name}
rbd create --size 1024 rbd-k8s1/yoogo
```

