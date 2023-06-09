# Kafka 容器化解决方案

Apache [Kafka](https://kafka.apache.org/) 容器化解决方案以及部署案例

[项目](https://github.com/sir5kong/kafka-docker)特色:

- 全面兼容 `KRaft`, 不依赖 ZooKeeper
- 灵活使用环境变量进行配置覆盖
- 提供 Kubernetes `helm chart`，支持集群外访问

相关链接:

- [GitHub](https://github.com/sir5kong/kafka-docker)
- [Docker Hub](https://hub.docker.com/r/sir5kong/kafka)
- [Dockerfile](https://github.com/sir5kong/kafka-docker/blob/main/Dockerfile)
- 初始化脚本 [entrypoint.sh](https://github.com/sir5kong/kafka-docker/blob/main/entrypoint.sh)

## Docker 部署

``` shell
## broker 默认端口 9092
docker run -d --name kafka-server \
  --network host \
  sir5kong/kafka:v3.5
```

自定义端口号：

``` shell
## 自定义端口号
docker run -d --name kafka-server \
  --network host \
  --env KAFKA_CONTROLLER_LISTENER_PORT=29091 \
  --env KAFKA_BROKER_LISTENER_PORT=29092 \
  sir5kong/kafka:v3.5
```

启动 kafka server 并持久化数据目录:

``` shell
docker volume create kafka_data
docker run -d --name kafka-server \
  --network host \
  -v kafka_data:/opt/kafka/data \
  sir5kong/kafka:v3.5
```

### Docker Compose

``` yaml
version: "3"

volumes:
  kafka-data: {}

## broker 默认端口 9092
## bootstrap-server: ${KAFKA_HOST_IP_ADDR}:9092

services:
  kafka:
    image: sir5kong/kafka:v3.5
    # restart: always
    network_mode: host
    volumes:
      - kafka-data:/opt/kafka/data
    environment:
      - KAFKA_HEAP_OPTS=-Xmx512m -Xms512m
      #- KAFKA_CONTROLLER_LISTENER_PORT=19091
      #- KAFKA_BROKER_LISTENER_PORT=9092

```

- 使用桥接网络请参考 [examples/docker-compose-bridge.yml](examples/docker-compose-bridge.yml)
- 更多部署案例和注解请参考 [examples](examples/)

## 环境变量

| 环境变量 | 默认值 | 描述 |
|---------|-------|------|
| `KAFKA_CLUSTER_ID`           | 随机生成 | 集群 id |
| `KAFKA_BROKER_LISTENER_PORT` | `9092` | broker 端口号，如果配置了 `KAFKA_CFG_LISTENERS` 则此项失效 |
| `KAFKA_CONTROLLER_LISTENER_PORT` | `19091` | controller 端口号，如果配置了 `KAFKA_CFG_LISTENERS` 则此项失效 |
| `KAFKA_HEAP_OPTS` | `null` | jvm 堆内存配置，例如 `-Xmx512m -Xms512m`|

### 配置覆盖

Kafka 所有配置项可以通过环境变量覆盖，除了 `log.dir` 和 `log.dirs`。环境变量名使用前缀 `KAFKA_CFG_` 加上配置参数

> 变量名中 `.` 需要替换为 `_`

例如 `KAFKA_CFG_LISTENERS` 对应配置参数 `listeners`，`KAFKA_CFG_ADVERTISED_LISTENERS` 对应配置参数 `advertised.listeners`

常用配置参数:

| 环境变量 | 配置参数 |
|---------|--------|
| `KAFKA_CFG_PROCESS_ROLES`     | `process.roles` |
| `KAFKA_CFG_LISTENERS`         | `listeners` |
| `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP`     | `listener.security.protocol.map` |
| `KAFKA_CFG_ADVERTISED_LISTENERS`               | `advertised.listeners` |
| `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS`           | `controller.quorum.voters` |

## helm chart 部署案例

``` shell
## 添加 helm 仓库
helm repo add kafka-repo https://helm-charts.itboon.top/kafka
helm repo update kafka-repo
```

``` shell
## 部署单节点集群, 仅启动一个 Pod
## 这里关闭持久化存储，仅演示部署效果
helm upgrade --install kafka \
  --namespace kafka-demo \
  --create-namespace \
  --set broker.combinedMode.enabled="true" \
  --set broker.persistence.enabled="false" \
  kafka-repo/kafka

## 部署 1 controller + 1 broker 集群, 并使用持久化存储
helm upgrade --install kafka \
  --namespace kafka-demo \
  --create-namespace \
  --set broker.persistence.size="20Gi" \
  kafka-repo/kafka

## 部署高可用集群, 3 controller + 3 broker
helm upgrade --install kafka \
  --namespace kafka-demo \
  --create-namespace \
  --set controller.replicaCount="3" \
  --set broker.replicaCount="3" \
  --set broker.heapOpts="-Xms4096m -Xmx4096m" \
  --set broker.resources.requests.memory="6Gi" \
  kafka-repo/kafka

## 生产集群详细 alues 请参考 https://github.com/sir5kong/kafka-docker/raw/main/examples/values-production.yml


## 以 LoadBalancer 对集群外暴露
helm upgrade --install kafka \
  --namespace kafka-demo \
  --create-namespace \
  --set broker.external.enabled="true" \
  --set broker.external.service.type="LoadBalancer" \
  --set broker.external.domainSuffix="kafka.example.com" \
  kafka-repo/kafka

## 以 NodePort 对集群外暴露
helm upgrade --install kafka \
  --namespace kafka-demo \
  --create-namespace \
  -f https://github.com/sir5kong/kafka-docker/raw/main/examples/values-nodeport.yml \
  kafka-repo/kafka

```
