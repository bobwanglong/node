# k8s中使用busybox访问集群内部服务
##### 查看正在运行的服务
```shell
kubectl get svc 
```
##### 使用busybox
```shell
kubectl run busybox --rm=true --image=busybox --restart=Never -it
```
###### 通过服务名可以访问服务
```shell
wget http://servername
```