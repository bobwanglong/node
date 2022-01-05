### 常用操作

#### 单个命名空间设置自动注入

指定demo命名空间设置自动注入

```shell
kubectl label namespace demo istio-injection=enabled
```
注入结果查看

```shell
kubectl get namespace -L istio-injection
```

