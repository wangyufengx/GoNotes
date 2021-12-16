# k8s镜像拉取策略

### 镜像拉取策略

容器的 `imagePullPolicy` 和镜像的标签会影响镜像的拉取。

- IfNotPresent

  只有当镜像在本地不存在时才会拉取。

- Always

  每当 kubelet 启动一个容器时，kubelet 会查询容器的镜像仓库， 将名称解析为一个镜像[摘要](https://docs.docker.com/engine/reference/commandline/pull/#pull-an-image-by-digest-immutable-identifier)。 如果 kubelet 有一个容器镜像，并且对应的摘要已在本地缓存，kubelet 就会使用其缓存的镜像； 否则，kubelet 就会使用解析后的摘要拉取镜像，并使用该镜像来启动容器。

- Never

  不会尝试获取镜像。如果镜像已经以某种方式存在本地， kubelet 会尝试启动容器；否则，会启动失败。 

### 未设置 `imagePullPolicy`时的拉取策略

- 如果你省略了 `imagePullPolicy` 字段，并且容器镜像的标签是 `:latest`， `imagePullPolicy` 会自动设置为 `Always`。
- 如果你省略了 `imagePullPolicy` 字段，并且没有指定容器镜像的标签， `imagePullPolicy` 会自动设置为 `Always`。
- 如果你省略了 `imagePullPolicy` 字段，并且为容器镜像指定了非 `:latest` 的标签， `imagePullPolicy` 就会自动设置为 `IfNotPresent`。

> 容器的 `imagePullPolicy` 的值总是在对象初次 *创建* 时设置的，如果后来镜像的标签发生变化，则不会更新。

