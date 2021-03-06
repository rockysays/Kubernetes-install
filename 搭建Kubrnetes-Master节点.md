# 部署Kubernetes Master节点

## kube-apiserver组件
kube-apiserver是Kubernetes最重要的核心组件之一，主要提供以下的功能：

+ 提供集群管理的REST API接口，包括认证授权、数据校验以及集群状态变更等
+ 提供其他模块之间的数据交互和通信的枢纽(其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd)。

## kube-scheduler组件
kube-scheduler是Kubernetes最重要的核心组件之一，主要提供以下的功能:
+ 负责分配调度Pod到集群内的节点上，它监听kube-apiserver，查询还未分配Node的Pod，然后根据调度策略为这些Pod分配节点(更新Pod的NodeName字段)。

## kube-controller-manager组件
Controller Manager是Kubernetes最重要的核心组件之一，主要提供以下的功能：
+ 主要kube-controller-manager和cloud-controller-manager组成，是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。

## 下载kubernetes组件的二进制文件

``` bash
# wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/kubernetes-server-linux-amd64.tar.gz
# tar fx kubernetes-server-linux-amd64.tar.gz
```

拷贝二进制文件

``` bash
# mkdir -p /usr/local/kubernetes-v1.7.4/bin
# ln -s /usr/local/kubernetes-v1.7.4 /usr/local/kubernetes
# cp -r `pwd`/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/kubernetes/bin
```

## 配置kube-config文件

``` bash
# vim /etc/kubernetes/kube-config
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=false"
 
# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"
 
# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"
 
# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=https://192.168.100.110:6443"
```

## 配置和启动kube-apiserver

创建kube-apiserver配置文件

``` bash
# vim /etc/kubernetes/kube-apiserver
###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=192.168.100.110 --bind-address=0.0.0.0 --insecure-bind-address=0.0.0.0"
#
## The port on the local server to listen on.
KUBE_API_PORT="--insecure-port=8080 --secure-port=6443"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://192.168.100.110:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=172.17.0.0/16"
#
## default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=--admission-control=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota"
#
## Add your own!
KUBE_API_ARGS="--apiserver-count=3 \
               --storage-backend=etcd3 \
               --requestheader-allowed-names \
               --authorization-mode=Node,RBAC \
               --service-node-port-range=1024-65535 \
               --requestheader-group-headers=X-Remote-Group \
               --requestheader-username-headers=X-Remote-User \
               --log-dir=/data/kubernetes/logs/kube-apiserver \
               --client-ca-file=/etc/kubernetes/ssl/ca.pem \
               --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem \
               --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
               --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
               --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"
```

创建kube-apiserver启动脚本

``` bash
# vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kube-apiserver Service
After=network.target
 
[Service]
EnvironmentFile=-/etc/kubernetes/kube-config
EnvironmentFile=-/etc/kubernetes/kube-apiserver
ExecStart=/usr/local/kubernetes/bin/kube-apiserver \
          $KUBE_LOGTOSTDERR \
          $KUBE_LOG_LEVEL \
          $KUBE_ETCD_SERVERS \
          $KUBE_API_ADDRESS \
          $KUBE_API_PORT \
          $KUBELET_PORT \
          $KUBE_ALLOW_PRIV \
          $KUBE_SERVICE_ADDRESSES \
          $KUBE_ADMISSION_CONTROL \
          $KUBE_API_ARGS
 
Restart=on-failure
Type=notify
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```

创建kube-apiserver日志目录

``` bash
# mkdir /data/kubernetes/logs/kube-apiserver -p
# systemctl daemon-reload
# systemctl enable kube-apiserver && systemctl start kube-apiserver && systemctl status kube-apiserver
```

## 配置和启动kube-controller-manager服务

创建kube-controller-manager配置文件

``` bash
# vim /etc/kubernetes/kube-controller-manager
###
# The following values are used to configure the kubernetes controller-manager
 
# defaults from config and apiserver should be adequate
 
# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS=--address=127.0.0.1 \
                             --leader-elect=true \
                             --cluster-name=kubernetes \
                             --use-service-account-credentials=true \
                             --root-ca-file=/etc/kubernetes/ssl/ca.pem \
                             --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
                             --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
                             --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
                             --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig
```

创建kube-controller-manager TLS认证配置文件

``` bash
# vim /etc/kubernetes/kube-controller-manager.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://192.168.100.110:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-controller-manager
  name: kube-context
current-context: kube-context
users:
- name: system:kube-controller-manager
  user:
    client-certificate: /etc/kubernetes/ssl/kube-controller-manager.pem
    client-key: /etc/kubernetes/ssl/kube-controller-manager-key.pem
```

创建kube-controller-manager启动脚本

``` bash
# vim /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kube-controller-manager Service
After=network.target
 
[Service]
EnvironmentFile=-/etc/kubernetes/kube-config
EnvironmentFile=-/etc/kubernetes/kube-controller-manager
ExecStart=/usr/local/kubernetes/server/bin/kube-controller-manager \
          $KUBE_LOGTOSTDERR \
          $KUBE_LOG_LEVEL \
          $KUBE_MASTER \
          $KUBE_CONTROLLER_MANAGER_ARGS
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```

创建kube-controller-manager日志目录

``` bash
# mkdir /data/kubernetes/logs/kube-controller-manager -p
# systemctl daemon-reload
# systemctl enable kube-controller-manager && systemctl start kube-controller-manager && systemctl status kube-controller-manager
```

## 配置和启动kube-scheduler服务

创建kube-scheduler配置文件

``` bash
# vim /etc/kubernetes/kube-scheduler
###
# kubernetes scheduler config
 
# default config should be adequate
 
# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true \
                     --address=127.0.0.1 \
                     --log-dir=/data/kubernetes/logs/kube-scheduler \
                     --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig"
```

创建kube-scheduler TLS认证配置文件

``` bash
# vim /etc/kubernetes/kube-scheduler.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://192.168.3.99:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-scheduler
  name: kube-context
current-context: kube-context
users:
- name: system:kube-scheduler
  user:
    client-certificate: /etc/kubernetes/ssl/kube-scheduler.pem
    client-key: /etc/kubernetes/ssl/kube-scheduler-key.pem
```

创建kube-scheduler启动脚本

``` bash
# vim /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=kube-scheduler Service
After=network.target
 
[Service]
EnvironmentFile=-/etc/kubernetes/kube-config
EnvironmentFile=-/etc/kubernetes/kube-scheduler
ExecStart=/usr/local/kubernetes/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```

创建kube-scheduler日志目录

``` bash
# mkdir /data/kubernetes/logs/kube-scheduler -p
# systemctl daemon-reload
# systemctl enable kube-scheduler && systemctl start kube-scheduler && systemctl status kube-scheduler
```
# 验证Kubernetes Master节点状态

``` bash
# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
```
