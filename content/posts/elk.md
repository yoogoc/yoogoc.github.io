---
title: elk docker单例搭建
tags:
- devops
date: 2020-02-15 01:48
---

## elasticsearch

```sh
sudo docker pull elasticsearch:7.7.1
sudo docker run -p 9200:9200 -p 9300:9300 --name elasticsearch -e "discovery.type=single-node" -d elasticsearch:7.7.1
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.7.1/elasticsearch-analysis-ik-7.7.1.zip
```



##  logstash

```sh
sudo docker pull logstash:7.7.1
```



## kibana

```sh
sudo docker pull kibana:7.7.1
sudo docker run --link elasticsearch:elasticsearch -d -it --rm -p 5601:5601 kibana:7.7.1
```

## rancher

```
sudo docker run -d --restart=unless-stopped -p 8080:8080 rancher/server \
    --db-host tieba.yoogo9.me --db-port 3307 --db-user root --db-pass 199633 --db-name rancher
```

