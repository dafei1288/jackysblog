
---
title: 十分钟搭建实验分布式数据库环境
date: 2022-04-08 14:30:15
tags: [database]
description: 构建一套分布式数据库的实验环境吧。我们使用`docker` 和 `postgres xl` 来完成
---

# 十分钟搭建实验分布式数据库环境


划水了好久，今天来跟大家分享一下如何用一台笔记本，构建一套分布式数据库的实验环境吧。我们使用`docker` 和 `postgres xl` 来完成。各位读者老爷们扣Q上车，Let's Go\!\!\!\!

## Postgres XL 简介

### 什么是Postgres-XL

XL的意思是：eXtensible Lattice，可以扩展的格子，即将PostgreSQL应用在多机器上的分布式数据库的形象化表达。Postgres-XL 是一个完全满足ACID的、开源的、可方便进行水平扩展的、多租户安全的、基于PostgreSQL的数据库解决方案。Postgres-XL 可非常灵活的应付各种负载，比如：

- OLAP（通过MPP并行化）
- OLTP
- OLAP \& OLTP
- 操作数据存储
- Key-value存储，包括JSON格式

不同的应用场景：

- 支持商业智能应用（数据仓库\&数据集市），因为PGXL支持MPP\(Massively Parallel Processing\)
- Web2.0，数据库扩容的解决方案
- 遗留系统的数据库扩容的解决方案
- 新应用，可以先使用PostgreSQL，之后随着数据库变大使用PGXL扩容

PGXL底层为PostgreSQL，这意味着它支持所有支持PostgresSQL类型的驱动，包括：JDBC, ODBC, OLE DB, Python, Ruby, perl DBI, Tcl, and Erlang.

### PostgreSQL与Postgres-XL

- 1994年，Postgre95发布，开源。
- 1996年，PostgreSQL继承了Postgre95，发布。
- 2010年，Postgres-XC发布。
- 2012年，前PGXC核心开发者创建StormDB公司，进行了一些改进，包括对MPP并行化的性能改进和多租户安全。
- 2013年，TransLattice收购了StormDB。
- 2014年，将项目开源，命名为Postgres-XL。

### Postgres-XC与Postgres-XL

PGXL的架构师和开发者 很多都是以前做PGXC的，PGXL的部分代码是从PGXC移植过来的。比起功能性，PGXL更强调稳定性, 正确性和性能.PGXL增加了一些重要的性能提升，比如MPP和replan avoidance on the data nodes，这些都是PGXC没有的。PGXC目前集中在OLTP的业务上面，PGXL则更加灵活，可以应用于很多不同种类的业务上，比如可以用在大数据处理领域，除此，在多租户的环境中，PGXL也更加安全。PGXL的社区非常开放。

### 架构

GXL有三个主要组件，分别是GTM，Coordinator\(CN\)和Datanode\(DN\)。![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagicZ7J30lHguUx0YZGNE27RrxnywjJgRSDbqUweXaCcaymYuOeX6dGl5dN3JZz1SIUBBBcdQLk3IRA/640?wx_fmt=png)GTM\(Gloable Transaction Manager\)负责提供事务的ACID属性；Datanode负责存储表的数据和本地执行由Coordinator派发的SQL任务；Coordinator负责处理每个来自Application的SQL任务，并且决定由哪个Datanode执行，然后将任务计划派发给相应的Datanode，根据需要收集结果返还给Application；  

## Postgres XL on Docker

我们采用一个GTM，2台CN，2台DN，结构如下图所示：![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagicZ7J30lHguUx0YZGNE27RrBUaFGy0t3JdiazV2C1gicURqB4veIW2L3CydmJfKr3xUN434pyxiam91g/640?wx_fmt=png)

### docker-compose.yml

配置文件如下所示，执行 `docker-compose up`，启动集群

```
version: "3"  
services:  
  db_gtm_1:  
    environment:  
      - PG_HOST=0.0.0.0  
      - PG_NODE=gtm_1  
      - PG_PORT=6666  
#      - PG_PASSWORD=dafei1288  
    build: .  
#    image: z_db_gtm_1  
    command: docker-cmd-gtm  
    entrypoint: docker-entrypoint-gtm  
    volumes:  
      - db_gtm_1:/var/lib/postgresql  
    networks:  
      - db_a  
    healthcheck:  
      test: ["CMD", "docker-healthcheck-gtm"]  
  db_coord_1:  
    ports:  
      - "25432:5432"  
    environment:  
      - PG_GTM_HOST=db_gtm_1  
      - PG_GTM_PORT=6666  
      - PG_HOST=0.0.0.0  
      - PG_NODE=coord_1  
      - PG_PORT=5432  
#      - PG_PASSWORD=dafei1288  
    build: .  
#    privileged: true  
#    image: z_db_coord_1  
    command: docker-cmd-coord  
    entrypoint: docker-entrypoint-coord  
    volumes:  
      - db_coord_1:/var/lib/postgresql  
    depends_on:  
      - db_gtm_1  
    networks:  
      - db_a  
      - db_b  
    healthcheck:  
      test: ["CMD", "docker-healthcheck-coord"]  
  db_coord_2:  
    ports:  
      - "25433:5432"  
    environment:  
      - PG_GTM_HOST=db_gtm_1  
      - PG_GTM_PORT=6666  
      - PG_HOST=0.0.0.0  
      - PG_NODE=coord_2  
      - PG_PORT=5432  
#      - PG_PASSWORD=dafei1288  
    build: .  
#    privileged: true  
#    image: z_db_coord_2  
    command: docker-cmd-coord  
    entrypoint: docker-entrypoint-coord  
    volumes:  
      - db_coord_2:/var/lib/postgresql  
    depends_on:  
      - db_gtm_1  
    networks:  
      - db_a  
      - db_b  
    healthcheck:  
      test: ["CMD", "docker-healthcheck-coord"]  
  db_data_1:  
    ports:  
      - "25434:5432"  
    environment:  
      - PG_GTM_HOST=db_gtm_1  
      - PG_GTM_PORT=6666  
      - PG_HOST=0.0.0.0  
      - PG_NODE=data_1  
      - PG_PORT=5432  
#      - PG_PASSWORD=dafei1288  
    build: .  
#    image: z_db_data_1  
    command: docker-cmd-data  
    entrypoint: docker-entrypoint-data  
    depends_on:  
      - db_gtm_1  
    volumes:  
      - db_data_1:/var/lib/postgresql  
    networks:  
      - db_a  
    healthcheck:  
      test: ["CMD", "docker-healthcheck-data"]  
  db_data_2:  
    ports:  
      - "25435:5432"  
    environment:  
      - PG_GTM_HOST=db_gtm_1  
      - PG_GTM_PORT=6666  
      - PG_HOST=0.0.0.0  
      - PG_NODE=data_2  
      - PG_PORT=5432  
#      - PG_PASSWORD=dafei1288  
    build: .  
#    image: z_db_data_2  
    command: docker-cmd-data  
    entrypoint: docker-entrypoint-data  
    depends_on:  
      - db_gtm_1  
    volumes:  
      - db_data_2:/var/lib/postgresql  
    networks:  
      - db_a  
    healthcheck:  
      test: ["CMD", "docker-healthcheck-data"]  
  pgpool:  
#    image: smirart/pgpool:latest  
    image: postdock/pgpool:latest  
    ports:  
      - "8686:8686"  
      - "8687:5432"  
      - "8688:9898"  
#    environment:  
#      - PG_PASSWORD=dafei1288  
    volumes:  
      - ./pgpool.conf:/var/pgpool_configs/pgpool.conf  
    restart: always  
    networks:  
      - db_b  
volumes:  
  db_gtm_1: {}  
  db_coord_1: {}  
  db_coord_2: {}  
  db_data_1: {}  
  db_data_2: {}  
networks:  
  db_a:  
    internal: true  
  db_b:  
    internal: true  
  
```

如果有需要，可以开启gppool，也可以注释掉，不影响使用

### pgpool.conf

```
listen_addresses = '*'  
port = 5432  
# pool_passwd = 'dafei1288'  
socket_dir = '/tmp'  
  
  
pcp_listen_addresses = '*'  
pcp_port = 9898  
pcp_socket_dir = '/tmp'  
listen_backlog_multiplier = 2  
serialize_accept = off  
  
replication_mode = on  
load_balance_mode = on  
  
backend_hostname0 = 'db_coord_1'  
backend_port0 = 5432  
backend_weight0 = 1  
backend_data_directory0 = '/data0'  
backend_flag0 = 'ALWAYS_MASTER'  
  
backend_hostname1 = 'db_coord_2'  
backend_port1 = 5432  
backend_weight1 = 1  
backend_data_directory1 = '/data1'  
backend_flag1 = 'ALLOW_TO_FAILOVER'  
  
  
  
health_check_period0 = 0  
health_check_timeout0 = 20  
health_check_user0 = '_healthcheck'  
health_check_password0 = ''  
health_check_database0 = ''  
health_check_max_retries0 = 0  
health_check_retry_delay0 = 1  
connect_timeout0 = 10000  
```

  

### 实验结果

  
![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagicZ7J30lHguUx0YZGNE27RrP8DJu2b2NiaspFYxRavCm0XckZNghDKHeMJXUHlPTuzXyic9qEaN6KUA/640?wx_fmt=png)  
![](https://mmbiz.qpic.cn/mmbiz_png/8uJ0ic6nAagicZ7J30lHguUx0YZGNE27Rr8E4TYh6TrM7UqicIWAlwogDX37dj65DD95f62sojmVJyxI6zU3gkJ9A/640?wx_fmt=png)  
本实验工程 fork自 `https://github.com/tiredpixel/z.2020-10-22.postgres-xl-docker`,由于原镜像已设置为只读，并且执行会出一些奇奇怪怪的错误，于是我就整理了一番，项目已托管到全球最大同仁网站gayhub，网址如下:  

`https://github.com/dafei1288/postgres-xl-docker`

## 参考列表

https://blog.csdn.net/yeruby/article/details/49004329
https://github.com/tiredpixel/z.2020-10-22.postgres-xl-docker
