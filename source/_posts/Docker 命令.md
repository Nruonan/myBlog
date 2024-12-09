---
title: Docker 命令
date: 2024-05-13 20:00:10
tags:

---



# Docker 命令

## 创建redis

```
docker run --restart=always -p 6379:6379 --name myredis -d redis:latest  --requirepass Nruonan996
```



## 创建rabbitmq

```
docker run -id --name=rabbitmq -v rabbitmq-home:/var/lib/rabbitmq -p 15672:15672 -p 5672:5672 -e RABBITMQ_DEFAULT_USER=xzn -e RABBITMQ_DEFAULT_PASS=Nruonan996 rabbitmq:management
```

## 创建minio

```
docker run -p 9000:9000 -p 9090:9090 --name minio \
     -d --restart=always \
     -e "MINIO_ACCESS_KEY=minio" \
     -e "MINIO_SECRET_KEY=feiwu996" \
     -v database:/data \
     minio/minio server \
     /data --console-address ":9090" -address ":9000"
```

