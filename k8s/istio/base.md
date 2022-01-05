### 常用操作

#### 手动注入 sidecar
命名空间:demo

执行如下语句：

```shell
 kube-inject -f nginx-deployment.yaml | kubectl apply  -n demo -f -
 ```

#### 单个命名空间设置自动注入

指定demo命名空间设置自动注入

```shell
kubectl label namespace demo istio-injection=enabled
```
注入结果查看

```shell
kubectl get namespace -L istio-injection
kubectl get ns demo --show-labels # 查看 label 是否成功创建
```
