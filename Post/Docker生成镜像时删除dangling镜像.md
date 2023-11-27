## 方案一

使用命令手动删除

`docker rmi $(docker images -f "dangling=true" -q)`

## 方案二

生成镜像的时候使用`--rm`参数自动移除中间镜像

`docker build --rm -t my-image:latest`
