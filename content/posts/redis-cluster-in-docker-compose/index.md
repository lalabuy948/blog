---
title: "Redis Cluster in Docker Compose"
date: 2020-08-16T16:37:03+02:00
draft: true
comments: true
cover: "img/redis-cluster-in-docker-compose/redis-docker.png"
tags:
  - redis
  - docker
  - devops
seo:
  ["redis cluster", "docker compose", "docker", "redis"]
---

> x: Why would you do this?

> y: Because one not enough? 

Initially your `redis` docker-compose setup would look like this:

```yml
version: '3'

services:
  redis:
    image: redis:6.0.6-alpine3.12
    container_name: redis_master
    ports:
      - "6379"
    restart: always
```

To be able to add second node slave/follower we need to change a bit our above example.

This [redis.conf](/stuff/redis-cluster-in-docker-compose/redis.dev.conf) changing
`bind 127.0.0.1` -> `bind 0.0.0.0` configuration. Plus let's add network.

```yml
version: '3'

networks:
  web-network:
    driver: bridge

services:
  redis:
    image: redis:6.0.6-alpine3.12
    container_name: redis_master
    networks:
      - web-network
    ports:
      - "6379"
    volumes:
      - ./infra/etc/redis.dev.conf:/redis.conf
    command: [ "redis-server", "/redis.conf" ]
    restart: always
```

Finally we can add second node

```yml
version: '3'

networks:
  web-network:
    driver: bridge

services:
  redis:
    image: redis:6.0.6-alpine3.12
    container_name: redis_master
    networks:
      - web-network
    ports:
      - "6379"
    volumes:
      - ./infra/etc/redis.dev.conf:/redis.conf
    command: [ "redis-server", "/redis.conf" ]
    restart: always

  redis-slave:
    image: redis:6.0.6-alpine3.12
    container_name: redis_slave
    networks:
      - web-network
    command: redis-server --slaveof redis 6379
    restart: always
```

If all good you will see something like this 

```sh
MASTER <-> REPLICA sync: Finished with success
```

ps
Redis will be accessible by this url `redis:6379`.
