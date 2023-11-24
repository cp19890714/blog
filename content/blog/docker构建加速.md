+++
title = "docker构建加速"
date = "2022-01-03"
description = ""
tags = [
    "docker",
    "云原生",
]
+++

https://docs.docker.com/build/cache/

https://juejin.cn/post/7022957854889869325

尽量使用缓存,优先执行`没有文件变动的命令`,让可缓存的命令在底层.

## 目标
1. 保持构建环境一致. 最好的方式是使用dockerFile进行构建, 保证jdk和maven版本完全一致.
2. 加速maven打包,以及dockerFile build的速度.

## maven构建加速

### 方式1. 提前解决依赖, 则COPY源码进行build
* copy pom.xml first, execute`mvn dependency:go-offline`, resolve all dependencies, this layer will be cached as long as pom.xml not changed.
* then copy source code, execute `mvn package`
在多模块项目中, 可以按照目录结构把父模块和子模块的pom.xml分开COPY, 然后执行`mvn dependency:go-offline`.

该方案的问题在于:
1. 如果pom.xml文件有变更, 则会导致所有依赖都重新下载, 无法利用缓存.
2. 私仓配置动态传入

### 方式2. 对构建过程中的依赖进行缓存挂载
`RUN --mount=type=cache,target=/root/.m2  mvn clean package -Dmaven.test.skip=true `
该样例中对`/root/.m2`目录进行缓存挂载.
第一次构建时较慢, 因为需要下载所有依赖, 后续构建就会很快, 即使pom.xml文件有变更, 也只会下载变更的依赖.

该方案的问题在于
1. 第一次构建需要下载所有依赖.
2. 私仓配置无法动态传入. 
   1. 如果需要配置私仓, 则需要自己构建base image, 并在base image中配置私仓, 然后使用该image作为build的基础镜像. 
   2. 或者把maven的settings.xml传入, 然后通过`mvn -s settings.xml package`指定settings.xml运行

### ~~方式3. 挂载宿主机的.m2目录到build阶段.~~
>该方案不可行, 因为.m2必须在dockerfile context中才能进行挂载, 但是把.m2放在context中会导致上下文非常大, 这会有性能问题. 另外在使用远程docker engine时,会把context传输到远程docker engine, context太大会导致问题.

`RUN --mount=type=bind,source=,target=/Users/chen/.m2 mvn clean package -Dmaven.test.skip=true`

该问题目前在讨论中, 没有结果: https://github.com/moby/buildkit/issues/1512

### 方式4. 使用三方构建工具的多阶段构建
例如shell脚本 jenkins gitlab-ci, 把maven构建和docker构建分开. 
1. 启动容器进行maven构建, 在`docker run `时, 可以挂载宿主机的.m2目录到容器中, 这样就可以利用宿主机的.m2缓存.
2. 把maven构建的产物复制到docker构建中.
3. 启动另外一个容器进行docker镜像构建.

### 方式5. 使用三方构建工具的缓存功能
例如gitlab-ci, 可以通过指定cache,来缓存相应的目录.

## spring boot 构建加速
  * unpack and layering the packaged jar
  * then copy every layer to docker image.

```shell
# jar包解压为多层
java -Djarmode=layertools -jar helloworld-controller/target/*.jar extract --destination helloworld-controller/target/extracted
```

```dockerfile
FROM ibm-semeru-runtimes:open-8u362-b09-jre-centos7
WORKDIR /app
ARG EXTRACTED=/workspace/app/helloworld-controller/target/extracted
COPY --from=build ${EXTRACTED}/dependencies/ ./
COPY --from=build ${EXTRACTED}/spring-boot-loader/ ./
COPY --from=build ${EXTRACTED}/company-dependencies/ ./
COPY --from=build ${EXTRACTED}/snapshot-dependencies/ ./
COPY --from=build ${EXTRACTED}/application/ ./
ENTRYPOINT ["java","org.springframework.boot.loader.JarLauncher"]
```

## CI/CD环境加速

在CI/CD中每次构建都会申请一个全新的环境进行构建,也意味着docker build的layer缓存无法被重复利用.

通过 `Cache backend`可以对构建缓存进行导入和导出

[Cache storage backends | Docker Documentation](https://docs.docker.com/build/cache/backends/)

> 该功能不能代替`--mount=type=cache/bind`, 它们的目的完全不同.
>
> * `–-mount`用于对构建中产生的产物相关数据进行缓存,这些数据与容器技术无关. 例如maven/npm 依赖. 即使该层的文件或命令发生变动, 缓存也不会失效.
> * `Cache backend`用于对构建中产品的容器相关layer进行缓存, 事实上, 如果该层的文件或命令发生变动,该层缓存会完全失效.
