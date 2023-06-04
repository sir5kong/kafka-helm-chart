# Kafka helm chart

## Prerequisites

- Kubernetes 1.18+
- Helm 3.3.0+

## 部署案例

``` shell
git clone https://github.com/sir5kong/kafka-docker.git
cd kafka-docker

# kubectl create namespace your-namespace

## 部署单节点集群, 仅启动一个 Pod
helm install kafka -n your-namespace -f ./examples/values-combined.yml ./charts/kafka

## 部署生产集群, 3 个 controller 实例, 3 个 broker 实例
helm install kafka -n your-namespace -f ./examples/values-production.yml ./charts/kafka

######  组合使用 values 文件  ######
## 部署生产集群，并开启 kafka-ui 和 kafka exporter
helm install kafka -n your-namespace \
  -f ./examples/values-exporter.yml \
  -f ./examples/values-ui.yml \
  -f ./examples/values-production.yml \
  ./charts/kafka

## 以 NodePort 对集群外暴露
helm install kafka -n your-namespace -f ./examples/values-nodeport.yml ./charts/kafka

## 以 LoadBalancer 对集群外暴露
helm install kafka -n your-namespace -f ./examples/values-loadbalancer.yml ./charts/kafka

## 开启 kafka-ui
helm install kafka -n your-namespace -f ./examples/values-ui.yml ./charts/kafka

```

## 混部模式 `combined mode`

`process.roles` 可以设置为 `broker` `controller` `broker,controller`, 设为 `broker,controller` 即混部模式。

混部的服务器部署和运维管理都更加方便，缺点是不方便扩容。例如部署一个高可用集群 3 个 controller 和 3 个 broker，后续如果需要扩容 broker 可以在线操作，不影响业务。如果是混部模式扩容操作会相对复杂，而且会造成业务中断。

以下是官网原文：

> Kafka servers that act as both brokers and controllers are referred to as "combined" servers. Combined servers are simpler to operate for small use cases like a development environment. The key disadvantage is that the controller will be less isolated from the rest of the system. For example, it is not possible to roll or scale the controllers separately from the brokers in combined mode. Combined mode is not recommended in critical deployment environments.

### Chart Values

| Key | 类型 | 默认值 | 描述 |
|-----|------|---------|-------------|
| broker.combinedMode.enabled | bool | `false` | 是否开启混部模式 |

``` yaml
## 混部案例
broker:
  combinedMode:
    enabled: true
  replicaCount: 1
  heapOpts: "-Xms1024m -Xmx1024m"
  persistence:
    enabled: true
    size: 20Gi
```

## Cluster ID 

早期版本中，Kafka 会自动初始化数据目录，目前的版本需要提供一个 `Cluster ID` 并手动初始化数据目录：

``` shell
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

一个集群所有的节点都需要使用同一个 `Cluster ID`，本项目为了解决这个问题会在首次部署时自动生成一个 `Cluster ID` 并保存到 `Secret`。

## node.id

`Cluster ID` 是集群内所有节点共享，而 `node.id` 是每个节点必须唯一。

例如在 `3 controller 3 broker` 的集群里，controller 的 `node.id` 分别是 `0` `1` `2`，跟 StatefulSet Pod 序号一致，但 broker 就不能再使用同样的 id 了。因此本项目引入 `KAFKA_NODE_ID_OFFSET` 这个变量，默认是 `1000`，所以 broker 的 `node.id` 分别是 `1000` `1001` `1002`。

## 外部暴露

想要在集群外连接 Kafka 服务器，必须把每个 Broker 暴露出去，并且正确配置 `advertised.listeners`。

这里支持 2 种暴露方式，`NodePort` 和 `LoadBalancer`，有几个 broker 节点就需要对应数量的 `NodePort` 或 `LoadBalancer`。

### Chart Values

| Key | 类型 | 默认值 | 描述 |
|-----|------|---------|-------------|
| broker.external.enabled | bool | `false` | 是否开启外部暴露 |
| broker.external.service.type | string | `NodePort` | 外部暴露类型，支持 `NodePort` 和 `LoadBalancer` |
| broker.external.service.annotations | object | `{}` | 外部暴露的 service 注解 |
| broker.external.nodePorts | list | `[]` | `NodePort` 端口号，至少提供一个端口号，如果端口数量少于 broker 节点数，则自动递增 |
| broker.external.domainSuffix | string | `kafka.example.com` | 使用 `LoadBalancer` 对外暴露则必须使用域名，broker 对应的外部域名是 `POD_NAME` + `域名后缀`，例如 `kafka-broker-0.kafka.example.com`，部署结束后需要完成域名解析配置 |

``` yaml
## NodePort 案例
broker:
  replicaCount: 3
  external:
    enabled: true
    service:
      type: "NodePort"
      annotations: {}
    nodePorts:
      - 31050
      - 31051
      - 31052
```

``` yaml
## LoadBalancer 案例
broker:
  replicaCount: 3
  external:
    enabled: true
    service:
      type: "LoadBalancer"
      annotations: {}
    domainSuffix: "kafka.example.com"
```
