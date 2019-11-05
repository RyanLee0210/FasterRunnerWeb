# FasterRunnerWeb

## 介绍
> 基于HttpRunner V1.5.15的Web系统docker-compose一键化部署项目
>
> 本项目依赖github地址：
>
> FasterRunner：https://github.com/RyanLee0210/FasterRunner
>
> FasterWeb：https://github.com/RyanLee0210/FasterWeb
>
> Fork-github地址：
>
> FasterRunner：https://github.com/httprunner/FasterRunner
>
> FasterWeb：https://github.com/httprunner/FasterWeb


### 项目结构
```
├── docker-compose.yml    # docker-compose配置文件
├── README.md    # 项目说明文件
├── FasterRunner    # 系统后端Django项目
│   ├── Dockerfile    # FasterRunner镜像docker脚本文件
│   ├── FasterRunner
│   ├── fastrunner
│   ├── fastuser
│   ├── manage.py
│   ├── README.md
│   ├── requirements.txt    # Django项目所需依赖
│   ├── start.sh    # celery启动脚本
│   ├── templates
└── FasterWeb    # 前端Web项目
    ├── conf.d    # nginx相关配置文件
    │   └── yelda.conf
    ├── dist    # 前端Web打包后的目标文件
    ├── Dockerfile    # FasterWeb镜像docker脚本文件
    └── README.md

```

### 相关技术：

* Docker：是一个开源的应用容器引擎。
* Docker-Compose：是 Docker 官方编排（Orchestration）项目之一，负责快速的部署分布式应用。定位是**定义和运行多个 Docker 容器的应用**（Defining and running multi-container Docker applications）。
* Django：一个使用 Python 编写的Web 应用程序框架。
* Nginx：一个高性能的HTTP和反向代理web服务器，同时也提供了IMAP/POP3/SMTP服务。
* Celery：是一个使用Python编写的异步任务的调度工具。



## 项目说明：

#### 目录说明

- `FasterRunner/`目录下是后端代码，也是后端启动目录。该应用监听的是`8000`端口。
- `FasterWeb/conf.d/`目录下的`yelda.conf`，是`Nginx`的配置文件。在这个文件中，我配置了`Nginx`监听`8082`端口，并且将`/api/`请求转发到`8000`端口，即后端应用运行的端口。（我使用的是`daocloud.io/library/nginx:1.13.0-alpine`版本镜像，旧版本下配置文件应该放置在`sites-enabled`目录下）。
- `FasterWeb/dist/`目录下是前端文件。

#### Dockerfile分别说明

- 后端容器的`Dockerfile`如下：

  ```bash
  FROM python:3.5.2
  
  MAINTAINER liwaiqiang
  
  ENV LANG C.UTF-8
  ENV TZ=Asia/Shanghai
  
  WORKDIR /opt/workspace/FasterRunner/
  
  COPY . .
  
  RUN  pip3 install -r ./requirements.txt
  
  EXPOSE 5000
  
  CMD bash ./start.sh
  ```
  
  完成了**拷贝代码，切换工作目录，安装依赖，设置启动命令**的任务。
  
- 负载均衡容器的`Dockerfile`如下：

  ```bash
  FROM daocloud.io/library/nginx:1.13.0-alpine
  
  MAINTAINER liwaiqiang
  
  COPY dist/ /static/
  COPY conf.d/ /etc/nginx/conf.d
  ```
  
  完成了**拷贝前端代码，拷贝配置文件**的任务。
  
  yelda.conf文件说明：
  
  **注意IP需要对应做修改**
```javascript
server {
      listen 8082;
  
      # SSL configuration
      # listen 443 ssl;
      server_name 192.168.1.100;
  
      root /static/;
  
      #access_log /log/access_log;
      #error_log /log/error_log;
  
      location /api {
          proxy_set_header Connection "";
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Host $http_host;
          proxy_set_header X-NginX-Proxy true;
          proxy_pass http://192.168.1.100:8000$request_uri;
          proxy_redirect off;
      }
  }

```

  这里的配置文件很简单，只是设置了根目录，并配置了转发。

### 使用 docker-compose.yml 关联容器

`docker-compose.yml`放置在根目录下：

``` bash
version: '3'
services:
  # 容器名
  mysql_db: 
      container_name: mysql_db
      # 镜像
      image: docker.io/mysql:5.7
      # 获取root权限
      privileged: true
      restart: always
      # 环境变量（自行设置密码）
      environment:
      - MYSQL_DATABASE=FasterRunner
      - MYSQL_ROOT_PASSWORD=password
      # 目录共享,格式 宿主机目录:容器目录
      volumes:
      - /var/lib/mysql:/var/lib/mysql
      - ./mysql/config/my.cnf:/etc/my.cnf
      - ./mysql/init:/docker-entrypoint-initdb.d/
      # 端口映射,格式 宿主机端口:容器端口
      ports:
      - 3306:3306
      # 容器开机启动命令
      command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci  --socket=/var/lib/mysql/mysql.sock
  
  fasterunner:
      build: ./FasterRunner
      container_name: fasterunner
      image: fasterunner:latest
      restart: always
      # 依赖
      depends_on:
      - mysql_db
      privileged: true
      # 共享目录到容器,每次启动容器会copy宿主机代码
      volumes:
      - ./FasterRunner:/opt/workspace/FasterRunner/
      network_mode: host
      ports:
      - 8000:8000
      command: /bin/sh -c 'python3 manage.py runserver 0.0.0.0:8000'
   
  fasterweb:
      build: ./FasterWeb
      container_name: fasterweb
      image: fasterweb:latest
      restart: always
      # 依赖
      privileged: true
      # 共享目录到容器,每次启动容器会copy宿主机代码
      volumes:
      - ./FasterWeb/dist:/static/
      - ./FasterWeb/conf.d/:/etc/nginx/conf.d/
      ports:
      - 8082:8082
      # command: /bin/sh -c '\cp -rf /share/fasterweb/default.conf /etc/nginx/conf.d/ && \cp -rf /share/fasterweb/dist/  /usr/share/nginx/html/ && nginx -g "daemon off;"'
      
      
  rabbitmq:
      # 镜像
      image: docker.io/rabbitmq:3.6.15-management-alpine
      container_name: rabbitmq
      # 环境变量（自行设置密码）
      environment:
      - RABBITMQ_DEFAULT_USER=user
      - RABBITMQ_DEFAULT_PASS=password
      restart: always
      # 端口映射,格式 宿主机端口:容器端口
      ports:
      - "15672:15672"
      - "5672:5672"
      logging:
          driver: "json-file"
          options:
             max-size: "200k"
             max-file: "10"

```

这个模板文件中，定义了4个`service`：mysql_db、fasterunner、fasterweb、rabbitmq，分别对应着数据库、后端应用、负载均衡器和消息队列服务。而这4个`service`就组成了一个`project`。

##### mysql_db service

数据持久化服务，主要用于保存系统生成的数据，并通过`volumes`关键字将数据文件挂载到宿主机的路径，以保证容器更新或者重新创建时数据不丢失；

将`my.cnf`文件挂载到本地，可实现相关mysql的配置修改；

将`init.sql`文件挂载到本地，可实现容器创建时，执行初始化的mysql语句；

##### fasterweb service

这一部分，实际上就是指定以`FasterWeb/`作为上下文构建镜像，类似于进入到`FasterWeb/`目录下，然后`docker build .`。`ports`指定了暴露的端口映射，类似于直接运行`docker run`时的`-p`参数。

##### fasterunner service

后端容器的构建也很简单，同样指定上下文构建，并暴露端口。`depends_on`指明了依赖关系，解决容器的依赖、启动先后的问题。这里指明了`fasterunner`必须在`mysql_db`之后启动。原因很简单，后端应用在启动时就需要连接数据库，那么数据库自然就要先于它启动了。

##### rabbitmq service

消息服务器，和后端服务中的celery想配合，完成一些异步任务。

## 项目编译和运行

``` bash
第一步： 先在linux下安装好docker和docker-compose；
第二步： 把本项目拉取到本地之后，进入到docker-compose.yml文件所在目录；
第三步：在docker-compose.yml文件中配置mysql和rabbitmq容器的用户名及密码；
第四步：将FasterRunner项目文件夹下文件拷贝到本项目对应文件夹下；
第五步：在FasterRunner/FasterRunner/settings.py文件中设置好连接mysql和rabbitmq配置信息，与docker-compose.yml中配置的一致；
第六步：在FasterWeb/src/restful/api.js中配置baseUrl为在FasterRunner后台地址如："http://192.168.1.100:8000"，或者为""
第七步：编译打包前端项目，克隆FasterWeb项目至本地，依次执行如下指令（生成FasterWeb/dist/文件夹及文件；此步骤需安装node.js，所以也可以在开发电脑上编译打包，再将FasterWeb/dist/文件夹及文件拷贝到本项目对应路径FasterWeb/dist/）：

cd ./FasterWeb
npm install
npm run build

第八步：配置FasterRunnerWeb/FasterWeb/conf.d/yelda.conf中的IP对应部署服务器IP
#接下来通过命令行进行操作
#切换到项目目录下
cd ./FasterRunnerWeb
docker-compose up
第九步：创建数据库表和初始化数据：
docker exec -it fasterunner bash  #进入容器内部
# make migrations for fastuser、fastrunner
python3 manage.py makemigrations fastrunner fastuser
# migrate for database
python3 manage.py migrate fastrunner
python3 manage.py migrate fastuser
python3 manage.py migrate djcelery

#若要关闭容器，使用如下命令
docker-compose down

第十步：open url: http://宿主机ip:8082/fastrunner/register
```



## FAQ

1. 关于`depends_on`，该参数只是保证了**启动顺序**，例如，在上文中，可以保证先启动`mysql_db`再启动`fasterunner`。然而，并不能保证`fasterunner`会在`mysql_db`**完全启动**之后再启动。如果遭遇到启动顺序上的问题，可以尝试手动先启动数据库。
2. 由于`Docker`会使用缓存机制，因此你有时候会发现容器内部的内容可能没有更新。例如，当更改了`docker-compose.yml`等文件时，直接使用`docker-compose up`是不正确的，需要重新构建，可以在后面加上`--build`参数。
3. 同样由于缓存机制，容器内部拷贝内容会没有更新，可以指定`--no-cache`命令，以及在运行时使用`--force-recreate`参数。如果这还没有解决问题，直接删除掉容器重新构建就好了。