# mysql查询

-- 查询
​	-- 查询所有字段
​	-- select * from 表名;
​	select * from students;
​	-- 查询指定字段
​	-- select 列1,列2,... from 表名;
​	select name, gender from students;


	-- sql语句完全的形式
	select students.* from students;
	select python_test_1.students.* from students;
	
	# 什么情况下数据表的名字是不建议省略的
	1. 当一条sql语句中出现了多个表的时候, 并且这些表中查询的字段有重名的就不能够省略表名
	select students.name,classes.name from students, classes;
	
	-- 使用 as 给字段起别名
	-- select 字段 as 名字.... from 表名;
	select name as "名字", gender as "性别" from students;	


	-- select 表名.字段 .... from 表名;
	select students.name, students.age, students.gender from students;


	-- 可以通过 as 给表起别名
	-- select 别名.字段 .... from 表名 as 别名;
	-- 在当前的sql 语句中 临时的给students 起了一个别名叫做s
	select s.name, s.age, s.gender from students as s;
	
	-- 消除重复行
	-- distinct 字段, 修饰所有需要查询的字段
	-- 查询班级学生的性别
	select gender from students;
	
	-- 查询班级有多少种性别
	select distinct gender from students;
	select distinct gender,age from students;
	select distinct gender,age,id from students;


-- 条件查询 where
select * from students where 1 > 0;
​	-- 比较运算符
​		-- >
​		-- 查询大于18岁的信息
​		select * from students where age > 18;

		-- <
		-- 查询小于18岁的信息
	
		-- >= 和 <=
	
		-- 表示相等 只有一个等于号
	
		-- != 或者 <>  实际开发中 最好使用 ! 表示不等于
		-- <> 不够通用


	-- 逻辑运算符
		-- and
		-- 18岁以上的女性
		select * from students where age > 18 and gender = 2;
	
		-- or
		-- 18以上或者身高超过180(包含)以上
		select * from students where age > 18 or height >= 180.00;
	
		-- not  非
		-- 年龄不是18岁的学生
		select * from students where age != 18;
		select * from students where not age = 18;
	
		-- 查询年龄大于18或者等于18岁的男生
		select * from students where age >= 18 and gender = 1;
	
		-- and的优先级比or的高 需要通过小括号提高优先级
		select * from students where (age > 18 or age = 18) and gender = 1;



	-- 模糊查询
		-- like 
		-- % 表示任意字符可有可无
		-- 查询姓名中 以 "小" 开始的名字
		select * from students where name like "小%";
	
		-- _ 表示任意一个字符
		-- 查询有2个字的名字
		select * from students where name like "__";
	
		-- 查询有3个字的名字
		select * from students where name like "___";
	
	-- 范围查询
		-- in 表示在一个非连续的范围内
		-- 查询 年龄为18、34岁的学生
		select * from students where age = 18 or age = 34;
		select * from students where age in (18, 34);
		
		-- not in 不在非连续的范围之内
		-- 年龄不是 18、34岁的学生的信息
		select * from students where age not in (18, 34);
	
		-- 18 ~ 34	
		select * from students where age >= 18 and age <= 34;
	
		-- between ... and ...表示在一个连续的范围内  两边都会包含
		-- 查询 年龄在18到34之间的的信息
		select * from students where age between 18 and 34;


​		
		-- not between ... and ...表示不在一个连续的范围内
		-- 查询 年龄不在在18到34之间的的信息
		select * from students where age not between 18 and 34;
		select * from students where not age between 18 and 34;  # 对条件取反
	
	-- 空判断 null  不能够使用比较运算符 应该使用is
		-- 查询身高为空的信息
		select * from students where height is NULL;
	
		-- 查询身高不为空的学生
		select * from students where height is not NULL;
		select * from students where not height is NULL;

-- 排序 从大到小--> 降序排序 从小到大-->升序排序
​	-- order by 字段 排序规则, 默认就是升序排序 asc 可以省略
​	-- asc从小到大排列，即升序
​	-- 查询年龄在18到34岁之间的男性，按照年龄从小到大排序
​	select * from students where age between 18 and 34 and gender = 1 order by age asc;
​	select * from students where age between 18 and 34 and gender = 1 order by age;

	-- 降序 desc
	-- desc从大到小排序，即降序
	-- 查询年龄在18到34岁之间的女性，身高从高到矮排序
	select * from students where age between 18 and 34 and gender = 2 order by height desc;
	
	-- order by 多个字段:  order by age asc, height desc
	-- 查询年龄在18到34岁之间的女性，身高从高到矮排序, 如果身高相同的情况下按照年龄从小到大排序
	select * from students where age between 18 and 34 and gender = 2 order by height desc, age asc;
	-- 查询年龄在18到34岁之间的女性，身高从高到矮排序, 如果身高相同的情况下按照年龄从小到大排序, 如果年龄也相同那么按照id从大到小排序
	select * from students where age between 18 and 34 and gender = 2 order by height desc, age asc,id desc;
	-- 按照年龄从小到大、身高从高到矮的排序
	select * from students order by age asc, height desc;



-- 聚合函数 做统计
​	-- 总数: count()
​	-- count(*) 以行单位来进行统计个数
​	-- count(*) 效率更高, 效率略差:count(height)--> 获取对应的行--> 获取该行对应字段是否为NULL
​	-- 查询班级有多少人
​	select count(*) from students;
​	select count(id) from students;


	-- 查询男性有多少人，女性有多少人
	select count(*) from students where gender = 1;
	select count(*) from students where gender = 2;


	-- 最大值: max() 
		-- 查询最大的年龄
		select max(age) from students;
	
		-- 查询女性的最高身高
		select max(height) from students where gender = 2;
	
		-- 查询最高身高的学生的名字  # 后面需要学习子查询语句
		select name from students where height = (select max(height) from students);


​	
	-- 最小值: min()
	
	-- 求和: sum()
		-- 计算所有人的年龄总和
		select sum(age) from students;


	-- 平均值: avg()
		-- 计算平均年龄
		select avg(age) from students;
	
		-- 计算平均身高
		select avg(height) from students;


	-- 四舍五入 round(123.23 , 1) 保留1位小数, 四舍五入
	-- 计算所有人的平均年龄，保留2位小数
	select round(avg(age),2) from students;
	
	-- 计算男性的平均身高 保留2位小数


- 在sql 中如何查看帮助文档  ?
- 如何查看函数帮助文档 ? functions;
- ? create, 或者 ? insert 等等都可以查询相关帮助文档;



-- 分组 ,分组的目的就是为了进行聚合统计
​	-- group by 字段
​	-- 查询班级学生的性别
​	select gender from students;

	-- 查看有哪几种性别
	select distinct gender from students;
	
	-- 按照性别分组
	select gender from students group by gender;
	
	-- 计算每种性别中的人数, 聚合函数会作用在分组之后的数据
	select gender,count(*) from students group by gender;
	
	-- group_concat(...)
	# 查询分组数据中的人的姓名
	# 
	select gender, name from students group by gender;
	select gender, group_concat(name) from students group by gender;
	
	一一对应的查询方式: 一个性别对应一个名字
	男 周杰伦
	男 彭于晏
	男 张学友
	...
	女 静香
	女 周杰
	
	一种性别对应这个性别下所有的名字
	男: 周杰伦, 彭于晏, 张学友
	女: 静香, 周杰
	
	-- 查询同种性别中的姓名和身高
	select gender, group_concat(name,":", height) from students group by gender;
	
	-- 计算男性的人数
	select count(*) from students where gender = 1;
	可以实现 但是不够好: select gender, count(*) from students where gender = 1 group by gender;
	
	-- having 需要对分组之后的数据做进一步的筛选
	select gender, count(*) from students group by gender having gender = 1;
	
	-- 除了男生以外的分组的人数
	select gender, count(*) from students group by gender having gender != 1;
	
	-- 查询每种性别中的平均年龄avg(age)
	select gender, avg(age) from students group by gender;
	
	-- 查询每种性别中的平均年龄avg(age), 最大年龄,平均身高,最高身高, 分组是为了更好的统计
	select gender, avg(age),max(age), avg(height), max(height) from students group by gender;	
	
	-- 查询平均年龄超过30岁的性别，以及姓名 , 有哪些性别的平均年龄超过30岁, 将满足条件的性别对应的名字查询到
	select gender,group_concat(name) from students group by gender having avg(age) > 30;


	-- having 和 where 的区别
	where 是对源数据做筛选操作
	having 是对分组之后的数据做进一步的筛选操作, 有having 就一定有group by 有 group by 不一定有having



-- 分页
​	-- limit start, count
​	-- start: 表示从哪里开始查询, start 默认值为0, 可以省略, 跳过多少条数据
​	-- count: 查询多少条

	获取第一页, 每页显示4条数据
	select * from students limit 0,4;
	
	第1页
	select * from students limit 0,4;
	第2页
	select * from students limit 4,4;
	第3页
	select * from students limit 8,4;
	
	第4页
	select * from students limit 12,4;
	
	每页显示的数量: count,  页码信息: page_num,  limit (page_num - 1) * count,count
	
	-- 每页显示4个，显示第3页的信息, 按照年龄从小到大排序 
	错误: select * from students limit 8,4 order by age asc;
	select * from students order by age asc limit 8,4;



-- 连接查询  将两个表按照某种条件合并到一起

	-- 查询学生的名字和学生对应的班级名字
	学生的名字: 学生表
	班级名字: 班级表
	select students.name, classes.name from students,classes;
	select * from students,classes where students.cls_id = classes.id;
	-- 笛卡尔积查询, 会产生很多无用的信息


	-- 连接查询 将两个表中的数据按照设置的连接条件进行筛选, 符合连接条件的数据才能够被筛选出来
	-- table1 inner join table2 on 设置内连接条件  -->  内连接查询
	select * from students inner join classes on students.cls_id = classes.id;


	-- 按照要求显示姓名、和学生对应的班级的名字(学生所在的班级)
	select students.name, classes.name from students inner join classes on students.cls_id = classes.id;
	select s.name, c.name from students as s inner join classes as c on s.cls_id = c.id;
	-- 在以上的查询中，将班级名字显示在第1列
	select c.name, s.name from students as s inner join classes as c on s.cls_id = c.id;
	-- 查询 学生所在的班级, 按照班级进行排序
	select c.name, s.name from students as s inner join classes as c on s.cls_id = c.id order by c.name;
	
	select c.name, group_concat(s.name) from students as s inner join classes as c on s.cls_id = c.id group by c.name asc;
	
	-- 外连接查询: left join + right join
	-- left join 左外连接查询
	-- 查询每位学生对应的班级信息, 不满足连接条件的数据会以NULL填充
	select * from students left join classes on students.cls_id = classes.id;



	-- right join 右外连接查询  使用的比较少
	-- 将数据表名字互换位置，用left join完成
	
	-- 扩充了解, 内连接和外连接的其他写法
	-- 内连接的其他写法
	select * from students join classes on students.cls_id = classes.id;
	select * from students cross join classes on students.cls_id = classes.id;
	
	-- 外连接的其他写法
	select * from students left outer join classes on students.cls_id = classes.id;




-- 自关联  自己关联自己 a inner join a
-- 通过 source 指令导入一个sql文件
​	-- 省级联动 url:http://demo.lanrenzhijia.com/2014/city0605/
​	-- 查询所有省份


	-- 查询出广东省有哪些市
	广东省 广州市
	广东省 深圳市
	...
	--  需要有想象力, 将areas 想象成两张表 , 一张是省表, 一张是市表\
	
	-- 查询出广州市有哪些区县
