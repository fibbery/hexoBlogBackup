---
title: "一次insert into on duplicate key upadte引发的问题"
date: 2019-01-09 17:05:51
categories: mysql
tags:
    - mysql
---

## 问题

  最近有业务场景使用insert into ... on duplicate key update进行插入更新操作，然后通过返回的affected rows来确定是进行了插入操作还是更新操作。印象中当不出现键冲突的时候affected rows应该为1（作用等同于insert into）, 而当键冲突的时候affected rows应该为2。但是实际情况是有键位冲突更新的时候affected rows返回的是1。  

## 解决思路

  首先我们梳理下表结构以及执行语句，业务表的设计大致如下：

```sql
create table Record(
    id int auto_increment primary key,
    user_id int not null comment '用户id',
    object_id int not null comment '对方id',
    create_time datetime not null default now() comment '记录创建时间',
    unique key idx_user_object(`user_id`,`object_id`)
)engine = innodb comment '记录表'
```

实际执行的业务操作语句如下:

```sql
insert into Record(user_id,object_id,create_time) values(1,1,now()) on duplicate key update set create_time = now()
```

查了[MySQL Reference](https://dev.mysql.com/doc/refman/5.7/en/insert-on-duplicate.html)可以发现是有对affected rows 进行说明的，内容如下:
>With ON DUPLICATE KEY UPDATE, the affected-rows value per row is 1 if the row is inserted as a new row, 2 if an existing row is updated, and 0 if an existing row is set to its current values. If you specify the CLIENT_FOUND_ROWS flag to the mysql_real_connect() C API function when connecting to mysqld, the affected-rows value is 1 (not 0) if an existing row is set to its current values.    

也就是说当无主键冲突的时候affected row = 1；主键冲突的时候回去自定义更新记录 affected rows = 2；主键冲突但是自定义更新的值和原有值相同的时候affected rows = 0。如果mysql的参数CLIENT_FOUND_ROWS是设置成mysql_real_connect()，那么这种情况下也会返回1。那么问题很明显了，因为我们设置create_time是秒级别的，那么肯定存在一种情况，在你使用api连接mysql的时候加上参数CLIENT_FOUND_ROWS，在同一秒内A事务插入了一条记录返回affected rows = 1,B事务去执行更新的时候由于更新值和当前值相同导致返回affected rows = 1。

查询CLIENT_FOUND_ROWS发现如下解释：
>useAffectedRows  
>Don't set the CLIENT_FOUND_ROWS flag when connecting to the server (not JDBC-compliant, will break most applications that rely on "found" rows vs. "affected rows" for DML statements), but does cause "correct" update counts from "INSERT ... ON DUPLICATE KEY UPDATE" statements to be returned by the server.  
Default: false  
Since version: 5.1.7

而默认jdbc是会默认传递CLIENT_FOUND_ROWS的，也就是返回的是寻找到的行，当然可以在连接参数上加上userAffectedRows=true来让其返回收影响的行

## 问题验证

直接在库中执行

```mysql
insert into `Record`(`user_id`,`object_id`,`create_time`) values(1,1,'2019-1-9 19:03:33') on duplicate key update create_time = '2019-1-9 19:03:33'
```

返回```Query OK, 1 row affected```，接下来继续执行相同的语句，返回```Query OK, 0 rows affected```，意味着情况和文档描述的一样。我们通过jdbc直接操作会发现第二次执行语句时是会返回affected rows = 1的，在连接字符串上加上useAffectedRows是会发现返回的是affected rows = 0 

> 注意  
如果使用了mycat中间件的话，在连接字符串加useAffectedRow也没设么用，因为mycat默认配置的就是CLIENT_FOUND_ROWS
![代码结构](/assets/blogImg/mycat-img01.jpg)
