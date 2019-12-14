---
title: "使用maven插件构建docker镜像"
date: 2017/12/22 21:22:00
---
# 为什么要用插件
主要还是自动化的考虑，如果额外使用Dockerfile进行镜像生成，可能会需要自己手动指定jar/war位置，并且打包和生成镜像间不同步，带来很多琐碎的工作。  

# 插件选择
使用比较多的是spotify的插件:https://github.com/spotify/docker-maven-plugin  
和 https://github.com/spotify/dockerfile-maven。  
但这里我选择另一款插件：https://github.com/fabric8io/docker-maven-plugin。
因为他文档比较详细，在使用上也比较方便。  
文档地址：https://dmp.fabric8.io/  

# 示例  
这里使用一个spring boot项目，只有一个最简单的`HelloController`，如下：
```java
@RestController
public class HelloController {
    @GetMapping("/")
    public String hello() {
        return "hello";
    }
}
```
`pom.xml`改动如下:
```xml
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.23.0</version>
    <configuration>
        <dockerHost>tcp://192.168.99.100:2376</dockerHost>
        <certPath>C:\Users\fairjm\.docker\machine\machines\default</certPath>
        <images>
            <image>
                <name>${project.name}:${project.version}</name>
                <build>
                    <from>openjdk:8-jre</from>
                    <args>
                        <JAR_FILE>${project.name}-${project.version}.jar</JAR_FILE>
                    </args>
                    <assembly>
                        <descriptorRef>artifact</descriptorRef>
                    </assembly>
                    <entryPoint>["java"]</entryPoint>
                    <cmd>["-jar","maven/${project.name}-${project.version}.jar"]</cmd>
                </build>
                <run>
                    <ports>
                        <port>8888:8080</port>
                    </ports>
                </run>
            </image>
        </images>
    </configuration>
</plugin>
```    
这里使用了在xml里写操作而不是使用Dockerfile的方式，个人感觉这种方式更加简单一点，不需要额外再维护一份文件，和Dockerfile相比使用的语法(注意entrypoint和cmd)也类似。  

接下来介绍一下configuration配置。  
`dockerHost`和`certPath`是连接docker使用，毕竟插件本身不包含docker和对应功能只是调用docker提供的API。  
这两个值在docker toolbox上可以通过`docker-machine env`获得。  
```bash
$ docker-machine env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="C:\Users\fairjm\.docker\machine\machines\default"
export DOCKER_MACHINE_NAME="default"
export COMPOSE_CONVERT_WINDOWS_PATHS="true"
# Run this command to configure your shell:
# eval $("D:\Docker Toolbox\docker-machine.exe" env)
```

image的build指定了构建相关的设置：  
- name指定image名，这里使用了项目名，tag使用项目版本；   
- from指定基于的image，和Dockerfile的FROM一致；  
- args和ARG一致（在这个例子中没有实际意义）；  
- assembly用来定义哪些文件进入镜像中，使用了插件已经定义好的行为，spring-boot生成的是fat jar不需要拷贝依赖所以选择了artifact。这个的配置比较丰富，可以查看文档获取更多的信息。  
- entryPoint和cmd也对应同样的Dockerfile命令。 

接着通过`mvn clean package docker:build`执行打包和build
```bash
[INFO] --- maven-jar-plugin:2.6:jar (default-jar) @ docker-test ---
[INFO] Building jar: D:\sts_workspace\docker-test\target\docker-test-0.0.1-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:1.5.9.RELEASE:repackage (default) @ docker-test ---
[INFO]
[INFO] --- docker-maven-plugin:0.23.0:build (default-cli) @ docker-test ---
[INFO] Copying files to D:\sts_workspace\docker-test\target\docker\docker-test\0.0.1-SNAPSHOT\build\maven
[INFO] Building tar: D:\sts_workspace\docker-test\target\docker\docker-test\0.0.1-SNAPSHOT\tmp\docker-build.tar
[INFO] DOCKER> [docker-test:0.0.1-SNAPSHOT]: Created docker-build.tar in 1 second
[INFO] DOCKER> [docker-test:0.0.1-SNAPSHOT]: Built image sha256:303c3
[INFO] DOCKER> [docker-test:0.0.1-SNAPSHOT]: Removed old image sha256:ea8a7
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```
完成打包，在对应连接的docker上也会出现这个镜像：  
```bash
$ docker image ls
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
docker-test                       0.0.1-SNAPSHOT      303c39c7d253        13 seconds ago      552MB
```  

run指定了运行参数 这里将host的8888和容器的8080绑定，可以使用`mvn docker:start`启动，访问8888端口即可连接到服务器。  
与其配对的是`mvn docker:stop`，可以停止并移除启动的容器。  

# 示例2  
上述使用xml的配置方式，这里再简单描述一下使用`Dockerfile`的配置方式。  
在进行一些操作的时候，可以发现使用xml会有些问题，比如指令的执行顺序。  
该插件xml的执行顺序和命令的定义顺序不一定一致，可能会带来一些问题，比如将`<user>`放于`<runCmds>`前但还是`<runCmds>`先触发，一些需要root权限的命令就会失败。  
比如这个issus(但不确定是feature还是bug，感觉是feature)：https://github.com/fabric8io/docker-maven-plugin/issues/913  

这时候就需要直接使用`Dockerfile`来进行配置。  
这里取一个实际的打成war的工程。  
插件配置如下：
```
<images>
    <image>
        <name>${project.name}:${project.version}</name>
        <build>
            <assembly>
                <descriptorRef>rootWar</descriptorRef>
            </assembly>
            <dockerFile>Dockerfile</dockerFile>
        </build>
        <run>
            <ports>
                <port>8888:8080</port>
            </ports>
        </run>
    </image>
</images>
```
这里更改了`descriptorRef`，换成`rootWar`，这会将target下的项目war拷贝到`maven\`下并且取名为`ROOT.war`。  
`Dockerfile`默认放置的位置是`src/main/docker`，我们在这里建对应的文件：  
```
FROM jetty
USER root
ENV JAVA_OPTIONS=-Xmx1g
RUN mkdir -p /root/xxx && touch /root/xxx/yyy && echo zz > /root/xxx/yyy
COPY maven/ /var/lib/jetty/webapps
```
基本和上面的配置类似，base image改为了jetty，查看jetty的`Dockerfile`可以发现他使用了一个新用户`jetty`，使用这个用户无法在root下建立目录，并且由于项目本身之前使用sudo执行的，所以为了能正常运行选择使用root用户。  
最后一步将`ROOT.war`拷贝到jetty的webapps目录下。    
  
关于`maven/`这个目录，在打包后，会在target下生成`target\docker\项目名\0.0.1-SNAPSHOT\build`，对应的`Dockerfile`和`maven\`就在这个目录下，实际执行的不是`src/main/docker/Dockerfile`，而是拷贝到上述目录下的`Dockerfile`，此外使用xml的方式也是在这个位置生成了一份`Dockerfile`（USER 总是被放置于最后...）。


# 更多  
本文简要说明了使用`fabric8`的docker maven插件进行构建运行相关的操作，该插件还有其他的功能可以通过上面的文档获取帮助。  

# 源码下载(示例1)
https://github.com/fairjm/spring-boot-docker
