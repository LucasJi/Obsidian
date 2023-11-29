## 使用helm安装redis  

  
1. 创建`values.yaml`文件, 自定义配置. 文件内容参考[官方配置](https://artifacthub.io/packages/helm/bitnami/redis).  
```yaml
auth:  
  enable: true  
  password: # password  
master:  
  persistence:  
    size: 4Gi 
  service:
	nodePorts:
	  redis: 30001
replica:  
  persistence:  
    size: 4Gi  
global:  
  storageClass: local-path
```
2. `helm repo add bitnami https://charts.bitnami.com/bitnami`  
3. `helm install my-redis bitnami/redis -f values.yaml`

## 从k3s外部访问Redis集群

```
kubectl port-forward --namespace default svc/my-redis-master 6379:6379
```

## 安装redis-client

```
kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:7.2.3-debian-11-r1 --command -- sleep infinity

kubectl exec --tty -i redis-client --namespace default -- bash
```
