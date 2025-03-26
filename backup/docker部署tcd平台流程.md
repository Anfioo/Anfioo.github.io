# 1.部署准备工作

# 2.部署流程

## 初始化

### 提权

```sh
sudo -i
```

### 创建文件夹

```sh
cd /
mkdir tcdapp
cd tcdapp
```

### 创建网络

```sh
docker network create -d bridge tcdnet
```

## minio存储服务

### 创建映射文件夹

```cmd
cd /tcdapp
mkdir minioData
mkdir minioConfig
ls
```

### 部署

```sh
docker run  --name tcdminio \
  --network tcdnet\
  -p 9000:9000 \
  -p 9090:9090 \
  -d --restart=always \
  -e "MINIO_ROOT_USER=3485977506" \
  -e "MINIO_ROOT_PASSWORD=3485977506" \
  -v /tcdapp/minioData:/data \
  -v /tcdapp/minioConfig:/root/.minio \
  minio/minio server \
  /data --console-address ":9090" --address ":9000"
```

## Redis服务

### 创建文件夹及文件

```sh
cd /tcdapp
mkdir redis
cd redis
mkdir conf
mkdir data
cd conf
touch redis.conf
```

### 编辑配置

```sh
vim redis.conf
```

```js
requirepass 123456
protected-mode no
bind 0.0.0.0
```

### 部署

```sh
docker run \
--restart=always \
--network tcdnet \
--log-opt max-size=100m \
--log-opt max-file=2 \
-p 6379:6379 \
--name tcdredis \
-v /tcdapp/redis/conf/redis.conf:/etc/redis/redis.conf  \
-v /tcdapp/redis/data:/data \
-d redis redis-server /etc/redis/redis.conf \
--appendonly yes \
--requirepass 123456 
```

## MySQL服务

### 创建文件夹及文件

```sh
mkdir -p  /tcdapp/mysql/{conf,data,log}
cd /tcdapp/mysql/conf
sudo touch my.cnf
vim my.cnf
```

```mysql
[client]
#设置客户端默认字符集utf8mb4
default-character-set=utf8mb4
[mysql]
#设置服务器默认字符集为utf8mb4
default-character-set=utf8mb4
[mysqld]
#配置服务器的服务号，具备日后需要集群做准备
server-id = 1
#开启MySQL数据库的二进制日志，用于记录用户对数据库的操作SQL语句，具备日后需要集群做准备
log-bin=mysql-bin
#设置清理超过30天的日志，以免日志堆积造过多成服务器内存爆满。2592000秒等于30天的秒数
binlog_expire_logs_seconds = 2592000
#解决MySQL8.0版本GROUP BY问题
sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'
#允许最大的连接数
max_connections=1000
# 禁用符号链接以防止各种安全风险
symbolic-links=0
# 设置东八区时区
default-time_zone = '+8:00'
```

### 部署

```sh
docker run \
  --restart=always \
  -p 3306:3306 \
  --network tcdnet \
  --name tcdmysql \
  --privileged=true \
  -v /tcdapp/mysql/log:/var/log/mysql \
  -v /tcdapp/mysql/data:/var/lib/mysql \
  -v /tcdapp/mysql/conf/my.cnf:/etc/mysql/my.cnf \
  -v /tcdapp/mysql/mysql-files:/var/lib/mysql-files \
  -e MYSQL_ROOT_PASSWORD=a12bCd3_W45pUq6 \
  -d mysql:latest
```

### 登录权限更新

#### 使用exec连接数据库

```cmd
docker exec -it tcdmysql /bin/bash
mysql -u root -pa12bCd3_W45pUq6
```

####  root用户所有ip皆可登录

```mysql
CREATE USER 'root'@'%' IDENTIFIED BY 'a12bCd3_W45pUq6';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
FLUSH PRIVILEGES;
```

## WebSSH服务

### 部署

```sh
docker run -d -p 5032:5032 --log-driver json-file --log-opt max-file=1 --log-opt max-size=100m --restart always --name webssh -e TZ=Asia/Shanghai jrohy/webssh
```

## minioService文件上传服务

### 代码修改

####  application.properties修改返还的ip地址

```properties
spring.application.name=minio
server.port=4321
minio.access-key=SJmWtUkA18rR6f2RHZkN
minio.secret-key=qBzkKzAJhKZphlbjo8q9qf9nfi2x08PjUtF09aiN
minio.bucket=tcd-service-platform-bucket,tcd-service-platform-product-img-bucket
minio.url=http://tcdminio:9000
minio.return-url=http://101.37.105.91:9000

spring.servlet.multipart.max-file-size=50MB
spring.servlet.multipart.max-request-size=50MB
```

### Dockerfile

```dockerfile
FROM openjdk:22

LABEL author=anfioo

COPY minio-0.0.1-SNAPSHOT.jar /minio-0.0.1-SNAPSHOT.jar

EXPOSE 4321

ENTRYPOINT ["java","-jar","/minio-0.0.1-SNAPSHOT.jar"]
```

### 构建docker容器

```sh
docker build -t tcdminioservice .
```

### 部署

```sh
docker run \
--restart=always \
--network tcdnet \
-p 4321:4321 \
--name tcdminioservice \
-d tcdminioservice

```

## *minio界面嵌入修改（可选）

### 创建并编辑文件

```sh
cd /tcdapp
mkdir nginxConfig
touch nginx.conf
```

```sh
vim nginx.conf
```

### 配置

```nginx

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
#    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    server {
        listen       80;
        server_name  localhost;

    location / {
        proxy_pass http://tcdminio:9090;  # 将常规 HTTP 请求代理到 WebSocket 服务
        proxy_hide_header X-Frame-Options;
    }
    # 处理 WebSocket 请求
    location /ws/ {
        proxy_pass http://tcdminio:9090;  # 将 WebSocket 请求代理到 WebSocket 服务
       proxy_set_header  X-Real-IP  $remote_addr;
       proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
	   proxy_cache_bypass $http_upgrade;
       proxy_set_header Host "127.0.0.1:9000";
       proxy_set_header Origin "http://127.0.0.1:9000";
     
    }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

```

### 部署

```sh
docker run -d \
  --network tcdnet\
  --name tcdnginx \
  -p 7777:80 \
  -v /tcdapp/nginxConfig:/etc/nginx \
  nginx:stable-alpine3.20-perl
```

### 说明

原有的minio web ui 无法嵌入ifarme 使用这个去掉这个特性

## tcd后端服务 框架部分

### 修改配置文件

#### application.yml

配置文件上传路径，redis账户设置

```properties
# 项目相关配置
ruoyi:
  # 名称
  name: RuoYi
  # 版本
  version: 3.8.8
  # 版权年份
  copyrightYear: 2024
  # 文件路径 示例（ Windows配置D:/ruoyi/uploadPath，Linux配置 /home/ruoyi/uploadPath）
  profile: /tcd/uploadPath
  # 获取ip地址开关
  addressEnabled: false
  # 验证码类型 math 数字计算 char 字符验证
  captchaType: math

# 开发环境配置
server:
  # 服务器的HTTP端口，默认为8080
  port: 1234
  servlet:
    # 应用的访问路径
    context-path: /
  tomcat:
    # tomcat的URI编码
    uri-encoding: UTF-8
    # 连接数满后的排队数，默认为100
    accept-count: 1000
    threads:
      # tomcat最大线程数，默认为200
      max: 800
      # Tomcat启动初始化的线程数，默认值10
      min-spare: 100

# 日志配置
logging:
  level:
    com.ruoyi: debug
    org.springframework: warn

# 用户配置
user:
  password:
    # 密码最大错误次数
    maxRetryCount: 5
    # 密码锁定时间（默认10分钟）
    lockTime: 10

# Spring配置
spring:
  # 资源信息
  messages:
    # 国际化资源文件路径
    basename: i18n/messages
  profiles:
    active: druid
  # 文件上传
  servlet:
    multipart:
      # 单个文件大小
      max-file-size: 10MB
      # 设置总上传的文件大小
      max-request-size: 20MB
  # 服务模块
  devtools:
    restart:
      # 热部署开关
      enabled: true
  # redis 配置
  redis:
    # 地址
    host: tcdredis
    # 端口，默认为6379
    port: 6379
    # 数据库索引
    database: 0
    # 密码
    password: 123456
    # 连接超时时间
    timeout: 10s
    lettuce:
      pool:
        # 连接池中的最小空闲连接
        min-idle: 0
        # 连接池中的最大空闲连接
        max-idle: 8
        # 连接池的最大数据库连接数
        max-active: 8
        # #连接池最大阻塞等待时间（使用负值表示没有限制）
        max-wait: -1ms

# token配置
token:
  # 令牌自定义标识
  header: Authorization
  # 令牌密钥
  secret: abcdefghijklmnopqrstuvwxyz
  # 令牌有效期（默认30分钟）
  expireTime: 30

# MyBatis配置
mybatis:
  # 搜索指定包别名
  typeAliasesPackage: com.ruoyi.**.domain
  # 配置mapper的扫描，找到所有的mapper.xml映射文件
  mapperLocations: classpath*:mapper/**/*Mapper.xml
  # 加载全局的配置文件
  configLocation: classpath:mybatis/mybatis-config.xml

# PageHelper分页插件
pagehelper:
  helperDialect: mysql
  supportMethodsArguments: true
  params: count=countSql

# Swagger配置
swagger:
  # 是否开启swagger
  enabled: true
  # 请求前缀
  pathMapping: /dev-api


# 防止XSS攻击
xss:
  # 过滤开关
  enabled: true
  # 排除链接（多个用逗号分隔）
  excludes: /system/notice
  # 匹配链接
  urlPatterns: /system/*,/monitor/*,/tool/*
#
#qiniu:
#  accessKey: i0GpVSe_nfw2b6Igrra1YIi2AR88LmC4SZfmsPdV
#  secretKey: ieJwYUYvKVioCiGeLQG1ENWuV7hjp8mC6sbZyt2D
#  bucket: tcd-service-text1
#  url: http://spv0e9fwq.hn-bkt.clouddn.com

```

#### application-druid.yml

使用自己的数据库架构名称 这里使用的是tcdmysql

```properties
# 数据源配置
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.cj.jdbc.Driver
        druid:
            # 主库数据源
            master:
                url: jdbc:mysql://tcdmysql:3306/tcd?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
                username: root
                password: a12bCd3_W45pUq6
            # 从库数据源
            slave:
                # 从数据源开关/默认关闭
                enabled: false
                url: 
                username: 
                password: 
            # 初始连接数
            initialSize: 5
            # 最小连接池数量
            minIdle: 10
            # 最大连接池数量
            maxActive: 20
            # 配置获取连接等待超时的时间
            maxWait: 60000
            # 配置连接超时时间
            connectTimeout: 30000
            # 配置网络超时时间
            socketTimeout: 60000
            # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
            timeBetweenEvictionRunsMillis: 60000
            # 配置一个连接在池中最小生存的时间，单位是毫秒
            minEvictableIdleTimeMillis: 300000
            # 配置一个连接在池中最大生存的时间，单位是毫秒
            maxEvictableIdleTimeMillis: 900000
            # 配置检测连接是否有效
            validationQuery: SELECT 1 FROM DUAL
            testWhileIdle: true
            testOnBorrow: false
            testOnReturn: false
            webStatFilter: 
                enabled: true
            statViewServlet:
                enabled: true
                # 设置白名单，不填则允许所有访问
                allow:
                url-pattern: /druid/*
                # 控制台管理用户名和密码
                login-username: ruoyi
                login-password: 123456
            filter:
                stat:
                    enabled: true
                    # 慢SQL记录
                    log-slow-sql: true
                    slow-sql-millis: 1000
                    merge-sql: true
                wall:
                    config:
                        multi-statement-allow: true
```

### Dockerfile

```dockerfile
FROM openjdk:17

LABEL author=anfioo

COPY ruoyi-admin.jar /ruoyi-admin.jar
RUN mkdir -p /tcd/uploadPath

EXPOSE 1234

ENTRYPOINT ["java","-jar","/ruoyi-admin.jar"]
```

### 构建docker容器

```sh
docker build -t tcd .
```

### 创建文件夹

```sh
mkdir /tcdapp/tcd
```

### 部署

```sh
docker run \
--restart=always \
-p 1234:1234 \
--network tcdnet \
--name tcd \
-v /tcdapp/tcd:/tcd/uploadPath \
-d tcd:latest
```

## tcd后端服务 模型训练部分

### 修改配置文件

#### application.properties

```properties
#spring.application.name=model-service
#server.port=5321
#spring.servlet.multipart.max-file-size=50MB
#spring.servlet.multipart.max-request-size=50MB
#params.backup-parent-folder=C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py
#params.bak-parent-folder=C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py\\backup
#params.restore-parent-folder=C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py
#params.SEPARATOR==-=-=QwErTyUiOpAsDfGhJkLzXcVbNm=-=-=
#params.python-interpreter=C:\\Apps\\Development\\anaconda3\\envs\\Matplotlib\\tcdgpu\\python.exe
#params.tensorboard-path=C:\\Apps\\Development\\anaconda3\\envs\\Matplotlib\\tcdgpu\\Scripts\\tensorboard.exe
#params.tensorboard-port=7765
#params.working-folder=C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py\\model_1
#params.custom-main-train-path="C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py\\model_1\\main_c.py"
#params.default-main-train-path="C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py\\model_1\\main_d.py"
#file.root-path=C:\\Users\\34859\\Desktop\\TCDServicePlatform-2025-2-12\\TCDServicePlatform\\model-service-py\\model_1\\logs



spring.application.name=model-service
server.port=5321
spring.servlet.multipart.max-file-size=50MB
spring.servlet.multipart.max-request-size=50MB
params.backup-parent-folder=/app/model-service-py
params.bak-parent-folder=/app/model-service-py/backup
params.restore-parent-folder=/app/model-service-py
params.SEPARATOR==-=-=QwErTyUiOpAsDfGhJkLzXcVbNm=-=-=
params.python-interpreter=/opt/conda/envs/tcd/bin/python
params.tensorboard-path=/opt/conda/envs/tcd/bin/tensorboard
params.tensorboard-port=7765
params.working-folder=/app/model-service-py/model_1
params.custom-main-train-path=/app/model-service-py/model_1/main_c.py
params.default-main-train-path=/app/model-service-py/model_1/main_d.py
file.root-path=/app/model-service-py/model_1/logs

```

### 上传部署所需的文件

#### environment.yml

```yaml
name: tcd
channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
dependencies:
  - _libgcc_mutex=0.1
  - _openmp_mutex=5.1
  - _tflow_select=2.3.0
  - abseil-cpp=20211102.0
  - absl-py=2.1.0
  - aiohappyeyeballs=2.4.0
  - aiohttp=3.10.5
  - aiosignal=1.2.0
  - astunparse=1.6.3
  - async-timeout=4.0.3
  - attrs=24.2.0
  - blas=1.0
  - blinker=1.6.2
  - bottleneck=1.3.7
  - brotli=1.0.9
  - brotli-bin=1.0.9
  - brotli-python=1.0.9
  - bzip2=1.0.8
  - c-ares=1.19.1
  - ca-certificates=2025.2.25
  - cachetools=5.3.3
  - certifi=2024.8.30
  - cffi=1.17.1
  - charset-normalizer=3.3.2
  - click=8.1.7
  - contourpy=1.0.5
  - cryptography=41.0.3
  - cycler=0.11.0
  - cyrus-sasl=2.1.28
  - dbus=1.13.18
  - expat=2.6.4
  - flatbuffers=24.3.25
  - fontconfig=2.14.1
  - fonttools=4.51.0
  - freetype=2.12.1
  - frozenlist=1.4.0
  - gast=0.4.0
  - giflib=5.2.2
  - glib=2.78.4
  - glib-tools=2.78.4
  - google-auth=2.29.0
  - google-auth-oauthlib=0.5.2
  - google-pasta=0.2.0
  - grpc-cpp=1.48.2
  - grpcio=1.48.2
  - gst-plugins-base=1.14.1
  - gstreamer=1.14.1
  - h5py=3.11.0
  - hdf5=1.12.1
  - icu=58.2
  - idna=3.7
  - importlib-metadata=7.0.1
  - importlib_resources=6.4.0
  - intel-openmp=2023.1.0
  - joblib=1.4.2
  - jpeg=9e
  - keras=2.12.0
  - keras-preprocessing=1.1.2
  - kiwisolver=1.4.4
  - krb5=1.20.1
  - lcms2=2.16
  - ld_impl_linux-64=2.40
  - lerc=4.0.0
  - libbrotlicommon=1.0.9
  - libbrotlidec=1.0.9
  - libbrotlienc=1.0.9
  - libclang=14.0.6
  - libclang13=14.0.6
  - libcups=2.4.2
  - libcurl=8.2.1
  - libdeflate=1.22
  - libedit=3.1.20230828
  - libev=4.33
  - libffi=3.4.4
  - libgcc-ng=11.2.0
  - libgfortran-ng=11.2.0
  - libgfortran5=11.2.0
  - libglib=2.78.4
  - libgomp=11.2.0
  - libiconv=1.16
  - libllvm14=14.0.6
  - libnghttp2=1.52.0
  - libpng=1.6.39
  - libpq=12.15
  - libprotobuf=3.20.3
  - libssh2=1.10.0
  - libstdcxx-ng=11.2.0
  - libtiff=4.5.1
  - libuuid=1.41.5
  - libwebp-base=1.3.2
  - libxcb=1.15
  - libxkbcommon=1.0.1
  - libxml2=2.10.4
  - lz4-c=1.9.4
  - markdown=3.4.1
  - markupsafe=2.1.3
  - matplotlib=3.7.2
  - matplotlib-base=3.7.2
  - mkl=2023.1.0
  - mkl-service=2.4.0
  - mkl_fft=1.3.8
  - mkl_random=1.2.4
  - multidict=6.0.4
  - mysql=5.7.24
  - ncurses=6.4
  - numexpr=2.8.4
  - numpy=1.23.5
  - numpy-base=1.23.5
  - oauthlib=3.2.2
  - openjpeg=2.5.2
  - openssl=1.1.1w
  - opt_einsum=3.3.0
  - packaging=24.1
  - pandas=2.0.3
  - pcre2=10.42
  - pillow=10.4.0
  - pip=24.2
  - platformdirs=3.10.0
  - ply=3.11
  - pooch=1.7.0
  - protobuf=3.20.3
  - pyasn1=0.4.8
  - pyasn1-modules=0.2.8
  - pycparser=2.21
  - pyjwt=2.8.0
  - pyopenssl=23.2.0
  - pyparsing=3.0.9
  - pyqt=5.15.10
  - pyqt5-sip=12.13.0
  - pysocks=1.7.1
  - python=3.8.18
  - python-dateutil=2.9.0post0
  - python-flatbuffers=24.3.25
  - python-tzdata=2023.3
  - pytz=2024.1
  - qt-main=5.15.2
  - re2=2022.04.01
  - readline=8.2
  - requests=2.32.3
  - requests-oauthlib=2.0.0
  - rsa=4.7.2
  - scikit-learn=1.3.0
  - scipy=1.10.1
  - setuptools=75.1.0
  - sip=6.7.12
  - six=1.16.0
  - snappy=1.2.1
  - sqlite=3.45.3
  - tbb=2021.8.0
  - tensorboard=2.12.1
  - tensorboard-data-server=0.7.0
  - tensorboard-plugin-wit=1.8.1
  - tensorflow=2.12.0
  - tensorflow-base=2.12.0
  - tensorflow-estimator=2.12.0
  - termcolor=2.1.0
  - threadpoolctl=3.5.0
  - tk=8.6.14
  - tomli=2.0.1
  - tornado=6.4.1
  - typing_extensions=4.11.0
  - unicodedata2=15.1.0
  - urllib3=2.2.3
  - werkzeug=3.0.3
  - wheel=0.44.0
  - wrapt=1.14.1
  - xz=5.6.4
  - yarl=1.11.0
  - zipp=3.20.2
  - zlib=1.2.13
  - zstd=1.5.6


```



### Dockerfile

```dockerfile
FROM continuumio/miniconda3:latest

LABEL author=anfioo

# 设置清华APT源
RUN printf "deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free\ndeb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free\ndeb https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free\n" > /etc/apt/sources.list

# 修复基础环境
RUN mkdir -p /usr/share/man/man1 && \
    apt-get update -q && \
    apt-get install -y --no-install-recommends \
        man-db \
        manpages-dev \
        software-properties-common \
        ca-certificates-java && \
    dpkg --configure -a

# 安装JDK
RUN apt-get update -q && \
    apt-get install -y --no-install-recommends \
        openjdk-17-jre-headless=17.0.14+7-1~deb11u1 \
        openjdk-17-jre=17.0.14+7-1~deb11u1 \
        openjdk-17-jdk-headless=17.0.14+7-1~deb11u1 \
        openjdk-17-jdk=17.0.14+7-1~deb11u1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 创建应用目录并复制文件
RUN mkdir -p /app/model-service-py
WORKDIR /app
COPY model-service-0.0.1-SNAPSHOT.jar ./
ADD model-service-py.tar /app/

# 复制并安装Conda环境
COPY environment.yml .
RUN conda config --set show_channel_urls yes && \
    conda env create -f environment.yml && \
    conda clean -afy

# 设置环境变量
ENV PATH="/opt/conda/envs/tcd/bin:$PATH" \
    JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

EXPOSE 5321 7765

ENTRYPOINT ["java", "-jar", "/app/model-service-0.0.1-SNAPSHOT.jar"]

```

### 构建docker容器

```sh
docker build -t tcdpy .
```

### 创建文件夹

```sh
mkdir tcdpy
```

### 部署

```sh
docker run \
--restart=always \
--network tcdnet \
-p 5321:5321 \
-p 7765:7765 \
-v /tcdapp/tcdpy:/app/model-service-py \
--name tcdpy \
-d tcdpy:latest
```

## tcd前端服务

### 编辑配置文件

#### nginx.conf

```nginx

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    server {
        listen       66;
        server_name  localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html index.html;
        try_files $uri $uri/ /index.html;
    }
	location /prod-api/ {
        proxy_pass http://tcd:1234/;
	}

			  # 配置 /upload 路径代理
    location /upload {
        proxy_pass http://tcdminioservice:4321/upload;  # 代理到后端

    }

    # 配置 /uploadProductImg 路径代理
    location /uploadProductImg {
        proxy_pass http://tcdminioservice:4321/uploadProductImg;  # 代理到后端
    }


    }
    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

```

### Dockerfile

```dockerfile
FROM nginx:stable-alpine3.20-perl

MAINTAINER anfioo

COPY dist/ usr/share/nginx/html/
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 66
ENTRYPOINT ["nginx","-g","daemon off;"]
```

### 构建docker容器

```sh
docker build -t tcdui .
```

### 部署

```
docker run \
--restart=always \
--network tcdnet \
-p 66:66 \
--name tcdui \
-d tcdui:latest
```