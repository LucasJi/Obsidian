
1. 进入容器
```bash
docker exec -it `container-id` bash
```
2. 导出堆栈文件
```bash
jmap -dump:live,format=b,file=dump.hprof 1
```
3. 下载堆栈文件
```bash
docker cp `container-id`:dump.hprof ~
```
