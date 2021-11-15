{% raw %}

# docker images

### 说明

镜像列表

### 用法

```bash
$ docker images [OPTIONS] [REPOSITORY[:TAG]]
```

### 拓展说明

默认`docker images`展示所有级别的镜像、存储库、标签和大小。

Docker 镜像具有中间层，通过允许缓存每个步骤来提高可重用性、减少磁盘使用并加速 `docker build`。默认情况下不显示这些中间层。

`SIZE`是镜像及其所有父镜像占用的累积空间。这也是 `docker save`镜像时创建的 Tar 文件的内容所使用的磁盘空间。

如果一个镜像有多个存储库名称或标签，它将被多次列出。 此单个镜像（可通过其匹配的 `IMAGE ID` 识别）仅使用一次列出的 `SIZE`。

有关此命令的示例用法，请参阅下面的[示例部分](# 示例)。

### 选项

| 名称，简写        | 默认 | 描述                             |
| ----------------- | ---- | -------------------------------- |
| `--all` , `-a`    |      | 展示所有镜像（默认隐藏中间镜像） |
| `--digests`       |      | 展示摘要                         |
| `--filter` , `-f` |      | 根据提供的条件过滤输出           |
| `--format`        |      | 使用 Go 模板打印漂亮的镜像       |
| `--no-trunc`      |      | 不要截断输出                     |
| `--quiet` , `-q`  |      | 仅显示图像 ID                    |

### 示例

#### 列出最近创建的镜像

```bash
$ docker images

REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
<none>                    <none>              77af4d6b9913        19 hours ago        1.089 GB
committ                   latest              b6fa739cedf5        19 hours ago        1.089 GB
<none>                    <none>              78a85c484f71        19 hours ago        1.089 GB
docker                    latest              30557a29d5ab        20 hours ago        1.089 GB
<none>                    <none>              5ed6274db6ce        24 hours ago        1.089 GB
postgres                  9                   746b819f315e        4 days ago          213.4 MB
postgres                  9.3                 746b819f315e        4 days ago          213.4 MB
postgres                  9.3.5               746b819f315e        4 days ago          213.4 MB
postgres                  latest              746b819f315e        4 days ago          213.4 MB
```

#### 按名称和标签列出镜像

该`docker images`命令采用一个可选`[REPOSITORY[:TAG]]`参数，该参数将列表限制为与该参数匹配的镜像。如果您指定 `REPOSITORY`但 no `TAG`，则该`docker images`命令会列出给定存储库中的所有镜像。

例如，要列出“java”存储库中的所有镜像，请运行以下命令：

```bash
$ docker images java

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
java                8                   308e519aac60        6 days ago          824.5 MB
java                7                   493d82594c15        3 months ago        656.3 MB
java                latest              2711b1d6f3aa        5 months ago        603.9 MB
```

该`[REPOSITORY[:TAG]]`值必须是“完全匹配”。这意味着，例如， `docker images jav`与图像不匹配`java`。

如果同时提供`REPOSITORY`和`TAG`，则仅列出与该存储库和标签匹配的镜像。要在标签为“8”的“java”存储库中查找所有本地镜像，您可以使用：

```bash
$ docker images java:8

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
java                8                   308e519aac60        6 days ago          824.5 MB
```

如果没有匹配项`REPOSITORY[:TAG]`，则列表为空。

```bash
$ docker images java:0

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

#### 镜像列表IDs全长

```bash
$ docker images --no-trunc

REPOSITORY                    TAG                 IMAGE ID                                                                  CREATED             SIZE
<none>                        <none>              sha256:77af4d6b9913e693e8d0b4b294fa62ade6054e6b2f1ffb617ac955dd63fb0182   19 hours ago        1.089 GB
committest                    latest              sha256:b6fa739cedf5ea12a620a439402b6004d057da800f91c7524b5086a5e4749c9f   19 hours ago        1.089 GB
<none>                        <none>              sha256:78a85c484f71509adeaace20e72e941f6bdd2b25b4c75da8693efd9f61a37921   19 hours ago        1.089 GB
docker                        latest              sha256:30557a29d5abc51e5f1d5b472e79b7e296f595abcf19fe6b9199dbbc809c6ff4   20 hours ago        1.089 GB
<none>                        <none>              sha256:0124422dd9f9cf7ef15c0617cda3931ee68346455441d66ab8bdc5b05e9fdce5   20 hours ago        1.089 GB
<none>                        <none>              sha256:18ad6fad340262ac2a636efd98a6d1f0ea775ae3d45240d3418466495a19a81b   22 hours ago        1.082 GB
<none>                        <none>              sha256:f9f1e26352f0a3ba6a0ff68167559f64f3e21ff7ada60366e2d44a04befd1d3a   23 hours ago        1.089 GB
tryout                        latest              sha256:2629d1fa0b81b222fca63371ca16cbf6a0772d07759ff80e8d1369b926940074   23 hours ago        131.5 MB
<none>                        <none>              sha256:5ed6274db6ceb2397844896966ea239290555e74ef307030ebb01ff91b1914df   24 hours ago        1.089 GB
```

#### 镜像列表显示摘要摘要

使用 v2 或更高版本格式的镜像具有称为`digest`. 只要用于生成镜像的输入不变，摘要值就是可预测的。要列出镜像摘要值，请使用以下`--digests`标志：

```bash
$ docker images --digests
REPOSITORY                         TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
localhost:5000/test/busybox        <none>              sha256:cbbf2f9a99b47fc460d422812b6a5adff7dfee951d8fa2e4a98caa0382cfbdbf   4986bf8c1536        9 weeks ago         2.43 MB
```

推送或拉取到 2.0 注册表时，`push`或`pull`命令输出包括镜像摘要。您可以`pull`使用摘要值。您还可以在`create`、`run`和`rmi`命令中通过摘要进行引用，以及`FROM`Dockerfile 中的镜像引用。

#### 过滤

过滤标志（`-f`或`--filter`）格式为“key=value”。如果有多个过滤器，则传递多个标志（例如，`--filter "foo=bar" --filter "bif=baz"`）

目前支持的过滤器有：

- dangling （布尔值 - 真或假）
- label （`label=<key>`或`label=<key>=<value>`）
- before ( `<image-name>[:<tag>]`, `<image id>`or `<image@digest>`) - 过滤在给定 id 或引用之前创建的镜像
- since( `<image-name>[:<tag>]`, `<image id>`or `<image@digest>`) - 过滤自给定 id 或引用以来创建的镜像
- reference （镜像reference 的模式） - 过滤参考与指定模式匹配的图像

##### 显示未标记的镜像（dangling ）

```bash
$ docker images --filter "dangling=true"

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              8abc22fbb042        4 weeks ago         0 B
<none>              <none>              48e5f45168b9        4 weeks ago         2.489 MB
<none>              <none>              bf747efa0e2f        4 weeks ago         0 B
<none>              <none>              980fe10e5736        12 weeks ago        101.4 MB
<none>              <none>              dea752e4e117        12 weeks ago        101.4 MB
<none>              <none>              511136ea3c5a        8 months ago        0 B
```

这将显示作为镜像树（不是中间层）的叶子的未标记镜像。当新构建的镜像`repo:tag`从镜像 ID 中删除，将其保留为`<none>:<none>`或未标记时，就会出现这些镜像 。如果在容器当前正在使用镜像时尝试删除镜像，则会发出警告。通过拥有这个标志，它允许批量清理。

您可以结合使用它`docker rmi ...`：

```
$ docker rmi $(docker images -f "dangling=true" -q)

8abc22fbb042
48e5f45168b9
bf747efa0e2f
980fe10e5736
dea752e4e117
511136ea3c5a
```

如果存在使用这些未标记镜像的任何容器，Docker 会向您发出警告。

##### 显示带有给定标签的镜像

`label`过滤器匹配基于存在的镜像`label`单独或`label`和值。

以下过滤器匹配带有`com.example.version`标签的镜像，而不管其值如何。

```
$ docker images --filter "label=com.example.version"

REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
match-me-1          latest              eeae25ada2aa        About a minute ago   188.3 MB
match-me-2          latest              dea752e4e117        About a minute ago   188.3 MB
```

以下过滤器匹配带有`com.example.version`标签和`1.0`值的镜像。

```
$ docker images --filter "label=com.example.version=1.0"

REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
match-me            latest              511136ea3c5a        About a minute ago   188.3 MB
```

在此示例中，`0.1`由于未找到匹配项，因此使用该值返回一个空集。

```
$ docker images --filter "label=com.example.version=0.1"
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
```

##### 按时间过滤镜像

该`before`过滤器只示出了与镜像给定id或参考镜像之前创建的。例如，有这些镜像：

```
$ docker images

REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
image1              latest              eeae25ada2aa        4 minutes ago        188.3 MB
image2              latest              dea752e4e117        9 minutes ago        188.3 MB
image3              latest              511136ea3c5a        25 minutes ago       188.3 MB
```

过滤`before`会给出：

```
$ docker images --filter "before=image1"

REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
image2              latest              dea752e4e117        9 minutes ago        188.3 MB
image3              latest              511136ea3c5a        25 minutes ago       188.3 MB
```

过滤`since`会给出：

```
$ docker images --filter "since=image3"
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
image1              latest              eeae25ada2aa        4 minutes ago        188.3 MB
image2              latest              dea752e4e117        9 minutes ago        188.3 MB
```

##### 按参考过滤镜像

该`reference`过滤器只示出其参考与指定模式匹配的镜像。

```
$ docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              e02e811dd08f        5 weeks ago         1.09 MB
busybox             uclibc              e02e811dd08f        5 weeks ago         1.09 MB
busybox             musl                733eb3059dce        5 weeks ago         1.21 MB
busybox             glibc               21c16b6787c6        5 weeks ago         4.19 MB
```

过滤`reference`会给出：

```
$ docker images --filter=reference='busy*:*libc'

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             uclibc              e02e811dd08f        5 weeks ago         1.09 MB
busybox             glibc               21c16b6787c6        5 weeks ago         4.19 MB
```

使用多个过滤`reference`将给出匹配 A 或 B：

```
$ docker images --filter=reference='busy*:uclibc' --filter=reference='busy*:glibc'

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             uclibc              e02e811dd08f        5 weeks ago         1.09 MB
busybox             glibc               21c16b6787c6        5 weeks ago         4.19 MB
```

### 格式化输出

格式化选项 ( `--format`) 将使用 Go 模板漂亮地打印容器输出。

下面列出了 Go 模板的有效占位符：

| 占位符          | 描述                     |
| :-------------- | :----------------------- |
| `.ID`           | 图像标识                 |
| `.Repository`   | 镜像仓库                 |
| `.Tag`          | 图片标签                 |
| `.Digest`       | 图片摘要                 |
| `.CreatedSince` | 自创建图像以来经过的时间 |
| `.CreatedAt`    | 创建图像的时间           |
| `.Size`         | 镜像磁盘大小             |

使用该`--format`选项时，该`image`命令将完全按照模板声明的方式输出数据，或者在使用该 `table`指令时，还将包括列标题。

以下示例使用没有标题的模板，`ID`并为所有镜像输出`Repository`以冒号 ( `:`)分隔的 和条目：

```
$ docker images --format "{{.ID}}: {{.Repository}}"
77af4d6b9913: <none>
b6fa739cedf5: committ
78a85c484f71: <none>
30557a29d5ab: docker
5ed6274db6ce: <none>
746b819f315e: postgres
746b819f315e: postgres
746b819f315e: postgres
746b819f315e: postgres
```

要以表格格式列出所有镜像及其存储库和标签，您可以使用：

```
$ docker images --format "table {{.ID}}\t {{.Repository}}\t {{.Tag}}"
IMAGE ID            REPOSITORY                TAG
77af4d6b9913        <none>                    <none>
b6fa739cedf5        committ                   latest
78a85c484f71        <none>                    <none>
30557a29d5ab        docker                    latest
5ed6274db6ce        <none>                    <none>
746b819f315e        postgres                  9
746b819f315e        postgres                  9.3
746b819f315e        postgres                  9.3.5
746b819f315e        postgres                  latest
```

### 父命令

| 命令                                                         | 描述                    |
| :----------------------------------------------------------- | :---------------------- |
| [docker](https://docs.docker.com/engine/reference/commandline/docker/) | Docker CLI 的基本命令。 |

{% endraw %}
