---
title: 使用docker搭建mysql双主配置
date: 2018-05-10 10:23:53
categories: mysql
tags:
    - mysql
    - docker
---
## 前言

互联网架构需要保证数据库高可用，常见的一种方式，使用双主同步+keepalived+虚ip的方式保证数据库的可用性。依照前文[使用docker搭建mysql简单主从集群](https://fibbery.me/2018/05/09/%E4%BD%BF%E7%94%A8docker%E6%90%AD%E5%BB%BAmysql%E7%AE%80%E5%8D%95%E4%B8%BB%E4%BB%8E%E9%9B%86%E7%BE%A4/) 想尝试一下使用docker搭建会有什么问题


<!-- more-->
## 实践步骤

1. 搭建好docker网络
    因为需要两个容器互联，所以需要使用docker连创建互联网络，参见[用network create解决Docker容器互相连接的问题](http://www.up4dev.com/2016/10/09/docker-network-create/)
    
    ```shell
        docker network create my-net  # 先创建一个网络
        docker network ls # 查看一下是否创建成功
    ```

2. 新建masterA库

    ```shell
         docker run -d -p 3306:3306 -v /data/mysqlcnf/masterA:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name masterA --net=my-net --net-alias=masterA mysql   # 利用自建网络搭建主库A
    ```

    数据库配置文件如下:

    ```shell
        server-id                = 1
        auto_increment_offset    = 1
        auto_increment_increment = 1

        log_bin                = mysql-bin
        binlog-format          = ROW
        log-slave-updates      = true
        relay-log              = slave-relay-bin
        relay-log-index        = slave-relay-bin.index
        lower_case_table_names = 1
    ```

    因为是互为主从，所以auto_increment_offset(起始位移点)和auto_increment_increment(步长)必须间隔设置，同时必须开启log-slave-updates设置，前文有提到过。

    重启masterA库

    ```shell
        docker restart masterA
    ```
3. 新建masterB 主库

    ```shell
        docker run -d -p 3307:3306 -v /data/mysqlcnf/masterB:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name masterB --net=my-net --net-alias=masterB mysql  # 利用自建网络搭建主库B
    ```

    数据库配置文件如下:

    ```shell
        [mysqld]
        server-id                = 2
        auto_increment_offset    = 2
        auto_increment_increment = 2

        log_bin                = mysql-bin
        binlog-format          = ROW
        log-slave-updates      = true
        relay-log              = slave-relay-bin
        relay-log-index        = slave-relay-bin.index
        lower_case_table_names = 1
    ```

    重启B库

    ```shell
        docker restart masterB
    ```

4. 配置数据库帐号以及主从配置
    分别登录masterA和masterB添加从库同步帐号

    ```sql
        grant replication slave on *.* to 'sync'@'%' identified by 'sync'  # 新建同步帐号
        flush privileges # 刷新权限缓存
    ```

    分别执行查看masterA和masterB的binlog情况

    ```sql
        show master status
    ```

    分别进行主从同步
    ```sql
        change master to master_host='${host}',master_port='${port}',master_user= '${master_user}' ,master_password='${master_password}',master_log_file='${master_log_file}',master_log_pos='${master_log_pos}'
        start slave
        show slave status\G
    ```　
    以上sql都可以观看前文得知，不再赘述。到此，双主mysql集群搭建完成。

