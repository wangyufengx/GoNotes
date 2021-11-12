# [docker API] docker client 之image接口（Golang）

docker 除了命令模式，也提供了通过api来进行一些命令行操作。

### 连接docker engine 

docker engine api支持使用本地引擎或远程引擎来进行操作。

- 本地引擎连接

```go
//建立连接
dockerClient, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
if err != nil {
	return err
}
```

- 远程引擎连接

```GO
//建立连接
host := "tcp://172.16.12.1:2375"
dockerClient, err := client.NewClientWithOpts(client.WithHost(host),client.WithAPIVersionNegotiation())
if err != nil {
	return err
}
```

### 拉取镜像

下面以docker hub上的`alpine:3.13.6`镜像为例：

```go
pull, err := dockerClient.ImagePull(ctx, "alpine:3.13.6", types.ImagePullOptions{})
if err != nil {
    return err
}
//标准输出
io.Copy(os.Stdout, pull)
```

### 按条件查询镜像信息

**查询指定镜像的命令行方式为`docker images --filter=reference="alpine:3.13.6"`**, 那么docker引擎的api该怎么实现这一操作呢，示例如下：

```go
//筛选条件
args := filters.NewArgs()
args.Add("reference", "alpine:3.13.6")
//查询
list, err := dockerClient.ImageList(ctx, types.ImageListOptions{
    All:     false,
    Filters: args,
})
if err != nil {
		return err
	}
fmt.Println(list)
```

### 推送镜像

镜像推送到私库或公库可能需要用户名密码，示例如下：

```go
authConfig := types.AuthConfig{
    Username:      "用户名",
    Password:      "密码",
    ServerAddress: "私库",
}
bytes, err := json.Marshal(authConfig)
if err != nil {
    return err
}

push, err := dockerClient.ImagePush(ctx, "alpine:3.13.6", types.ImagePushOptions{
    RegistryAuth: base64.URLEncoding.EncodeToString(bytes),
})
if err != nil {
    return err
}
io.Copy(os.Stdout, push)
```

### 镜像导出

docker引擎api通过调用`ImageSave`方法来导出**镜像流**，需要我们自行创建文件保存。

`ImageSave`方法接收的是一个**镜像ID数组**。

```go
//导出镜像流
save, err := dockerClient.ImageSave(ctx, []string{list[0].ID})
if err != nil {
    return err
}
//存储
if err = utils.GenerateFile("./alpine.tar.gz", save); err != nil {
    return err
}
```

生成镜像包

```go
//生成镜像包
func GenerateFile(fileName string, data io.ReadCloser) error {
	file, err := os.Create(fileName)
	if err != nil {
		return err
	}
	defer file.Close()
	w := bufio.NewWriter(file)
	if _, err := io.Copy(w, data); err !=nil {
		return err
	}
	w.Flush()
	return nil
}
```

### 本文详细代码

https://github.com/wangyufengGoGoGo/GoLangStudy/blob/master/goStudy/docker/image.go

### 参考文档

https://pkg.go.dev/github.com/docker/docker

https://docs.docker.com/engine/api/v1.41/#tag/Image

https://docs.docker.com/engine/reference/commandline/images/

