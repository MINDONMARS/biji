-- 创建表
create table tb_areas(
​    aid int primary key,
​    atitle varchar(20),
​    pid int
);


-- 自关联  自己关联自己 a inner join a, 无限向下分级的业务可以使用自关联
-- 通过 source 指令导入一个sql文件
​	-- 省级联动 url:http://demo.lanrenzhijia.com/2014/city0605/
​	-- 查询所有省份
​	select * from areas where pid is NULL;


	-- 查询出广东省有哪些市
	广东省 广州市
	广东省 深圳市
	...
	--  需要有想象力, 将areas 想象成两张表 , 一张是省表, 一张是市表\
	select p.atitle, c.atitle from areas as p inner join areas as c on p.aid = c.pid where p.atitle = "广东省";
	-- 查询出广州市有哪些区 p: parent, s:son
	select p.atitle, s.atitle from areas as p inner join areas as s on p.aid = s.pid where p.atitle = "广州市";

-- 子查询 select .... select 
​	-- 需要在小括号之中, 优先执行
​	-- 是一个能够独立执行的sql语句

	-- 查询最大年龄对应的学生的信息
	select * from students where age = (select max(age) from students);
	select max(age) from students;
	
	按照查询的结构来分类:
	- 标量子查询  子查询语句查询的结果是一个基本的值, 一行一列
	-- 查询班级最高身高的学生信息
	select * from students where height = (select max(height) from students);
	
	-- 查询出高于平均身高的信息
	select * from students where height > (select avg(height) from students);


	- 列级子查询  查询的结果是一列多行


​	
	select * from students where age in (select age from students where age in (18,20,30));
	select age from students where age in (18,20,30);
	
	-- 查询哪些班级有学生  查找的是班级的名字
	select name from classes where id in (select cls_id from students);
	
	-- 通过连接查询来实现
	select distinct classes.name from classes inner join students on classes.id = students.cls_id;
	-- 查找哪些班级没有学生



	- 行级子查询 ,一行多列
	-- 行元素,将多个字段合成一个行元素,在行级子查询中会使用到行元素
	-- 查找班级年龄最大,身高最高的学生
	select * from students where (age, height) = (select max(age), max(height) from students);


	- 表级子查询 子查询语句查询出来的结果是多行多列
	-- select * from students;
	select tmp.* from (select * from students) as tmp;
	# 错误 select * from (select * from students);  -- 需要起别名


-- 数据库备份与恢复
​	-- 备份 将某一台主机的数据完成一个副本的备份 mysqldump 是liunx下的指令
​	mysqldump -uroot -p > ~/Desktop/python_res.sql

	-- 恢复  通过xxx.sql 文件将数据恢复到指定的数据库
	1. 在某台mysql服务下创建数据库 python_restore
	create database python_restore charset=utf8;
	2. 将数据恢复到指定的数据库
	mysql -uroot -p python_restore < ~/Desktop/python_res.sql




-- 数据库设计

	-- 三范式 


	-- E-R模型
		- 1 对 1
		- 1 对 多
		- 多 对 多



-- 类似京东商城
-- 求所有商品的平均价格,并且保留两位小数
select round(avg(price),2) from goods;

-- 查询所有价格大于平均价格的商品，并且按价格降序排序
select * from goods where price > (select round(avg(price),2) from goods) order by price desc;
-- -- 标量子查询


-- 查询类型cate_name为 '超级本' 的商品名称、价格
select name,price from goods where cate_name = "超级本";

-- 显示商品的种类
select distinct cate_name from goods;

select cate_name from goods group by cate_name;


-- 显示每种类型的商品的平均价格
select cate_name,avg(price) from goods group by cate_name;


-- 查询每种类型的商品中 最贵、最便宜、平均价、数量
select cate_name,max(price),min(price),avg(price),count(*) from goods group by cate_name;

-- 查询每种类型中最贵的商品信息
select goods.* from goods inner join (select cate_name,max(price) as max_price,min(price),avg(price),count(*) from goods group by cate_name) as tmp on goods.cate_name = tmp.cate_name and goods.price = tmp.max_price; 

-- 通过行级子查询也可以实现该需求
错误: select * from goods where (cate_name, price) = (select cate_name,max(price) from goods group by cate_name);
select * from goods where (cate_name, price) in (select cate_name,max(price) from goods group by cate_name);

\G结尾 让查询的结果的每一列单独成行显示

-- 创建商品分类表
create table if not exists goods_cates(
​    id int unsigned primary key auto_increment,
​    name varchar(40) not null
);


-- insert into xx表  values (), (),...; 需要手动写比较麻烦
-- insert ... select  # 将查询的结果插入到某一个表中
-- 子查询语句
-- 在sql中所有的语句都称为查询语句
-- 结构化查询语言
-- 查询到的数据的字段需要和插入数据表指定的字段一一对应
insert into goods_cates (name) select cate_name from goods group by cate_name;


-- 连接更新
-- 根据goods_cates 表 更新goods 表, 更新 cate_name
-- 需要将多张表连接起来
update xx表  set col = xx; 
update goods inner join goods_cates on goods.cate_name = goods_cates.name set goods.cate_name = goods_cates.id;

-- 修改表结构
alter table goods change cate_name cate_id int unsigned not null;



-- brand_name 品牌名称
-- 创建表 并且插入数据 一步到位  create ... select
-- 如果查找的数据的字段和表中的字段名不一样 此时会在表中创建一个新的名字一样的字段,将查询到的数据一一对应的自动创建的字段上面
-- 给brand_name 起别名, 或者将goods_brands表中的name 修改为brand_name
create table goods_brands (
​	id int unsigned primary key auto_increment,
​	name varchar(40)
) select brand_name as name from goods group by brand_name;


---  根据goods_brand表 来更新 goods表
update goods inner join goods_brands on goods.brand_name = goods_brands.name set goods.brand_name = goods_brands.id;

-- 修改表结构



-- 连接多张表完成查询
-- 显示商品的完整信息
-- 连接多张表直接inner join 不需要and 或者 ,
select * from goods as g inner join goods_cates as c on g.cate_id = c.id inner join goods_brands as b on g.brand_id = b.id;

-- left join 左外连接查询
select * from goods as g left join goods_cates as c on g.cate_id = c.id left join goods_brands as b on g.brand_id = b.id;
select * from goods as g right join goods_cates as c on g.cate_id = c.id right join goods_brands as b on g.brand_id = b.id;

select * from goods as g left join goods_cates as c on g.cate_id = c.id right join goods_brands as b on g.brand_id = b.id;


insert into goods (name,cate_id,brand_id,price) values('惠普LaserJet Pro P1606dn 黑白激光打印机', 12, 4,'1849');

-- 外键
-- 对于已经存在的表 设置外键约束--> 更新  alter  ... add
-- 给goods表中的 brand_id 添加外键约束, goods表已经存在, 修改约束应该是用alter sql 语句
-- references: 关联或者引用
alter table goods add foreign key(brand_id) references goods_brands(id);

alter table goods add foreign key(cate_id) references goods_cates(id);

能否在创建表的时候就设置外键约束?
-- 在添加 cate_id外键约束的时候 必须是goods_cates表已经被创建才能够添加外键约束


-- 删除外键
-- alter table goods drop foreign key 外键约束的名字;
-- 如何获取外键约束的名字
alter table goods drop foreign key goods_ibfk_1;
alter table goods drop key brand_id;

-- 外键名称从表的创建语句中来查看
show create table goods;



学习在python调用 mysql (数据的增删改查)


事务只对表中的数据有效
对于数据库的操作和表结构的更改无效