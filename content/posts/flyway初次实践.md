---
title: flyway初次实践
tags:
- java
date: 2020-10-10 22:27:01
---

## 为什么需要flyway/liquibase

准确的说应该是为什么需要database migrations？

1. 方便于多环境下的数据结构一致，例如多开发/测试环境下的数据库一致性，开发、测试、生产的结构一致性
2. 迁移文件跟随每次git迭代而迭代，保证了对db结构的版本控制
3. 清楚的通过代码了解数据库所处的状态
4. db环境轻松克隆

## database migrations工具不算缺点的缺点

1. 额外的增加了运维成本，此类工具需要与发布流程集成，即再每次发布应用前先执行migration操作，判断迁移成功后继续发布应用，对于运维和开发的协作来说，有一定的沟通成本和学习成本
2. 增加发布部署负担，因为无法自动化知道哪一次发布需要迁移，所以需要在每次执行，会增加一定的发布时间
3. 对于锁表的一些迁移只能通过手工执行而不是通过工具来执行，否则很可能发生部署事故，所以需要code reviewer 严格审核

## 基于flyway java api的开发环境

模块结构：

```
.
├── Dockerfile
├── Makefile
├── README.md
├── build.gradle
└── src
    └── main
        ├── java
        │   └── Migrant.java
        └── resources
            ├── db
            │   └── migration
            │       ├── V20200911114448__change_some_table.sql
            │       ├── V20200915102250__create_log_tables.sql
            │       ├── V20200919095412__add_colume_for_order.sql
            │       ├── V20200930171100__rename_colume_for_order.sql
            │       └── V20201009165800__add_colume_for_order.sql
            ├── migrant.properties
            └── migrant.properties.example
```

1. flyway 默认db/migration/为迁移目录
2. migrant.properties.example 配置example，migrant.properties为个性化配置，加入gitignore，方便区分多环境

## 基于flyway docker image 的部署方案

1. Dockerfile

   ```dockerfile
   FROM flyway/flyway:7-alpine
   COPY src/main/resources/db/migration/ /flyway/sql
   ```

   看起来非常简单，只是将本模块下的迁移文件复制到镜像中

2. Makefile

   ```
   NAME				:= "mall-migration"
   DOCKER_BASE_NAME	:= $(NAME):$(tag)
   BUILD_TAG			:= $(tag)

   ifeq ("$(BUILD_TAG)","")
   	BUILD_TAG	:= $(shell date +%Y%m%d%H%M)
   endif

   .PHONY: docker
   .DEFAULT_GOAL := docker

   docker: $(info ======== build $(NAME) release image:)
   	docker build -t $(NAME):$(BUILD_TAG) --rm -f Dockerfile .

   ```

   加入Makefile是为了方便运维，运维只需要make tag=xxx就可以了，屏蔽了docker build细节

3. run 这个镜像

    K8s job方式:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration
  namespace: migration
spec:
  template:
    spec:
      imagePullSecrets:
      - name: registry-xx
      restartPolicy: Never
      containers:
      - name: migration
        image: xxx:xx
        restartPolicy: Never
        imagePullPolicy: IfNotPresent
        args:
          - info
          - migrate
          - info
        env:
          - name: FLYWAY_URL
            value: jdbc:postgresql://pgsql:5432/xx?characterEncoding=utf8&useSSL=false
          - name: FLYWAY_USER
            value: postgres
          - name: FLYWAY_PASSWORD
            value: postgres # 可以放到secrets里，这里为了直观就直接放了
          - name: FLYWAY_SCHEMAS
            value: "false"
          - name: FLYWAY_VALIDATE_ON_MIGRATE
            value: "false"
          - name: FLYWAY_BASELINE_ON_MIGRATE
            value: "true"
          - name: FLYWAY_OUT_OF_ORDER
            value: "true"
  backoffLimit: 0
```

​    docker 运行：`docker run --name xx-migration xx/migration migration  `

4. 加入到你的部署流程里，放在部署应用之前

5. 除了配置数据源外其他有用的配置

   a. createSchemas: 是否创建一个新的schema

   b. schemas: 如果迁移文件未指定schema，则以这里指定的schema为准

   c. validateOnMigrate: 是否验证迁移文件，在执行完migration后，flyway会计算出每个文件的checksum，如果开启，那迁移完毕后再次调整这个文件时候抛出异常的

   d. baselineOnMigrate: 已存在的数据库需要指定baselineOnMigrate为true,才会生成flyway_schema_history表

   e. outOfOrder: 无序执行迁移，如果outOfOrder为false，则已经应用了1.0和3.0版本，现在又找到了2.0版，那么它也会执行而不是忽略。

   所有配置： https://flywaydb.org/documentation/configuration/parameters/

