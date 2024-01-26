#### 1. 部署pod

1）准备 Pod 的 yaml 文件

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```



2）执行 kubectl 命令部署

```
kubectl create -f ${POD_YAML}
```



#### 2. 部署deployment

1）准备 Deployment 的 yaml 文件

```
apiVersion: extensions/v1beta1
 kind: Deployment
 metadata:
   name: rss-site
   namespace: mem-example
 spec:
   replicas: 2
   template:
     metadata:
       labels:
         app: web
     spec:
      containers:
       - name: memory-demo-ctr
         image: polinux/stress
         resources:
         limits:
           emory: "200Mi"
         requests:
           memory: "100Mi"
         command: ["stress"]
         args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
         volumeMounts:
         - name: redis-storage
           mountPath: /data/redis
     volumes:
     - name: redis-storage
       emptyDir: {}
```



这里相比pod.yaml多了replicas字段

- replicas：副本个数。也就是该 Deployment 需要起多少个相同的 Pod，如果用户成功在 K8S 中配置了 n（n>1）个，那么 Deployment 会确保在集群中始终有 n 个服务在运行。



2）执行 kubectl 命令部署

```
kubectl create -f ${DEPLOYMENT_YAML}
```

kubectl 会根据kind判断需要构建的服务类型，deployment会自动创建ReplicaSet和Pod

![](https://pcc.huitogo.club/k8s12.png)



#### 3. 部署Service

```
apiVersion: v1
kind: Service
metadata:
  name: cloud-native-hello-py
spec:
  selector:
    app: cloud-native-hello-py
  ports:
    - protocol: TCP
      port: 1024
      targetPort: 1024
```



部署pod或deployment需要对外暴露端口的，需要声明Service

注意：**targetPort 必须和 deployment 步骤里容器的导出端口一致！**



#### 4. 部署更新

1）查看服务

```
$ kubectl get|describe ${RESOURCE} [-o ${FORMAT}] -n=${NAMESPACE} # 
${RESOURCE}有: pod、deployment、replicaset(rs)
```



注意：要查看一个服务，也就是一个 Pod，必须首先指定 namespace

```
# 查看所有namespace
kubectl get ns
# 查看所有namespace下的pod
kubectl get pod --all-namespaces
```



2）更新服务

方法一：修改 yaml 文件后通过 kubectl 更新。

```
# 更新服务，不存在即创建 
kubectl apply -f ${YAML}
```



方法二：通过 kubectl 直接编辑 K8S 上的服务

```
kubectl edit ${RESOURCE} ${NAME}
```



注意：无论方法一还是方法二，能修改的内容还是有限的，从笔者实战下来的结论是：只能修改/更新镜像的地址和个别几个字段。



3）删除服务

使用deployment创建的pod，先删deployment再删pod，否则直接删pod就好

```
kubectl delete ${RESOURCE} ${NAME}
```



#### 5. 部署问题排查

![](https://pcc.huitogo.club/k8s13.png)

**总结：遇事不决先看部署日志，再看服务日志**

```
# 查看资源日志
kubectl describe ${RESOURCE} ${NAME}
# 查看服务容器日志
kubectl log ${POD_NAME} -c ${CONTAINER_NAME}
# 进入服务容器  即 docker exec xxx
kubectl exec -it [options] ${POD_NAME} -c ${CONTAINER_NAME} [args]
```



#### 6. 集群搭建

**1）本地环境**

使用minikube、kind搭建k8s

```
minikube start --vm-driver=docker --image-mirror-country='cn' 
kind create cluster --name test
```



**2）生产环境**

- 安装kubelet, kubectl, kubeadm 三件套
- kubeadm初始化集群

```
# 直接初始化
kubeadm init --apiserver-advertise-address=0.0.0.0 \
--apiserver-cert-extra-sans=127.0.0.1 \
--image-repository=registry.aliyuncs.com/google_containers \
--ignore-preflight-errors=all \
--kubernetes-version=v1.21.1 \
--service-cidr=10.10.0.0/16 \
--pod-network-cidr=10.18.0.0/16
```



或者手动指定配置

```
# 指定初始化配置文件
kubeadm config print init-defaults > kubeadm-config.yaml

vim kubeadm-config.yaml
    advertiseAddress: 192.168.88.10 # 配置主机IP
    imageRepository: registry.aliyuncs.com/google_containers # 修改镜像下载地址
    kubernetesVersion: 1.21.5 #设置版本
    networking:
    dnsDomain: cluster.local
    podSubnet: "10.244.0.0/16" # 设置pod 网络
    serviceSubnet: 10.96.0.0/12

# 拉取镜像
kubeadm config images pull --config kubeadm-config.yaml
# 初始化
kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.log
```



- Worker Node加入集群

```
kubeadm join 192.168.88.10:6443 --token abcdef.0123456789abcdef
--discovery-token-ca-cert-hash sha256:cdc49b48bf9de52eabef39cee3c2b41ad744f4cc9d9133d11318e4d0f45a5f6f
```

