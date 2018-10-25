# mysql

http://www.runoob.com/  菜鸟教程


启动服务: 
​	sudo service mysql start
停止服务: 
​	sudo service mysql stop
重启服务:
​	sudo service mysql restart


搜索mysql服务进程信息
ps ajx|grep mysql


数据库用户名: root
密码:        mysql

-- 数据库的操作

    -- 链接数据库 liunx 命令
    mysql -uroot -pmysql 
    
    -- 退出数据库
    eixt,   quit,   ctrl + d
    
    -- 查看创建数据库
    show databases;
    
    -- 查看当前使用的数据库
    select database();
    
    -- 使用数据库
    use python10;
    
    -- sql语句最后需要有分号;结尾
    -- 显示数据库版本
    select version();
    
    -- 显示时间
    select now()
    
    -- 创建数据库 create 
    错误没有指定字符集: create database demo;
    
    -- 指定字符集 不是 utf-8
    create database demo charset=utf8;
    
    -- 查看数据库的创建语句
    show create database demo;
    
    -- 删除数据库
    drop database demo;    


-- 数据表的操作

    -- 查看当前数据库中所有表
    show tables;
    
    -- 创建表
    -- auto_increment表示自动增长
    -- 创建一个学生的数据表(id、name、age、height、gender、cls_id)
    -- create table 数据表名字 (字段 类型 约束[, 字段 类型 约束]);  []在sql语法结构中表示的是可有可无
    -- 多个约束 不分先后顺序
    -- enum 表示枚举
    -- 最后一个字段不要添
    -- 创建students表
create table students(
​    id int unsigned primary key not null auto_increment,
​    name varchar(15) not null,
​    age tinyint unsigned default 18,
​    height decimal(5,2) default 0,
​    gender enum("男",'女',"中性","保密") default "保密",
​    cls_id int unsigned not null
);

    -- 查看表的创建语句
    show create table students;
    
    -- 查看表结构
    desc students;
    
    -- 修改表结构  alter  add/modify/change
    -- 修改表-添加字段
    -- alter table 表名 add 列名 类型/约束;
    -- 生日信息 
    alter table students add birthday datetime default "2011-11-11 11:11:11";
    
    -- 修改表-修改字段：不重命名版
    -- alter table 表名 modify 列名 类型及约束;
    alter table students modify birthday date default "2011-11-11";
    
    -- 修改表-修改字段：重命名版
    -- alter table 表名 change 原列名 新列名 类型及约束;
    alter table students change birthday birth date default "2011-11-11";


    -- 修改表-删除字段
    alter table students drop birth;
    
    -- 删除表
    drop table students;


​    
-- 数据增删改查(curd)

+--------+-------------------------------------+------+-----+---------+----------------+
| Field  | Type                                | Null | Key | Default | Extra          |
+--------+-------------------------------------+------+-----+---------+----------------+
| id     | int(10) unsigned                    | NO   | PRI | NULL    | auto_increment |
| name   | varchar(15)                         | NO   |     | NULL    |                |
| age    | tinyint(3) unsigned                 | YES  |     | 18      |                |
| height | decimal(5,2)                        | YES  |     | 0.00    |                |
| gender | enum('男','女','中性','保密')         | YES  |     | 保密    |                |
| cls_id | int(10) unsigned                    | NO   |     | NULL    |                |
+--------+-------------------------------------+------+-----+---------+----------------+


    -- 增加 insert 
    	-- insert [into] 表名 values (值1,值2,...)
        -- 全列插入  值和表的字段的顺序一一对应
        -- 枚举中的原始值 都会有一个小标与之对应, 下标的起始值是1
        -- 全列插入在实际开发的时候很少使用, 当表结构发生修改(增加 删除字段), 全列插入的sql语句就产生错误
        insert into students values (0,'张飞',18,180.00,'男',1);
        insert into students values (NULL,'小乔',18,180.00,'女',1);
        insert into students values (Default,'大乔',16,180.00,'女',1);
        insert into students values (Default,'大乔',16,180.00,'女',1);
        insert into students values (Default,'刘备',16,180.00,1,1);
        insert into students values (Default,'大乔',16,180.00,2,1);
        错误: insert into students values (Default,'大乔',16,180.00,2);
    
        -- 指定列插入
        -- 值和列一一对应
        -- insert into 表名 (列1,...) values (值1,...)
        insert into students (name, gender, cls_id) values ("周瑜",1,2);


        -- 多行插入  批量插入
        -- insert into 表名 (列1,...) values (值1,...),(值1,...)...
        insert into students (name, gender, cls_id) values ("关羽",1,2),("赵云",1,2),("黄忠",1,2);
    
    -- 修改 update
    -- where 表示修改的范围
    -- update 表名 set 列1=值1,列2=值2... [where 条件]
    -- 如果没有where条件部分就是全表更新
    错误: update students set age = 20;
    update students set age = 18 where id = 2;
    -- sql中  通过一个等于号表示相等 



    dba 
    -- 删除 delete
        -- 物理删除
        -- DELETE FROM tbname [where 条件判断]
        整表删除: delete from students;
        delete from students where id = 5;
    
        诺基亚: 下架之后删除手机相关的数据 我的订单--> 订单详情 -->商品详情
    
        逻辑删除: 通过一个字段 标记某一条数据是否被删除
        1. 修改表结构
        alter table students add is_delete bit default 0;
        2. 更新数据
        update students set is_delete = 1 where id = 9;
    
        -- 查询有哪些学生没有被删除
        select * from students where is_delete = 0;
    
    -- 查询基本使用 select 
        -- 查询所有列
        -- select * from 表名;
        select * from students;
    
        -- 指定字段查询
        select name, gender, age from students;
        -- sql 中表示相等 使用 = 而不是 ==
    
        -- 指定条件查询
    
        -- 查询指定列
    
        -- 字段的顺序
    
        -- 可以使用as为列或表指定别名




​    