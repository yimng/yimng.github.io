## 利用nexus3创建docker registry

### 1. 用nexus3提供的镜像来启动个nexus3的容器


1. 先创建个nexus使用的卷：
   `docker volume create --name nexus-data`

2. 然后启动nexus容器，注意要加下8443，8444，8445端口，以后docker 拉推镜像会用到8444，8445端口，8443是https用的

   ```
   docker run -d -p 8081:8081 -p 8443:8443 -p 8444:8444 -p 8445:8445 -v /etc/localtime:/etc/localtime -v nexus-data:nexus:data  --name nexus sonatype/nexus3
   ```

   

3. 然后等待nexus容器起来，会有一会的延迟，然后访问http:ip:8081,点击登录，会弹出登录的按钮，默认密码在容器内，可用下面命令找到：
   先进入容器 
   `docker exec -it container-id /bin/bash`
   然后
   `cat /nexus-data/admin-password`，把密码拷贝出来去登录，第一次会让你设置个自己的密码。




### 2. 配置nexus3 使用 https

参见[https://support.sonatype.com/hc/en-us/articles/217542177](https://support.sonatype.com/hc/en-us/articles/217542177)这篇文章
具体步骤：

1. 登录到nexus容器内：
   `docker exec -it nexus-container-id /bin/bash`

2. 在$data-dir/etc/ssl/keystore.jks创建java keystore 文件, $data-dir/为/nexus-data目录，其中，把${NEXUS_DOMAIN} 替换成nexus的域名，${NEXUS_IP_ADDRESS}替换成nexus的ip

   ```
   keytool -genkeypair -keystore keystore.jks -storepass password -keypass password -alias jetty -keyalg RSA -keysize 2048 -validity 5000 -dname "CN=*.${NEXUS_DOMAIN}, OU=Example, O=Sonatype, L=Unspecified, ST=Unspecified, C=US" -ext "SAN=DNS:${NEXUS_DOMAIN},IP:${NEXUS_IP_ADDRESS}" -ext "BC=ca:true"
   ```

   

3. 编辑$data-dir/etc/nexus.properties文件，增加一行
   `application-port-ssl=8443`
   8443是https用的端口，可以改为其它端口。
   把包含nexus-args的一行开头注释去掉，然后在逗号分隔的那些值里加入${jetty.etc}/jetty-https.xml
   修改后的那一行应该是这个样子：

   ```
   nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-https.xml,${jetty.etc}/jetty-requestlog.xml
   ```

   

4. (可选)如果你不想用http connector，可以把${jetty.etc}/jetty-http.xml去掉

5. 增加一行:
   `ssl.etc=${karaf.data}/etc/ssl`

6. 编辑 $install-dir/etc/jetty/jetty-https.xml
   正确设置 keystore 的密码，如果你用password作为密码，这一步就不是必须的了
   (建议)
   在包含<Set name=KeyStorePath">的这一行前面加上下面一行，例如如果你的PrivateKeyEntry 叫jetty, 增加下面一行：
   `<Set name="certAlias">jetty</Set>`

7. 重启nexus容器


### 3. 配置docker repository

配置docker repository，添加proxy, hosted, group 类型的docker repository, 其中 hosted,和group类型的用https, 端口用8445，8444

### 4. 在docker 客户端配置nexus使用的自签名的ca

如果是centos:

```
openssl s_client -showcerts -connect nexus-ip:8443 </dev/null 2>/dev/null | openssl x509 -outform PEM > /etc/pki/ca-trust/source/anchors/nexus-ip.crt
update-ca-trust
```

如果是在ubuntu:

```
openssl s_client -showcerts -connect nexus-ip:8443 </dev/null 2>/dev/null | openssl x509 -outform PEM > /usr/local/share/ca-certificates/nexus-ip.crt
update-ca-certificates
```




### 5. docker login nexus-ip:8445 

用户名密码为登录nexus的admin和相应的密码
如果push 或者pull的时候，出现no basic auth的错误，修改~/.docker/config里相应的端口号
