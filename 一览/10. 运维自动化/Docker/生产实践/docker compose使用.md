#### 1. 命令

基础使用格式

```
docker compose [-f=<arg>...] [options] [COMMAND] [ARGS...]
```



##### 4.1 options

- -f, --file FILE  指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定。
- -p, --project-name NAME 指定项目名称，默认将使用所在目录名称作为项目名。
- --verbose 输出更多调试信息。
- -v, --version 打印版本并退出。



##### 5.2 COMMAND

**1）build**

构建（重新构建）项目中的服务容器。

```
docker compose build [options] [SERVICE...]
```


options包括：

- --force-rm 删除构建过程中的临时容器。
- --no-cache 构建镜像过程中不使用 cache（这将加长构建过程）。
- --pull 始终尝试通过 pull 来获取更新版本的镜像。

 

**2）config**

验证 Compose 文件格式是否正确，若正确则显示配置，若格式错误显示错误原因。

 

**3）down**

此命令将会停止 up 命令所启动的容器，并移除网络。

 

**4）exec**

进入指定的容器。

 

**5）help**

获得一个命令的帮助。

 

**6）images**

列出 Compose 文件中包含的镜像。

 

**7）kill**

通过发送 SIGKILL 信号来强制停止服务容器。

支持通过 -s 参数来指定发送的信号，例如通过如下指令发送 SIGINT 信号。

```
docker compose kill -s SIGINT
```

 

**8）logs**

查看服务容器的输出。

 

**9）pause**

暂停一个服务容器

```
docker compose pause [SERVICE...]
```

 

**10）port**

打印某个容器端口所映射的公共端口。

```
docker compose port [options] SERVICE PRIVATE_PORT
```



options选项：

- --protocol=proto 指定端口协议，tcp（默认值）或者 udp。
- --index=index 如果同一服务存在多个容器，指定命令对象容器的序号（默认为 1）。

 

**11）ps**

列出项目中目前的所有容器。

```
docker compose ps [options] [SERVICE...]
```



options选项：

- -q 只打印容器的 ID 信息。

 

**12）pull**

拉取服务依赖的镜像。

```
docker compose pull [options] [SERVICE...]
```

options选项：

- --ignore-pull-failures 忽略拉取镜像过程中的错误。

 

**13）push**

推送服务依赖的镜像到 Docker 镜像仓库。

 

**14）restart**

重启项目中的服务

```
docker compose restart [options] [SERVICE...]
```



options选项：

- -t, --timeout TIMEOUT 指定重启前停止容器的超时（默认为 10 秒）

 

**15）rm**

删除所有（停止状态的）服务容器。推荐先执行 docker compose stop 命令来停止容器。

```
docker compose rm [options] [SERVICE...]
```



options选项：

- -f, --force 强制直接删除，包括非停止状态的容器。一般尽量不要使用该选项。
- -v 删除容器所挂载的数据卷。

 

**16）run**

在指定服务上执行一个命令

```
docker compose run [options] [-p PORT...] [-e KEY=VAL...] SERVICE [COMMAND] [ARGS...]
```



例如启动一个 ubuntu 服务容器，并执行 ping docker.com 命令。

```
docker compose run ubuntu ping docker.com
```



默认情况下，如果存在关联，则所有关联的服务将会自动被启动，除非这些服务已经在运行中。

如果不希望自动启动关联的容器，可以使用 --no-deps 选项，例如

```
docker compose run --no-deps web python manage.py shell
```



options选项： 

- -d 后台运行容器。
- --name NAME 为容器指定一个名字。
- --entrypoint CMD 覆盖默认的容器启动指令。
- -e KEY=VAL 设置环境变量值，可多次使用选项来设置多个环境变量。
- -u, --user="" 指定运行容器的用户名或者 uid。
- --no-deps 不自动启动关联的服务容器。
- --rm 运行命令后自动删除容器，d 模式下将忽略。
- -p, --publish=[] 映射容器端口到本地主机。
- --service-ports 配置服务端口并映射到本地主机。
- -T 不分配伪 tty，意味着依赖 tty 的指令将无法运行。

 

**17）start**

启动已经存在的服务容器。

```
docker compose start [SERVICE...]
```

 

**18）stop**

停止已经处于运行状态的容器，但不删除它。通过 docker compose start 可以再次启动这些容器。

```
docker compose stop [options] [SERVICE...]
```



options选项：

- -t, --timeout TIMEOUT 停止容器时候的超时（默认为 10 秒）。

 

**19）top**

查看各个服务容器内运行的进程。

 

**20）unpause**

恢复处于暂停状态中的服务。

 

**21）up**

该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

```
docker compose up [options] [SERVICE...]
```



默认情况，如果服务容器已经存在，docker compose up 将会尝试停止容器，然后重新创建（保持使用 volumes-from 挂载的卷），以保证新启动的服务匹配 docker-compose.yml 文件的最新内容。如果用户不希望容器被停止并重新创建，可以使用 docker compose up --no-recreate。这样将只会启动处于停止状态的容器，而忽略已经运行的服务。如果用户只想重新部署某个服务，可以使用 docker compose up --no-deps -d <SERVICE_NAME> 来重新创建服务并后台停止旧服务，启动新服务，并不会影响到其所依赖的服务。



options选项：

- -d 在后台运行服务容器。
- --no-color 不使用颜色来区分不同的服务的控制台输出。
- --no-deps 不启动服务所链接的容器。
- --force-recreate 强制重新创建容器，不能与 --no-recreate 同时使用。
- --no-recreate 如果容器已经存在了，则不重新创建，不能与 --force-recreate 同时使用。
- --no-build 不自动构建缺失的服务镜像。
- -t, --timeout TIMEOUT 停止容器时候的超时（默认为 10 秒）。

 

**22）version**

打印版本信息



#### 2. 模板文件

**1）build**

指定 Dockerfile 所在文件夹的路径（可以是绝对路径，或者相对 docker-compose.yml 文件的路径）。 Compose 将会利用它自动构建这个镜像，然后使用这个镜像。

```
version: '3'
services:

  webapp:
    build: ./dir
```

build指令下的选项有

- context 指令指定 Dockerfile 所在文件夹的路径
- dockerfile 指令指定 Dockerfile 文件名
- arg 指令指定构建镜像时的变量
- cache_from 指定构建镜像的缓存



```
version: '3'
services:

  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
      cache_from:
        - alpine:latest
        - corp/web_app:3.14
```

 

**2）cap_add, cap_drop**

指定容器的内核能力（capacity）分配。

```
# 让容器拥有所有能力
cap_add:
  - ALL
# 去掉 NET_ADMIN 能力  
cap_drop:
  - NET_ADMIN
```

 

**3）command**

覆盖容器启动后默认执行的命令。

command: echo "hello world"

 

**4）configs**

仅用于 Swarm mode

 

**5）cgroup_parent**

指定父 cgroup 组，意味着将继承该组的资源限制。

cgroup_parent: cgroups_1

 

**6）container_name**

指定容器名称。默认将会使用 项目名称_服务名称_序号

container_name: docker-web-container

注意: 指定容器名称后，该服务将无法进行扩展（scale），因为 Docker 不允许多个容器具有相同的名称。

 

**7）deploy**

仅用于 Swarm mode

 

**8）devices**

指定设备映射关系。

devices:

 \- "/dev/ttyUSB1:/dev/ttyUSB0"

 

**9）depends_on**

解决容器的依赖、启动先后的问题。以下例子中会先启动 redis db 再启动 web

```
version: '3'

services:
  web:
    build: .
    depends_on:
      - db
      - redis

  redis:
    image: redis

  db:
    image: postgres
```

web 服务不会等待 redis db 「完全启动」之后才启动。

 

**10）dns**

自定义 DNS 服务器。可以是一个值，也可以是一个列表。

```
dns: 8.8.8.8

dns:
  - 8.8.8.8
  - 114.114.114.114
```

 

**11）dns_search**

配置 DNS 搜索域。可以是一个值，也可以是一个列表。

```
dns_search: example.com

dns_search:
  - domain1.example.com
  - domain2.example.com
```

 

**12）tmpfs**

挂载一个 tmpfs 文件系统到容器。

```
tmpfs: /run
tmpfs:
  - /run
  - /tmp
```

 

**13）env_file**

从文件中获取环境变量，可以为单独的文件路径或列表。

```
env_file: .env

env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

如果有变量名称与 environment 指令冲突，则按照惯例，以后者为准。

 

**14）environment**

设置环境变量。你可以使用数组或字典两种格式。

```
environment:
  RACK_ENV: development
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SESSION_SECRET
```



注意：只给定名称的变量会自动获取运行 Compose 主机上对应变量的值，可以用来防止泄露不必要的数据。

注意：如果变量名称或者值中用到 true|false，yes|no 等表达 布尔 (opens new window)含义的词汇，最好放到引号里，避免 YAML 自动解析某些内容为对应的布尔语义。

 

**15）expose**

暴露端口，但不映射到宿主机，只被连接的服务访问。

```
expose:
 - "3000"
 - "8000"
```

 

**16）external_links**

链接到 docker-compose.yml 外部的容器，甚至并非 Compose 管理的外部容器。

```
external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
```

 

**17）extra_hosts**

类似 Docker 中的 --add-host 参数，指定额外的 host 名称映射信息。

```
extra_hosts:
 - "googledns:8.8.8.8"
 - "dockerhub:52.1.157.61"
```

 

**18）healthcheck**

通过命令检查容器是否健康运行。

```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
```

 

**19）image**

指定为镜像名称或镜像 ID。如果镜像在本地不存在，Compose 将会尝试拉取这个镜像。

```
image: ubuntu
image: orchardup/postgresql
image: a4bc65fd
```

 

**20）labels**

为容器添加 Docker 元数据（metadata）信息。例如可以为容器添加辅助说明信息。

```
labels:
  com.startupteam.description: "webapp for a startup team"
  com.startupteam.department: "devops department"
  com.startupteam.release: "rc3 for v1.0"
```

 

**21）logging**

配置日志选项。

```
logging:
  # 支持syslog、json-file、none
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
    max-size: "200k"
    max-file: "10"
```

 

**22）network_mode**

设置网络模式。使用和 docker run 的 --network 参数一样的值。

```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

 

**23）networks**

配置容器连接的网络。

```
version: "3"
services:

  some-service:
    networks:
     - some-network
     - other-network

networks:
  some-network:
  other-network:
```

 

**24）pid**

跟主机系统共享进程命名空间。打开该选项的容器之间，以及容器和宿主机系统之间可以通过进程 ID 来相互访问和操作。

```
pid: "host"
```

 

**25）ports**

暴露端口信息。

 

使用宿主端口：容器端口 (HOST:CONTAINER) 格式，或者仅仅指定容器的端口（宿主将会随机选择端口）都可以

```
ports:
 - "3000"
 - "8000:8000"
 - "49100:22"
 - "127.0.0.1:8001:8001"
```

 

**26）secrets**

存储敏感数据，例如 mysql 服务密码。

```
version: "3.1"
services:

mysql:
  image: mysql
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
  secrets:
    - db_root_password
    - my_other_secret

secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```

 

**27）security_opt**

指定容器模板标签（label）机制的默认属性（用户、角色、类型、级别等）。例如配置标签的用户名和角色名。

```
security_opt:
    - label:user:USER
    - label:role:ROLE
```

 

**28）stop_signal**

设置另一个信号来停止容器。在默认情况下使用的是 SIGTERM 停止容器。

```
stop_signal: SIGUSR1
```

 

**29）sysctls**

配置容器内核参数。

```
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0

sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

 

**30）ulimits**

指定容器的 ulimits 限制值。

 

例如，指定最大进程数为 65535，指定文件句柄数为 20000（软限制，应用可以随时修改，不能超过硬限制） 和 40000（系统硬限制，只能 root 用户提高）。

```
  ulimits:
    nproc: 65535
    nofile:
      soft: 20000
      hard: 40000
```

 

**31）volumes**

数据卷所挂载路径设置。

可以设置为宿主机路径(HOST:CONTAINER)或者数据卷名称(VOLUME:CONTAINER)，并且可以设置访问模式 （HOST:CONTAINER:ro）。

```
volumes:
 - /var/lib/mysql
 - cache/:/tmp/cache
 - ~/configs:/etc/configs/:ro

# 如果路径为数据卷名称，必须在文件中配置数据卷。
volumes:
  mysql_data: 
```

 

**32）其他指令**

此外，还有包括 domainname, entrypoint, hostname, ipc, mac_address, privileged, read_only, shm_size, restart, stdin_open, tty, user, working_dir 等指令，基本跟 docker run 中对应参数的功能一致。

 

指定服务容器启动后执行的入口文件。

```
entrypoint: /code/entrypoint.sh
```



指定容器中运行应用的用户名。

```
user: nginx
```



指定容器中工作目录。

```
working_dir: /code
```



指定容器中搜索域名、主机名、mac 地址等。

```
domainname: your_website.com

hostname: test

mac_address: 08-00-27-00-0C-0A
```



允许容器中运行一些特权命令。

```
privileged: true
```



指定容器退出后的重启策略为始终重启。该命令对保持服务始终运行十分有效，在生产环境中推荐配置为 always 或者 unless-stopped。

```
restart: always
```



以只读模式挂载容器的 root 文件系统，意味着不能对容器内容进行修改。

```
read_only: true
```



打开标准输入，可以接受外部输入。

```
stdin_open: true
```



模拟一个伪终端。

```
tty: true
```



#### 3. 案例

搭建Django环境

1）搭建容器依赖环境 --- Dockerfile

```
FROM python:3
ENV PYTHONUNBUFFERED 1
RUN mkdir /code
WORKDIR /code
COPY requirements.txt /code/
RUN pip install -r requirements.txt
COPY . /code/
```

 

在 requirements.txt 文件里面写明需要安装的具体依赖包名。

```
Django>=2.0,<3.0
psycopg2>=2.7,<3.0
```

 

2）搭建项目运行环境 --- docker compose

```
version: "3"
services:

  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: 'postgres'

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/code
    ports:
      - "8000:8000"
```

 

3）启动项目

```
docker compose run web django-admin startproject django_example .
```

 

4）其他工作

run启动完后 会在当前目录生成一个默认的django_example项目，我们需要修改默认连接信息



修改 django_example/settings.py 

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
        'PASSWORD': 'postgres',
    }
}
```



如果你的系统是 Linux,记得更改文件权限。

```
 sudo chown -R $USER:$USER .
```



最后重新构建 + 启动 = up

```
docker compose up
```

 