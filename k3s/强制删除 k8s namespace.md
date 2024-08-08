
> 以 kubernetes-dashboard 为例

### 1. 获取需要强制删除的 namespace 的信息

```
kubectl get namespace kubernetes-dashboard -o json > kubernetes-dashboard.json
```

### 2. 删除 finalizers 中的内容

![image.png](https://obsidian-1312372886.cos.ap-shanghai.myqcloud.com/20231129110722.png)

### 3. 运行 kube-proxy

```
kubectl proxy
```

### 4. 通过 api 删除 namespace

```
curl -k -H "Content-Type: application/json" -X PUT --data-binary @kubernetes-dashboard.json http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/finalize
```

### 5. 关闭 kube-proxy 并检查 namespace 是否被删除

```
kubectl get ns
```
