运行 busybox 容器

```
kubectl run dns-test -it --image=busybox:1.28 --rm
```

在 busybox 容器内使用 nslookup 命令检测域名是否生效, 以 Redis 主从架构为例

```
nslookup my-redis-headless
```

运行结果

```
/ # nslookup my-redis-headless
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-redis-headless
Address 1: 10.42.0.39 10-42-0-39.my-redis-replicas.default.svc.cluster.local
Address 2: 10.42.0.36 my-redis-master-0.my-redis-headless.default.svc.cluster.local
Address 3: 10.42.1.33 10-42-1-33.my-redis-replicas.default.svc.cluster.local
Address 4: 10.42.0.37 my-redis-replicas-0.my-redis-headless.default.svc.cluster.local
```
