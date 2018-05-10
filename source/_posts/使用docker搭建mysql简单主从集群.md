---
title: 使用docker搭建mysql简单主从集群
date: 2018-05-09 20:32:37
categories: mysql
tags:
	- mysql
	- docker
---
## 前言

最近对mysql集群搭建以及后期如何在线上进行拆分和迁移产生了兴趣，所以初步想在自己本机通过docker搭建一个简单的主从集群来满足一下好奇心

## 开发环境

OS X EI Capitan 10.11.6
docker版本17.09.0-ce-mac35 (19611)
mysql镜像版本5.7(这里我直接下载的标签为latest的最新版本)

## 具体实现步骤

### 建立主库容器

1. 建立主数据库

    ```shell
        docker run -d -p 3307:3306 -v /data/mysqlcnf/master:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name master mysql
    ```

2. 在本机目录/data/mysqlcnf/master中新建mysql数据库配置文件my.cnf，内容如下:

    ```shell
        [mysqld]
        server-id                = 1
        log_bin                  = mysql-bin
        binlog-format            = ROW
        lower_case_table_names   = 1
    ```
    需要注意下mysql镜像自带是有配置文件的，我们映射过来的文件夹并不能完全覆盖，所以不要让配置文件冲突

3. 重启容器

    ```shell
        docker restart master
    ```

4. 建立授权帐号

    ```sql
        grant replication slave on *.* to 'sync'@'%' identified by 'sync'
        flush privileges
    ```

5. 查询下主库当前binlog情况

    ```sql
        show master status
    ```

    记录下file和position，等到设置从库时需要

### 建立从库容器

1. 建立从数据库(使用link实现从库连接主库)

    ```shell
        docker run -d -p 3307:3306 -v /data/mysqlcnf/slave:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name slave --link master:master mysql
    ```

2. 从库配置文件

    ```shell
        [mysqld]
        server-id              = 2

        log-bin                = mysql-bin
        binlog-format          = ROW
        lower_case_table_names = 1
        log-slave-updates      = 1
        relay-log              = slave-relay-bin
        relay-log-index        = slave-relay-bin.index
        read-only              = 1
    ```

    log-slave-updates参数的意义在于，如果该库是从库，从主库binlog读取的更新是不会继续写入该库的binlog，设置该参数后则可以实现读取主库binlog之后写入自己库的binlog，用来做其他从库的主库

3. 重启容器

    ```shell
        docker restart slave
    ```

4. 登录从库服务器设置主库

    ```sql
        CHANGE MASTER TO MASTER_HOST='master',MASTER_USER='sync', MASTER_PASSWORD='sync',MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS=154;
        flush privileges
        start slave
    ```

    这里设置的值master_log_file以及master_log_pos根据当初查询主库得来

5. 查看从库状态

    ```sql
        show slave status\G
    ```

至此简单的mysql的主从集群搭建完成