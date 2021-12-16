# k8s apiVersion学习

编写k8s YAML文件的时候我们输入的第一个参数是`apiVersion`，那么这个参数是干什么用的呢，是我们自己定义自己编写的这个pod(等)的版本么，并不是。**`apiVersion`定义的是我们要使用的k8s API的版本。**

### apiVersion级别

- Alpha级别：

  该软件可能包含错误。启用一个功能可能会导致bug。

  随时可能会丢弃对该功能的支持，恕不另行通知

- Beta级别：

  版本名称包含 `beta` （例如， `v2beta3`）。

  软件经过很好的测试。启用功能被认为是安全的。

  默认情况下功能是开启的。

   细节可能会改变，但功能在后续版本不会被删除。

- Stable级别：

  版本名称如 `vX`，其中 `X` 为整数。

  稳定版本。

  将出现在后续发布的软件版本中。

### 查看当前可用的apiVersion

```
kubectl api-versions
```

- kubernetes v1.21.7

  ```
  admissionregistration.k8s.io/v1
  admissionregistration.k8s.io/v1beta1
  apiextensions.k8s.io/v1
  apiextensions.k8s.io/v1beta1
  apiregistration.k8s.io/v1
  apiregistration.k8s.io/v1beta1
  apps/v1
  authentication.k8s.io/v1
  authentication.k8s.io/v1beta1
  authorization.k8s.io/v1
  authorization.k8s.io/v1beta1
  autoscaling/v1
  autoscaling/v2beta1
  autoscaling/v2beta2
  batch/v1
  batch/v1beta1
  certificates.k8s.io/v1
  certificates.k8s.io/v1beta1
  coordination.k8s.io/v1
  coordination.k8s.io/v1beta1
  discovery.k8s.io/v1
  discovery.k8s.io/v1beta1
  events.k8s.io/v1
  events.k8s.io/v1beta1
  extensions/v1beta1
  flowcontrol.apiserver.k8s.io/v1beta1
  helm.cattle.io/v1
  k3s.cattle.io/v1
  metrics.k8s.io/v1beta1
  networking.k8s.io/v1
  networking.k8s.io/v1beta1
  node.k8s.io/v1
  node.k8s.io/v1beta1
  policy/v1
  policy/v1beta1
  rbac.authorization.k8s.io/v1
  rbac.authorization.k8s.io/v1beta1
  scheduling.k8s.io/v1
  scheduling.k8s.io/v1beta1
  storage.k8s.io/v1
  storage.k8s.io/v1beta1
  traefik.containo.us/v1alpha1
  v1
  ```

### apiVersion的含义

- **v1**：Kubernetes API的稳定版本，包含很多核心对象：pod、service等
- **apps/v1**：包含一些通用的应用层的api组合，如：Deployments, RollingUpdates, and ReplicaSets。

### 参考文档

[k8s-apiVersion](https://www.jianshu.com/p/9318374b7072)

[[K8S的apiVersion该用哪个](https://segmentfault.com/a/1190000017134399)](https://segmentfault.com/a/1190000017134399)

[官方文档](https://kubernetes.io/zh/docs/reference/using-api/)

[官方文档 理解k8s对象必需字段](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields)

