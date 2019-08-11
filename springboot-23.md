---
title: 学习Spring Boot：（二十三）Spring Boot 中使用 Docker
date: 2018-03-13 22:12:03
tags: [Spring Boot,Docker]
categories: 学习笔记
---

### 前言
简单的学习下怎么在 Spring Boot 中使用 Docker 进行构建，发布一个镜像，现在我们通过远程的 docker api 构建镜像，运行容器，发布镜像等操作。

这里只介绍两种方式：

1. 远程命令 api （需要知道 Docker 命令）
2. maven 插件 （不需要了解 Docker 命令）

<!--more-->

### 开启 Docker api 远程访问

开启 docker api 远程操作的功能，
例如，centos 7 中在 `/usr/lib/systemd/system/docker.service`，文件中，修改 `ExecStart` 的参数：
```bash
ExecStart=/usr/bin/dockerd  -H tcp://0.0.0.0:2375  -H unix:///var/run/docker.sock
```
端口自定义设置即可。

重载所有修改过的配置文件，并且重启 docker，
```bash
systemctl daemon-reload    
systemctl restart docker.service 
```

> 需要注意的是，由于没有密码登陆任何权限验证，外网或者生产环境需要上证书使用。

### 命令方式构建镜像

这种方式其实非常简单，就是需要懂得 docker 命令，才能操作。

经过上面开启 Docker Api 后，我们可以使用网络环境操作 Docker 引擎了。

1. 新建 `Dockerfile` 构建镜像文件，新创建一个文件夹，专门放构建镜像需要的文件，我创建的是 `/src/docker/`

   ```dockerfile
   FROM java:8
   EXPOSE 8080

   VOLUME /tmp
   ADD springboot-docker.jar app.jar
   ENTRYPOINT ["java","-jar","/app.jar"]
   ```

2. 执行 maven 命令 ，将项目打包 `mvn clean package --DskipTests`，然后将打好的 jar 包，也放入到 `Dockerfile`项目目录中。

3. 然后进入 `src/docker` 目录下执行 ：

   ```bash
   docker -H tcp://xxx.xxx.xxx.xxx:2375 build -t test .
   ```

   开始构建镜像：

   ```bash
   Sending build context to Docker daemon  31.74MB
   Step 1/5 : FROM java:8
    ---> d23bdf5b1b1b
   Step 2/5 : EXPOSE 8080
    ---> Using cache
    ---> 060a43a42146
   Step 3/5 : VOLUME /tmp
    ---> Using cache
    ---> b4f88fde6181
   Step 4/5 : ADD springboot-docker.jar app.jar
    ---> 3a40188825b0
   Step 5/5 : ENTRYPOINT ["java","-jar","/app.jar"]
    ---> Running in ab093916fc4c
   Removing intermediate container ab093916fc4c
    ---> 45a3966feb60
   Successfully built 45a3966feb60
   Successfully tagged test:latest

   ```

   

### 使用 docker-maven-plugin构建镜像 

#### 在 maven 项目下加入 `docker-maven-plugin` 

```xml
            <!--打包docker插件相关参数的配置-->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.14</version>
                <configuration>
                    <!--打包的镜像名-->
                    <imageName>${project.groupId}/${project.artifactId}</imageName>
                    <!--Dockerfile文件位置，以项目的 root 目录为根节点，建议到单独建立一个目录-->
                    <dockerDirectory>./src/docker/</dockerDirectory>
                    <!--Docker 远程的 API 地址及端口-->
                    <dockerHost>http://xxx.xxx.xxx.199:2375</dockerHost>
                    <imageTags>
                        <imageTag>latest</imageTag>
                    </imageTags>
                    <!--执行构建docker镜像的时候需要哪些文件，springboot项目指定 打包好的jar 镜像就好-->
                    <resources>
                        <resource>
                            <!--这里指定的文件是target中的jar文件-->
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
```
#### 创建 Dockerfile
需要跟`pom.xml` 上面配置的路径保持一致，所以我的路径是 `${baseProjectFolder}/src/docker`的文件夹下新建一个文件 `Dockerfile`，添加构建 docker 相关命令参数：
```dockerfile
FROM java:8
EXPOSE 8080

VOLUME /tmp 
ADD springboot-docker.jar app.jar # 根据打包的jar 包文件名进行修改
ENTRYPOINT ["java","-jar","/app.jar"]
```
#### 打包
在应用的根目录下执行命令（打包加 dokcer build）：

```bash
$ mvn clean package docker:build -DskipTests
```

比如使用我的工程，进行打包后完成了 docker 的构建的信息：

```bash
[INFO] Building image com.wuwii/springboot-docker
Step 1/5 : FROM java:8

 ---> d23bdf5b1b1b
Step 2/5 : EXPOSE 8080

 ---> Running in b7936baae57f
Removing intermediate container b7936baae57f
 ---> 060a43a42146
Step 3/5 : VOLUME /tmp

 ---> Running in 65e2b8ac44d3
Removing intermediate container 65e2b8ac44d3
 ---> b4f88fde6181
Step 4/5 : ADD springboot-docker.jar app.jar

 ---> aa3762cda143
Step 5/5 : ENTRYPOINT ["java","-jar","/app.jar"]

 ---> Running in d9f5f63b9736
Removing intermediate container d9f5f63b9736
 ---> 622a7d1e315c
ProgressMessage{id=null, status=null, stream=null, error=null, progress=null, progressDetail=null}
Successfully built 622a7d1e315c
Successfully tagged com.wuwii/springboot-docker:latest

```

### 使用镜像

1. 进入安装 docker 的主机中，使用命令查看镜像（`IMAGE ID` 和上面是一致的）：

   ```bash
   $ docker image ls com.wuwii/springboot-docker
   REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
   com.wuwii/springboot-docker   latest              622a7d1e315c        22 minutes ago      659MB
   ```

2. 运行容器：

   ```bash
   $ docker run -d -p 8080:8080 --name learn  com.wuwii/springboot-docker

   180fe4a7ddfc10c0cf2c37649ae1628e804564bfe1594ef05840e707801e6da3
   ```

   监听 8080 端口，测试是否成功。



### 服务编排 compose

一般的我们的 WEB 项目会使用到很多外部工具，例如 Redis ，MYSQL, ES等，如果一个一个启动搭建部署，太麻烦了，还要测试如果把这个一套环境拿到别的地方还能用吗？

使用服务编排可以避免这些坑。

加入我们的项目中增加了 Mysql 的数据库，在根目录新建一个 `docker-compose.yml`：

```yaml
version: '3'
services:
  web:
    depends_on:
      - db
    ports:
      - "8080:8080" # 建议加上引号，如果单独两位数的数字，可能出现解析问题
    restart: always
   # build:
    #  context: ./src/docker # Dockerfile 文件的目录，可以远程地址，绝对 or 相对
     # dockerfile: Dockerfile # 如果你的 Dockerfile 重命名了，需要指定
    image: test:latest
    environment:
      DB_HOST: db:3306
      DATABASE: learn
      DB_USERNAME: root # 测试用下 root
      DB_PASSWORD: 123456 #  # 建议使用 secret

  db:
    image: mysql:5.7
    volumes:
        - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: learn
      MYSQL_USER: kronchan
      MYSQL_PASSWORD: 123456

volumes:
  db_data:  # 使用的数据卷必须声明
```

上面我使用的是前面已经构建好的镜像，然后执行的编排，更好的是直接使用 `build ` 让它自己编排服务。

系统配置文件`application.yml`使用缺省值的方式，不影响开发的使用：

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST:localhost:3306}/${DATABASE:learn}?useSSL=false&allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
    username: ${DB_USERNAME:root}
    password: ${DB_PASSWORD:123456}
    driver-class-name: com.mysql.jdbc.Driver
  jpa:
    show-sql: true
    database: mysql
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL57Dialect # 方言根据 数据库版本选择吧
```



也可以使用不同的 `spring.profiles`指定不同的环境，在 `docker-compose.yml` 中覆盖执行命令指定环境也是常见做法的：`command: mvn clean spring-boot:run -Dspring-boot.run.profiles=xxx`



最后启动，在 `docker-compose.yml`目录下执行 ： `docker-compose up`

关闭服务 `docker-compose down`



#### 注意

docker-compose 顺序的问题，这个是开始学习编排的时候需要注意的问题，如果上面的服务编排中 mysql 启动的慢， web 项目就会启动失败，它启动的时候不知道被依赖的服务是否启动完成，就会出现这样的问题。

解决的办法有以下几种：

- 足够的容错和重试机制，比如连接数据库，在初次连接不上的时候，服务消费者可以不断重试，直到连接上位置
- docker-compose拆分，分成两部分部署，将要先启动的服务放在一个docker-compose中，后启动的服务放在两一个docker-compose中，启动两次，两者使用同一个网络。
- 同步等待，使用`wait-for-it.sh`或者其他`shell`脚本将当前服务启动阻塞，直到被依赖的服务加载完毕
  `wait-for-it`的github地址为：[wait-for-it](https://link.jianshu.com?t=https://github.com/vishnubob/wait-for-it) 



### 总结

1. 主要是写 Dockerfile 的时候最好单独的拿出一个文件夹来放它，我开始的时候就是直接放在项目的根路径，结果构建镜像的时候总是出现了将其他的文件也一起复制到了 Docker 目录中，WINDOW下使用 maven 插件操作这个需要注意这个上下文环境，不然很容易将一个磁盘的文件都拷贝进来了，初学者血的教训。解决办法就是单独创建一个文件夹，将需要的东西单独放置，就不用考虑这么多问题。


[代码](https://github.com/kaimz/learning-code/tree/master/springboot-docker)
