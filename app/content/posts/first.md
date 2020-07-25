+++
title = "构建 Go 应用的一个示例"
date = "2019-01-25"
author = "Gavin"
cover = "img/hello.jpg"
description = "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nullam nec interdum metus. Aenean rutrum ligula sodales ex auctor, sed tempus dui mollis. Curabitur ipsum dui, aliquet nec commodo at, tristique eget ante."
+++

使用多阶段构建
多阶段构建可以有效减小镜像体积，特别是对于需编译语言而言，一个应用的构建过程往往如下

安装编译工具
安装第三方库依赖
编译构建应用

而在前两步会有大量的镜像体积冗余，使用多阶段构建可以避免这一问题
这是构建 Go 应用的一个示例
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
复制代码这是构建前端应用的一个示例，可以参考 如何使用 docker 高效部署前端应用
FROM node:10-alpine as builder

ENV PROJECT_ENV production
ENV NODE_ENV production

# http-server 不变动也可以利用缓存
WORKDIR /code

ADD package.json /code
RUN npm install --production

ADD . /code
RUN npm run build

# 选择更小体积的基础镜像
FROM nginx:10-alpine
COPY --from=builder /code/public /usr/share/nginx/html
复制代码避免安装不必要的包
减小体积，减少构建时间。如前端应用使用 npm install --production 只装生产环境所依赖的包。
一个容器只做一件事
如一个web应用将会包含三个部分，web 服务，数据库与缓存。把他们解耦到多个容器中，方便横向扩展。如果你需要网络通信，则可以将他们至于一个网络下。
