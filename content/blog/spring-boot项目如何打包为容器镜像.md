+++
title = "springBoot打包容器镜像的多种方式实践"
date = "2022-03-03"
description = ""
toc = true
tags = [
    "spring",
    "云原生",
]
+++

## 方式一. maven docker各自运作

1. 使用maven打包为可执行jar包.
2. 编写Dockerfile, `复制jar, 运行jar包`.

## 方式二. maven docker各自运行, 手动分层

1. 使用maven打包为可执行jar包.
2. 解压缩jar包, 并放置到不同层目录中. 
   `mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)`
3. 编写Dockerfile, 复制各层的文件, 运行main方法.
    ```dockerfile
    FROM beevelop/java:latest
    COPY target/dependency/BOOT-INF/lib /app/lib
    COPY target/dependency/META-INF /app/META-INF
    COPY target/dependency/BOOT-INF/classes /app
    ENTRYPOINT ["java","-cp","app:app/lib/*","com.example.springbootdocker.SpringBootDockerApplication"]
    ```

## 方式三. 在maven执行打包容器镜像.

1. pom.xml中配置容器参数
2. 执行`mvn spring-boot:build-image`打包.
3. 如果配置了registry, 会自动push
可在pom.xml中配置容器参数.
可在mvn 命令中传入容器参数.

docker运行时, 可以通过`-e`传递springboot运行参数.例如:
`docker run -e "SPRING_PROFILES_ACTIVE=prod" -p 8080:8080 -t springio/gs-spring-boot-docker`

## 方式四. 在maven执行打包容器镜像, 自动分层.

在spring-boot-maven-plugin中配置layers参数. 本质上只是打包时,添加了一个layers.idx文件.
```xml
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <layers>
                        <enabled>true</enabled>
                    </layers>
                </configuration>
			</plugin>
		</plugins>
	</build>
```

## 实践建议
spring-boot的`spring-boot-maven-plugin`使用了buildpacker来进行构建镜像, 无需做任何配置, 是最简单的方式, 但是由于国内网络,导致buildpacker相关组件无法下载, 所以该方式不能正常使用.
建议构建流程如下:
1. 通过spring-boot的分层工具, 定义分层.
2. mvn package 打出jar包
3. 使用spring-boot分层工具,解包.
4. 在Dockerfile中分别复制每层

自定义分层
```xml
<layers xmlns="http://www.springframework.org/schema/boot/layers"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/boot/layers
                          https://www.springframework.org/schema/boot/layers/layers-2.7.xsd">
    <application>
        <into layer="spring-boot-loader">
            <include>org/springframework/boot/loader/**</include>
        </into>
        <into layer="application" />
    </application>
    <dependencies>
        <into layer="snapshot-dependencies">
            <include>*:*:*SNAPSHOT</include>
        </into>
        <into layer="company-dependencies">
            <include>com.bizmatics:*</include>
        </into>
        <into layer="dependencies"/>
    </dependencies>
    <layerOrder>
        <layer>dependencies</layer>
        <layer>spring-boot-loader</layer>
        <layer>snapshot-dependencies</layer>
        <layer>company-dependencies</layer>
        <layer>application</layer>
    </layerOrder>
</layers>

```

```dockerfile
...
...
# 解包
RUN mkdir -p helloworld-controller/target/extracted && (java -Djarmode=layertools -jar helloworld-controller/target/*.jar extract --destination helloworld-controller/target/extracted)
# 复制
ARG EXTRACTED=/workspace/app/helloworld-controller/target/extracted
COPY --from=build ${EXTRACTED}/dependencies/ ./
COPY --from=build ${EXTRACTED}/spring-boot-loader/ ./
COPY --from=build ${EXTRACTED}/company-dependencies/ ./
COPY --from=build ${EXTRACTED}/snapshot-dependencies/ ./
COPY --from=build ${EXTRACTED}/application/ ./
# 运行
ENTRYPOINT ["java","org.springframework.boot.loader.JarLauncher"]
```

## 可用于java打包的多种方案
* Cloud Native Buildpacks（Spring Boot 2.3+ 版本开始支持）
* Google 的 jib-maven-plugin
* fabric8 和 spotify 的 docker-maven-plugin

## Jib
使用宿主机进行build, 然后使用Dockerfile构建镜像.
