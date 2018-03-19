---
layout: post
title: Docker基础-docker compose配置共享
categories: [docker]
description: docker compose配置共享
keywords: docker
---

compose 支持两种方式来共享通用配置：

1.使用多个compose文件来扩展整个compose文件
2.使用extends字段扩展单个服务

<!--more-->
# 多个compose文件组合
使用多个compose文件，可以为不同环境或不同工作流自定义compose应用程序

## 理解多compose文件组合

默认，compose会读取两个文件，一个`docker-compose.yml`和一个可选的`docker-compose.override.yml`文件。通常，`docker-compose.yml`文件包含你的基本配置，而`docker-compose.override.yml`，顾名思义，就是包含的现有服务配置的覆盖内容，或完全新的配置。

如果一个服务在这两个文件中都有定义，那么compose将使用[添加和覆盖配置](https://docs.docker.com/compose/extends/#adding-and-overriding-configuration)中所描述的规则来合并服务。

要使用多个override文件或不同名称的override文件，可以使用`-f`选项来指定文件列表。compose根据在命令行指定的顺序来合并它们。

当使用多个配置文件时，必须确保文件中所有的路径都是相对于`base compose`文件的(-f 指定的第一个compose文件)。这样要求是因为override文件不需要一个有效的compose文件。override文件可以只包含配置中的一小片段。跟踪一个服务的片段是相对于那个路径的，这是很困难的事，所以一定要保持路径容易理解，所以路径必须定义为相对于base文件的路径。

## 示例
在本段介绍两个常见的多compose文件用例：为不同的环境更改compose app和正对compose app运行管理任务。

### 不同环境
使用多文件的一个常用案例是更改用于类生产环境(可能是生产环境，临时环境或配置项)的开发compose app。要支持这样的更改，可以把一个compose配置文件分割为多个不同的文件：
从定义服务的规范配置的基本文件开始。
**docker-compose.yml**

```bash
web:
  image: example/my_web_app:latest
  links:
    - db
    - cache

db:
  image: postgres:latest

cache:
  image: redis:latest
```

**docker-compose.override.yml**
在下面这个示例中，开发配置内容向主机公开了一些端口，将代码安装为卷，并构建web镜像
```bash
web:
  build: .
  volumes:
    - '.:/code'
  ports:
    - 8883:80
  environment:
    DEBUG: 'true'

db:
  #command: '-d'
  ports:
    - 5432:5432

cache:
  ports:
    - 6379:6379
```
当执行`docker-cmopose up`命令时，它会自动读取这个override文件。

现在我们创建另一个用于生产环境的override文件
**docker-compose.prod.yml**
```bash
web:
  ports:
    - 80:80
  environment:
    PRODUCTION: 'true'

cache:
  environment:
    TTL: '500'
```
要使用这个生产compose文件部署，运行如下命令：
```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
这将会使用`docker-compose.yml`和`docker-compose.prod.yml`来部署这三个服务

## 管理任务

另一个常见的用力是正对在`compose app`中的一个或多个服务运行adhoc或管理任务。这个示例演示备份运行中的数据库。

**docker-compose.yml**
```bash
web:
  image: example/my_web_app:latest
  links:
    - db

db:
  image: postgres:latest
```

在`docker-cmopose.admin.yml`中添加一个新的服务来运行数据库导出或备份
**docker-compose.admin.yml**
```bash
dbadmin:
  build: database_admin/
  links:
    - db
```

要启动一个正常的环境运行`docker-compose up -d`。要备份数据库，运行：
```bash
docker-compose -f docker-compose.yml -f docker-compose.admin.yml \
    run dbadmin db-backup
```

# 扩展服务

docker compose的extends关键字能在不同的文件甚至是完全不同的项目之间共享通用的配置。如果有几个服务使用同样的通用配置项，则扩展服务非常有用。使用extends你可以在一个地方定义一个通用的服务选项集，然后在任何地方引用。
> **注意：** 不能使用extends来共享`links`,`volumes_from`和`depends_on`配置。这是为了避免隐式依赖-始终在本地定义links和volumes_from.这确保了当读取当前文件时服务之间的依赖是清晰可见的。在本地定义这些还确保对引用文件的更改不会导致破坏。

## 理解extends配置

当在`docker-compose.yml`定义服务时，我们可以像这样声明扩展另一个服务：
```bash
web:
  extends:
    file: common-services.yml
    service: webapp
```
这告诉compose重用定义在common-services.yml文件的webapp服务配置。假设common-services.yml的内容如下：
```bash
webapp:
  build: .
  ports:
    - "8000:8000"
  volumes:
    - "/data"
```
在这种情况下，docker-compose.yml中定义的web服务将与webapp配置一样。

然后，我们可以进一步在本地定义(或重新定义)docker-compose.yml：
```bash
web:
  extends:
    file: common-services.yml
    service: webapp
  environment:
    - DEBUG=1
  cpu_shares: 5

important_web:
  extends: web
  cpu_shares: 10
```

也可以定义其他服务并在web服务链接它们：
```bash
web:
  extends:
    file: common-services.yml
    service: webapp
  environment:
    - DEBUG=1
  cpu_shares: 5
  links:
    - db
db:
  image: postgres
```

## extends示例

当多个服务有一个通用配置时，扩展单个服务非常有用。下面示例是有两个服务的compose app：一个web应用程序和一个queue worker。两个服务使用相同的代码和共享许多配置项

在`common.yml`我们定义通用的配置：
```bash
app:
  build: .
  environment:
    CONFIG_FILE_PATH: /code/config
    API_KEY: xxxyyy
  cpu_shares: 5
```

在`docker-compose.yml`文件我们定义使用通用配置的具体服务：
```bash
webapp:
  extends:
    file: common.yml
    service: app
  command: /code/run_web_app
  ports:
    - 8080:8080
  links:
    - queue
    - db

queue_worker:
  extends:
    file: common.yml
    service: app
  command: /code/run_worker
  links:
    - queue
```

## 添加和覆盖配置

compose从原始服务复制配置到本地服务。如果配置项同时定义在原始服务和本地服务中，则本地服务的值会替换或扩展原始服务的值。
对于单值选项如image，command或mem_limit，新值替换旧值
```bash
# original service
command: python app.py

# local service
command: python otherapp.py

# result
command: python otherapp.py
```

> **注意：** 当使用的是version 1的compose文件格式时，build和image两个选项，在本地服务使用一个选项会导致compose忽略另一个定义在原始服务的选项
> 例如，如果原始服务定义了image:webapp且本地服务定义了build:. 那么结果是服务有build:. 选项没有image选项
> 因为在version 1文件中build和image不能一起使用

对于多值选项，ports，expose，extenal_link，dns，dns_search 和tmpfs，compos合并两组值
```bash
# original service
expose:
  - "3000"

# local service
expose:
  - "4000"
  - "5000"

# result
expose:
  - "3000"
  - "4000"
  - "5000"
```

对于environment，labels，volumes和devices，compose以本地定义的值优先原则来合并它们：
```bash
# original service
environment:
  - FOO=original
  - BAR=original

# local service
environment:
  - BAR=local
  - BAZ=local

# result
environment:
  - FOO=original
  - BAR=local
  - BAZ=local
```
