![](https://wilhelmguo.tk/api/file/getImage?fileId=5b24c7aeaddba4075b00000c)

随着微服务架构的火爆，Etcd作为服务发现或者分部式存储的基础平台也越来越频繁的出现在我们的视野里。因此对于快速部署一套高可用的Etcd集群的需求也越来越强烈，本次就带领大家一起使用Kubernetes的Statefulset特性快速部署一套Etcd集群。

## 什么是Kubernetes？
Kubernetes 是一个用于容器集群的自动化部署、扩容以及运维的开源平台。

使用Kubernetes，你可以快速高效地响应客户需求：

- 快速并且无意外的部署你的应用。
- 动态地对应用进行扩容。
- 无缝地发布新特性。
- 仅使用需要的资源以优化硬件使用。

## 什么是Etcd？
Etcd的目的是提供一个分布式键值动态数据库,维护一个"Configuration Registry"。
这个Registry的基础之一是Kubernetes集群发现和集中的配置管理。
它在某些方面类似于Redis,经典的LDAP配置后端以及Windows注册表。

Etcd的目标是：

- 简单：定义良好，面向用户的API（JSON and gRPC）
- 安全：自动TLS和可选的客户端证书身份验证
- 快速：benchmarked 写 10,000 次/秒
- 可靠：使用Raft协议作为分布式基础

## 官方已经有了Etcd-Operator,我为什么还要使用这种方式部署？

**首先来看优点：**

- Etcd-Operator的部署方式对Etcd版本和Kubernetes版本都有要求,详见[官方文档](https://github.com/coreos/etcd-operator)。
- 如果使用Etcd v2 Api则无法对数据进行备份，官方的Etcd集群数据备份仅仅支持Etcd v3版本
- Statefulset部署Etcd数据更加可靠。如果使用Etcd-Operator部署Etcd，不幸的是某天我的Kubernetes集群出现了问题，导致所有的Etcd Pod出现故障，那么很遗憾，如果没有对数据进行备份，我只能Pray God Bless，祈求自己不要背锅。即便使用Etcd V3 Api并且使用了数据备份，那么也可能丢失一部分数据，因为Etcd-Operator的备份是定时备份的。
- Statefulset的配置更加灵活，比如需要亲和性的配置，可以直接在Statefulset中增加即可。

**当然，任何美好的事务都不是完美的**。
使用Statefulset部署Etcd也需要一定的条件：
- 必须有可靠的网络存储作为支持。
- 创建集群比Etcd-Operator略微繁琐。

好，接下来，让我们进入正题：
## 如何在Kubernetes上快速部署一套Etcd集群。

首先，在Kubernetes上创建一个[Headless](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)的Service

``` yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: infra-etcd-cluster
    app: infra-etcd
  name: infra-etcd-cluster
  namespace: default
spec:
  clusterIP: None
  ports:
  - name: infra-etcd-cluster-2379
    port: 2379
    protocol: TCP
    targetPort: 2379
  - name: infra-etcd-cluster-2380
    port: 2380
    protocol: TCP
    targetPort: 2380
  selector:
    k8s-app: infra-etcd-cluster
    app: infra-etcd
  type: ClusterIP
```
创建Headless类型的Service是为了方便使用域名访问到Etcd的节点。2379和2380分别对应Etcd的Client Port和Peer Port。

接下来，让我们来创建Statefulset资源：

> 前提条件：你的集群必须提前创建好PV，以便Statefulset生成的Pod可以使用到。如果你使用StorageClass来管理PV，则无需手动创建。这里已Ceph-RBD为例。

``` yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    k8s-app: infra-etcd-cluster
    app: etcd
  name: infra-etcd-cluster
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: infra-etcd-cluster
      app: etcd
  serviceName: infra-etcd-cluster
  template:
    metadata:
      labels:
        k8s-app: infra-etcd-cluster
        app: etcd
      name: infra-etcd-cluster
    spec:
      containers:
      - command:
        - /bin/sh
        - -ec
        - |
          HOSTNAME=$(hostname)
          echo "etcd api version is ${ETCDAPI_VERSION}"

          eps() {
              EPS=""
              for i in $(seq 0 $((${INITIAL_CLUSTER_SIZE} - 1))); do
                  EPS="${EPS}${EPS:+,}http://${SET_NAME}-${i}.${SET_NAME}.${CLUSTER_NAMESPACE}:2379"
              done
              echo ${EPS}
          }

          member_hash() {
              etcdctl member list | grep http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380 | cut -d':' -f1 | cut -d'[' -f1
          }

          initial_peers() {
                PEERS=""
                for i in $(seq 0 $((${INITIAL_CLUSTER_SIZE} - 1))); do
                PEERS="${PEERS}${PEERS:+,}${SET_NAME}-${i}=http://${SET_NAME}-${i}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380"
                done
                echo ${PEERS}
          }

          # etcd-SET_ID
          SET_ID=${HOSTNAME##*-}
          # adding a new member to existing cluster (assuming all initial pods are available)
          if [ "${SET_ID}" -ge ${INITIAL_CLUSTER_SIZE} ]; then
              export ETCDCTL_ENDPOINTS=$(eps)

              # member already added?
              MEMBER_HASH=$(member_hash)
              if [ -n "${MEMBER_HASH}" ]; then
                  # the member hash exists but for some reason etcd failed
                  # as the datadir has not be created, we can remove the member
                  # and retrieve new hash
                  if [ "${ETCDAPI_VERSION}" -eq 3 ]; then
                      ETCDCTL_API=3 etcdctl --user=root:${ROOT_PASSWORD} member remove ${MEMBER_HASH}
                  else
                      etcdctl --username=root:${ROOT_PASSWORD} member remove ${MEMBER_HASH}
                  fi
              fi
              echo "Adding new member"
              rm -rf /var/run/etcd/*
              # ensure etcd dir exist
              mkdir -p /var/run/etcd/
              # sleep 60s wait endpoint become ready
              echo "sleep 60s wait endpoint become ready,sleeping..."
              sleep 60

              if [ "${ETCDAPI_VERSION}" -eq 3 ]; then
                  ETCDCTL_API=3 etcdctl --user=root:${ROOT_PASSWORD} member add ${HOSTNAME} --peer-urls=http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380 | grep "^ETCD_" > /var/run/etcd/new_member_envs
              else
                  etcdctl --username=root:${ROOT_PASSWORD} member add ${HOSTNAME} http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380 | grep "^ETCD_" > /var/run/etcd/new_member_envs
              fi
              
              

              if [ $? -ne 0 ]; then
                  echo "member add ${HOSTNAME} error."
                  rm -f /var/run/etcd/new_member_envs
                  exit 1
              fi

              cat /var/run/etcd/new_member_envs
              source /var/run/etcd/new_member_envs

              exec etcd --name ${HOSTNAME} \
                  --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380 \
                  --listen-peer-urls http://0.0.0.0:2380 \
                  --listen-client-urls http://0.0.0.0:2379 \
                  --advertise-client-urls http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2379 \
                  --data-dir /var/run/etcd/default.etcd \
                  --initial-cluster ${ETCD_INITIAL_CLUSTER} \
                  --initial-cluster-state ${ETCD_INITIAL_CLUSTER_STATE}
          fi

          for i in $(seq 0 $((${INITIAL_CLUSTER_SIZE} - 1))); do
              while true; do
                  echo "Waiting for ${SET_NAME}-${i}.${SET_NAME}.${CLUSTER_NAMESPACE} to come up"
                  ping -W 1 -c 1 ${SET_NAME}-${i}.${SET_NAME}.${CLUSTER_NAMESPACE} > /dev/null && break
                  sleep 1s
              done
          done

          echo "join member ${HOSTNAME}"
          # join member
          exec etcd --name ${HOSTNAME} \
              --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380 \
              --listen-peer-urls http://0.0.0.0:2380 \
              --listen-client-urls http://0.0.0.0:2379 \
              --advertise-client-urls http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2379 \
              --initial-cluster-token etcd-cluster-1 \
              --data-dir /var/run/etcd/default.etcd \
              --initial-cluster $(initial_peers) \
              --initial-cluster-state new

        env:
        - name: INITIAL_CLUSTER_SIZE
          value: "3"
        - name: CLUSTER_NAMESPACE
          valueFrom: 
            fieldRef:
              fieldPath: metadata.namespace
        - name: ETCDAPI_VERSION
          value: "3"
        - name: ROOT_PASSWORD
          value: '@123#'
        - name: SET_NAME
          value: "infra-etcd-cluster"
        - name: GOMAXPROCS
          value: "4"
        image: gcr.io/etcd-development/etcd:v3.3.8
        imagePullPolicy: Always
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -ec
              - |
                HOSTNAME=$(hostname)

                member_hash() {
                    etcdctl member list | grep http://${HOSTNAME}.${SET_NAME}.${CLUSTER_NAMESPACE}:2380 | cut -d':' -f1 | cut -d'[' -f1
                }

                eps() {
                    EPS=""
                    for i in $(seq 0 $((${INITIAL_CLUSTER_SIZE} - 1))); do
                        EPS="${EPS}${EPS:+,}http://${SET_NAME}-${i}.${SET_NAME}.${CLUSTER_NAMESPACE}:2379"
                    done
                    echo ${EPS}
                }
                
                export ETCDCTL_ENDPOINTS=$(eps)

                SET_ID=${HOSTNAME##*-}
                # Removing member from cluster
                if [ "${SET_ID}" -ge ${INITIAL_CLUSTER_SIZE} ]; then
                    echo "Removing ${HOSTNAME} from etcd cluster"
                    if [ "${ETCDAPI_VERSION}" -eq 3 ]; then
                        ETCDCTL_API=3 etcdctl --user=root:${ROOT_PASSWORD} member remove $(member_hash)
                    else
                        etcdctl --username=root:${ROOT_PASSWORD} member remove $(member_hash)
                    fi
                    if [ $? -eq 0 ]; then
                        # Remove everything otherwise the cluster will no longer scale-up
                        rm -rf /var/run/etcd/*
                    fi
                fi
        name: infra-etcd-cluster
        ports:
        - containerPort: 2380
          name: peer
          protocol: TCP
        - containerPort: 2379
          name: client
          protocol: TCP
        resources:
          limits:
            cpu: "4"
            memory: 4Gi
          requests:
            cpu: "4"
            memory: 4Gi
        volumeMounts:
        - mountPath: /var/run/etcd
          name: datadir
  updateStrategy:
    type: OnDelete
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      selector:
        matchLabels:
          k8s.cloud/storage-type: ceph-rbd

```

> 注意：SET_NAME必须与Statefulset的Name一致

这个时候你的Etcd已经可以在内部通过
``` sh 
http://${SET_NAME}-${i}.${SET_NAME}.${CLUSTER_NAMESPACE}:2379
```
访问了。

最后一步：创建Client Service

> 如果你的集群网络方案使Pod可以从Kubernetes集群外部访问，或者你的Etcd 集群只需要在Kubernetes集群内部访问，可以省略该步骤

``` yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: infra-etcd-cluster-client
    app: infra-etcd
  name: infra-etcd-cluster-client
  namespace: default
spec:
  ports:
  - name: infra-etcd-cluster-2379
    port: 2379
    protocol: TCP
    targetPort: 2379
  selector:
    k8s-app: infra-etcd-cluster
    app: infra-etcd
  sessionAffinity: None
  type: NodePort

```

大功告成！你可以使用NodePort顺利访问到Etcd集群。

## 扩容与缩容

###扩容

只需要将Statefulset中的replicas改变即可。例如，我想把集群数量扩容为5个。

``` sh
kubectl scale --replicas=5 statefulset infra-etcd-cluster
```

### 缩容

然后某一天我发现五个节点对我来说有些浪费，想使用三个节点。OK，只需要执行以下命令即可，
``` sh
kubectl scale --replicas=3 statefulset infra-etcd-cluster
```

![](https://wilhelmguo.tk/api/file/getImage?fileId=5b24d554addba4075b00000d)
