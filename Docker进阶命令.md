# 删除镜像



删除所有镜像

```
docker rmi `docker images -q`
```



镜像名包含关键字

```
docker rmi `docker images | grep goharbor | awk '{print $3}'`
```



# 删除容器



删除所有容器

```
docker rm `docker ps -a -q`
```



