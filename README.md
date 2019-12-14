该镜像用于作为一个中间层，将输出的二进制文件压缩的更小

```Dockerfile
FROM ssrcoder/upx as compress

COPY --from=builder output .

RUN ./upx -9 -o app output
```

下面是一个完整的例子，将go语言程序编译、压缩、然后运行

```Dockerfile
# 1. 编译 Go 程序
# 设置Go编译器版本，默认为最新版
ARG GO_VERSION=latest

FROM golang:${GO_VERSION} AS builder

# 镜像作者信息
MAINTAINER SsrCoder

# 镜像元数据，可以使用 docker inspect 查看
LABEL author="SsrCoder@outlook.com"

# 设置工作目录
WORKDIR $GOPATH/src/code.ssrcoder.com/uploader

# 将当前文件夹下的文件拷贝到工作目录
COPY . .

# 静态链接编译Go程序
RUN GO111MODULE=on GOPROXY="https://goproxy.cn,direct" CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "-s -w" -o output/output .

####################################################
# 2. 压缩
FROM ssrcoder/upx as compress

COPY --from=builder /go/src/code.ssrcoder.com/uploader/output/output .

RUN ./upx -9 -o app output
####################################################
# 3. 运行
FROM alpine

# 镜像作者信息
MAINTAINER SsrCoder

# 镜像元数据，可以使用 docker inspect 查看
LABEL author="SsrCoder@outlook.com"

EXPOSE 80

ENV APP_RUN_DIR /data

WORKDIR $APP_RUN_DIR

RUN apk update \
        && apk --no-cache add wget ca-certificates \
        && apk add -f --no-cache git \
        && apk add -U tzdata \
        && ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime

COPY --from=compress /data/app .

CMD ["./app"]
```