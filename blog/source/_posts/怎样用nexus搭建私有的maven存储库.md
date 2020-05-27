## 怎样用nexus搭建私有的maven存储库

1. 用nexus docker 镜像来启动nexus, nexus的镜像使用，可以参考[docker hub](https://hub.docker.com/r/sonatype/nexus3) 上的介绍。

2. 先创建个卷：`docker volume create --name nexus-data`

3. 然后启动nexus的容器：`docker run -d -p 8081:8081 --name nexus -v nexus-data:nexus:data sonatype/nexus3`。如果下载nexus比较慢的话，可以用国内的镜像源，修改/etc/docker/daemon.json,
    `{"registry-mirrors": [ "加速地址" ], "insecure-registries": [] }{ "registry-mirrors": [ "加速地址" ], "insecure-registries": [] }`

4. 然后等待nexus 容器起来，会有一会的延迟，然后访问http:nexus-host-ip:8081,点击登录，会弹出登录的按钮，默认密码在容器内，可用下面命令找到：

  先进入容器`docker exec -it container-id /bin/bash`
  然后`cat /nexus-data/admin-password`，把密码拷贝出来去登录，第一次会让你设置个自己的密码。

5. nexus现在已经默认创建好了maven-central, maven-release, maven-snapshot, maven-public repo, 其中public是个group的类型，已经包括了maven-{central,release,snapshot},我们配置settings.xml的时候用这个就好。

6. settings.xml的设置
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <localRepository/>
    <interactiveMode/>
    <usePluginRegistry/>
    <offline/>
    <pluginGroups/>
    <servers>
        <server>
            <id>nexus-snapshots</id>
            <username>admin</username>
            <password>XXXXXXXX</password>
        </server>
        <server>
            <id>nexus-releases</id>
            <username>admin</username>
            <password>XXXXXXXX</password>
        </server>
    </servers>
    <mirrors>
        <mirror>
            <id>central</id>
            <mirrorOf>*</mirrorOf>
            <name>central</name>
            <url>http://nexus-host-ip:8081/repository/maven-public/</url>
        </mirror>
    </mirrors>
    <proxies/>
    <profiles/>
    <activeProfiles/>
</settings>
```
7. 然后是设置pom, 如果只是相从nexus下载jar,只用添加下面的:
```xml
<repositories>
    <repository>
        <id>maven-group</id>
        <url>http://nexus-host-ip:8081/repository/maven-public/</url>
    </repository>
</repositories>
```
8. 如果想向nexus私服发布jar,需要添加下面的：
```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://nexus-host-ip:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
    <repository>
        <id>nexus-releases</id>
        <url>http://nexus-host-ip:8081/repository/maven-releases/</url>
    </repository>
</distributionManagement>

```