# docker kong打包自定义插件

### Dockerfile
```Dockerfile

FROM docker.io/library/golang:1.18.3-alpine3.15 AS builder


RUN mkdir -p /tmp/go/src/ \
    && go env -w GOPROXY=https://goproxy.cn,direct

COPY plugins-code/ /tmp/go/src
COPY kong.conf.default /tmp/go/src/kong.conf.default

RUN cd /tmp/go/src/go-hello  \
    && go mod tidy \
    && go build -o go-hello \
    && ls /tmp/go/src/go-hello

FROM kong:2.8.1-alpine
USER root
COPY --from=builder  /tmp/go/src/go-hello/go-hello /usr/local/bin/
COPY --from=builder  /tmp/go/src/kong.conf.default /etc/kong/kong.conf.default
RUN sed -i 's|local plugins = {|local plugins = {\n  "go-hello",|g' /usr/local/share/lua/5.1/kong/constants.lua

```

### deploy

```deploy.sh
# 删除容器
docker rm -f `docker ps -a|grep kong:2.8.0-personalise|awk '{print $1}'`
#删除镜像
docker rmi -f `docker images --filter=reference='kong:2.8.0-personalise' | awk '{print $3}'`

# 进入容器
#  docker exec -it `docker ps |grep kong:2.8.0-personalise |awk '{print $1}'`  /bin/sh

chmod +x kong/install.sh

docker build -f Dockerfile . -t kong:2.8.0-personalise
```

### run

```bash
docker run --rm  \
    -p 8000:8000 \
    -p 8001:8001 \
    -p 8443:8443 \
    -p 8444:8444 \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_PORT=5432" \
    -e "KONG_PG_HOST={IP}" \
    -e "KONG_PG_DATABASE=kong" \
    -e "KONG_PG_USER=postgres" \
    -e "KONG_PG_PASSWORD=kong" \
    -e "KONG_PLUGINS=bundled, go-hello" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    kong:2.8.0-personalise kong migrations bootstrap -c /etc/kong/kong.conf.default

docker run -d --name kong --restart=no \
    -p 8000:8000 \
    -p 8001:8001 \
    -p 8443:8443 \
    -p 8444:8444 \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_PORT=5432" \
    -e "KONG_PG_HOST={IP}" \
    -e "KONG_PG_DATABASE=kong" \
    -e "KONG_PG_USER=postgres" \
    -e "KONG_PG_PASSWORD=kong" \
    -e "KONG_PLUGINS=bundled, go-hello" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    kong:2.8.0-personalise kong start -c /etc/kong/kong.conf.default
```
