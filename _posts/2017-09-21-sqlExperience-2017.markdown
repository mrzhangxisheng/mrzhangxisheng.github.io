---
layout:     post
title:      "sql优化心得"
subtitle:   " \"学无止境\""
date:       2017-09-20 16:27:00
author:     "Zhangxs"
header-img: "null.jpg"
catalog: true
tags:
    - 技术
---



## 概述
sql优化谈谈你的见解。。。这些问题我特么的被问过好多遍了，都不怎么能答上来。。。所以花心思学了一阵子。。

<p id = "build"></p>
---

## 博文
谈到sql优化，不管会不会的人，都知道要建索引。例：

```
    
        CREATE INDEX index_name  ON table_name (index_col_name,...)普通索引   ...多字段代表复合索引
        CREATE UNIQUE INDEX index_name  ON table_name (index_col_name,...) 唯一索引（主键也是）
        CREATE FULLTEXT INDEX index_name  ON table_name (index_col_name,...) 全文索引
        ALTER TABLE `table_name` ADD FULLTEXT ( column,...)  全文索引
        ALTER TABLE `table_name` ADD INDEX index_name ( column,... ) 普通索引
```

---
上面就是两种创建索引的方法。（太简单，就不多解释）。
######为什么建立索引后查询会变快？而更新就会受影响？
这里就首先谈一谈，mysql索引的处理（这里只讲B+TREE类型的）

索引索引就是mysql为高效获取数据维护的一个数据结构。也就是说，更新数据的同时，mysql还会同事更新索引结构。

引擎InnoDB和MyISAM索引实现都是b+tree作为索引结构
区别

MyISAM的索引方式叫做“非聚集”，InnoDB的聚集索引

MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而在InnoDB中，表数据文件本身就是按B+Tree组织的一个索引结构，这棵树的叶节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。

MyISAM例图:![Alt text](/img/2017-sqlpost-1.jpg)

InnoDB中例图:![Alt text](/img/2017-sqlpost-2.jpg)

具体可以看看这篇[博客](http://blog.jobbole.com/24006/)。

---
#######索引优化肯定离不开Explain(sql的解析)

explain参数解析

id:sql的执行顺序。id相同顺序从上而下，id不同顺序成降序（大到小）。

select_type
  
1. simple 它表示简单的select,没有union和子查询
2. primary 最外面的select,在有子查询的语句中，最外面的select查询就是primary
3. SUBQUERY 在SELECT或WHERE列表中包含了子查询
4. 在FROM列表中包含的子查询被标记为：DERIVED（衍生）用来表示包含在from子句中的子查询的select，mysql会递归执行并将结果放到一个临时表中。服务器内部称为"派生表"，因为该临时表是从子查询中派生出来的
5. 若第二个SELECT出现在UNION之后，则被标记为UNION；若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED
6. 从UNION表获取结果的SELECT被标记为：UNION RESULT

table:显而易见，就是所用到的数据表

type (ALL, index,  range, ref, eq_ref, const, system)从左到右，优化等级越高

1. system const类型的特例，当查询的表只有一行的情况下。
2. const  表最多有一个匹配行，const用于比较primary key 或者unique索引。因为只匹配一行数据，所以很快
记住一定是用到primary key 或者unique，并且只检索出一条数据的 情况下才会是const.如下图plan_id是主键

```

	mysql> explain select plan_id from plan where plan_id=14;
	+----+-------------+-------+-------+-----------------------+---------+---------+-------+------+-------------+
	| id | select_type | table | type  | possible_keys         | key     | key_len | ref   | rows | Extra       |
	+----+-------------+-------+-------+-----------------------+---------+---------+-------+------+-------------+
	|  1 | SIMPLE      | plan  | const | PRIMARY,index_plan_id | PRIMARY | 4       | const |    1 | Using index |
	+----+-------------+-------+-------+-----------------------+---------+---------+-------+------+-------------+ 
```

3. eq_ref 对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件

```

	mysql> explain select plan.plan_id from plan_record left join plan on plan_record.plan_id=plan.plan_id;
	+----+-------------+-------------+--------+-----------------------+---------------+---------+------------------------------------+------+-------------+
	| id | select_type | table       | type   | possible_keys         | key           | key_len | ref                                | rows | Extra       |
	+----+-------------+-------------+--------+-----------------------+---------------+---------+------------------------------------+------+-------------+
	|  1 | SIMPLE      | plan_record | index  | NULL                  | index_plan_id | 4       | NULL                               |   89 | Using index |
	|  1 | SIMPLE      | plan        | eq_ref | PRIMARY,index_plan_id | PRIMARY       | 4       | granary_manage.plan_record.plan_id |    1 | Using index |
	+----+-------------+-------------+--------+-----------------------+---------------+---------+------------------------------------+------+-------------+
```

4. ref 使用非唯一索引扫描或者唯一索引的前缀扫描，返回匹配某个单独值的记录行

```

	mysql> explain select plan_name from plan where remark='14';
	+----+-------------+-------+------+---------------+--------------+---------+-------+------+-----------------------+
	| id | select_type | table | type | possible_keys | key          | key_len | ref   | rows | Extra                 |
	+----+-------------+-------+------+---------------+--------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | plan  | ref  | index_remark  | index_remark | 303     | const |    1 | Using index condition |
	+----+-------------+-------+------+---------------+--------------+---------+-------+------+-----------------------+
```
5. range 索引范围扫描，对索引的扫描开始于某一点，返回匹配值域的行。显而易见的索引范围扫描是带有between或者where子句里带有<, >查询。当mysql使用索引去查找一系列值时，例如IN()和OR列表，也会显示range（范围扫描）,当然性能上面是有差异的。

```

	mysql> explain select * from plan where plan_id>14;
	+----+-------------+-------+-------+-----------------------+---------+---------+------+------+-------------+
	| id | select_type | table | type  | possible_keys         | key     | key_len | ref  | rows | Extra       |
	+----+-------------+-------+-------+-----------------------+---------+---------+------+------+-------------+
	|  1 | SIMPLE      | plan  | range | PRIMARY,index_plan_id | PRIMARY | 4       | NULL |   39 | Using where |
	+----+-------------+-------+-------+-----------------------+---------+---------+------+------+-------------+
```
6. index  该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和Index都是读全表，但index是从索引中读取的，而all是从硬盘中读的）当查询只使用作为单索引一部分的列时，MySQL可以使用该联接类型。

```

		mysql> explain select * from plan;
		+----+-------------+-------+------+---------------+------+---------+------+------+-------+
		| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
		+----+-------------+-------+------+---------------+------+---------+------+------+-------+
		|  1 | SIMPLE      | plan  | ALL  | NULL          | NULL | NULL    | NULL |   40 | NULL  |
		+----+-------------+-------+------+---------------+------+---------+------+------+-------+
		
		mysql> explain select plan_id from plan;
		+----+-------------+-------+-------+---------------+---------------+---------+------+------+-------------+
		| id | select_type | table | type  | possible_keys | key           | key_len | ref  | rows | Extra       |
		+----+-------------+-------+-------+---------------+---------------+---------+------+------+-------------+
		|  1 | SIMPLE      | plan  | index | NULL          | index_plan_id | 4       | NULL |   40 | Using index |
		+----+-------------+-------+-------+---------------+---------------+---------+------+------+-------------+
		mysql> show profiles;--性能查看如果不起作用执行set profiling=1;开启这个功能。
		+----------+------------+-------------------------------------------+
		| Query_ID | Duration   | Query                                     |
		+----------+------------+-------------------------------------------+
		|        1 |  0.0005255 | explain select * from plan                |
		|        2 | 0.00042275 | explain select plan_id from plan          |
		+----------+------------+-------------------------------------------+
```

7. all 全局扫描（如果数据量很大比如超过3000条那么就需要优化了）

 possible_keys
 
 提示可能会使用那些索引，就是mysql能找到的。

 key 

 mysql使用的索引，简单且重要


 key_len
 
 MYSQL使用的索引长度（越短约好，但不要吝啬。）

 ref

 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

 rows

 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数

 extra(介绍关键的几个)
 
1. Using index:
该值表示相应的select操作中使用了覆盖索引（Covering Index）
这种情况禁忌写（select *）MySQL可以利用索引返回select列表中的字段，而不必根据索引再次读取数据文件（这也就是为什么查询是不用select * from table）

2. Using where:不多说就是用到了where
3. Using temporary:用到了临时表（注意能优化，尽量优化）

```

		mysql> explain select * from plan union select * from plan;
		+------+--------------+------------+------+---------------+------+---------+------+------+-----------------+
		| id   | select_type  | table      | type | possible_keys | key  | key_len | ref  | rows | Extra           |
		+------+--------------+------------+------+---------------+------+---------+------+------+-----------------+
		|    1 | PRIMARY      | plan       | ALL  | NULL          | NULL | NULL    | NULL |   40 | NULL            |
		|    2 | UNION        | plan       | ALL  | NULL          | NULL | NULL    | NULL |   40 | NULL            |
		| NULL | UNION RESULT | <union1,2> | ALL  | NULL          | NULL | NULL    | NULL | NULL | Using temporary |
		+------+--------------+------------+------+---------------+------+---------+------+------+-----------------+ 

```
4.  Using filesort
外部排序 MySQL中无法利用索引完成的排序操作称为“文件排序”。如果是复合索引，你没有遵循最左前缀原则。你建立的索引就不会用到。

```

	mysql>explain select id,age from t1 order by name; 
	+----+-------------+-------+------+---------------+------+---------+------+------+----------------+
	| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra          |
	+----+-------------+-------+------+---------------+------+---------+------+------+----------------+
	|  1 | SIMPLE      | t1    | ALL  | NULL          | NULL | NULL    | NULL |    4 | Using filesort |
	+----+-------------+-------+------+---------------+------+---------+------+------+----------------+
	mysql>explain select id,age from t1 order by age; 
	+----+-------------+-------+-------+---------------+------+---------+------+------+-------------+
	| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+-------+-------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | t1    | index | NULL          | age  | 5       | NULL |    4 | Using index |
	+----+-------------+-------+-------+---------------+------+---------+------+------+-------------+
```

5. Using join buffer
改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。

6.  Impossible where
没有符合条件的数据

explain 基本参数解释完了。

查看该字段是否有建立索引的必要(必要性有多大)。

```
SELECT count(DISTINCT(colName))/count(*) AS Selectivity FROM tableName;
```

---

######sql优化例子

######最左前缀
 建表sql,注意我建立了一个复合索引顺序是（name age sex） phone是没有建立索引（为了更有效的突出最左前缀问题，不然会出现索引覆盖查询等级会变成index）

```

	CREATE TABLE `person` (
	  `id` int(11) NOT NULL AUTO_INCREMENT,
	  `name` varchar(255) DEFAULT NULL,
	  `age` varchar(255) DEFAULT NULL,
	  `sex` varchar(255) DEFAULT NULL,
	  `phone` varchar(255) DEFAULT NULL,
	  PRIMARY KEY (`id`),
	  KEY `idx_name_age_sex` (`name`,`age`,`sex`)
	) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
```

全匹配：就是复合索引的所有字段都出现（不包括！=、<>、<、>等范围比较）。就不需要考虑where条件后面的顺序。mysql优化器会自动排序去最优。
例：

```

	mysql> explain select * from person where name='张三' and age=2 and sex='男';
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	1 row in set
	mysql> explain select * from person where name='张三' and sex='男' and age=2;
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	1 row in set
	mysql> explain select * from person where sex='男' and name='张三' and age=2;
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	1 row in set
```

根据解析器的返回可以看出三者都一样。没有顺序的问题（也就是没有最左前缀问题）。

最左前缀匹配:复合索引最左边字段开头

```

	mysql> explain select * from person where name='张三' and sex='男';
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	1 row in set
	
	mysql> explain select * from person where name='张三' and age=2;
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
```

可以看出都是ref级别的查询。如果我们不以name开头查询。。。会出现奇迹

```

	mysql> explain select * from person where age=2;
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using where |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select * from person where age=2 and sex='男';
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using where |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select * from person where sex='男';
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using where |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
```

看出来都变成全局扫描了all。。rows：变大了 key：没有用到索引。。所以如果查询是复合索引尽量满足最左前缀原则（按照建立索引的顺序来排）

#######like查询

```

	mysql> explain select * from person where name like '%张三';
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using where |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select * from person where name like '%张三%';
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using where |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select * from person where name like '张三%';
	+----+-------------+--------+-------+------------------+------------------+---------+------+------+-----------------------+
	| id | select_type | table  | type  | possible_keys    | key              | key_len | ref  | rows | Extra                 |
	+----+-------------+--------+-------+------------------+------------------+---------+------+------+-----------------------+
	|  1 | SIMPLE      | person | range | idx_name_age_sex | idx_name_age_sex | 768     | NULL |    1 | Using index condition |
	+----+-------------+--------+-------+------------------+------------------+---------+------+------+-----------------------+
	1 row in set
```

可以看出只有%在最右边的时候查询等级是range用到了索引。其他情况索引是失效的。



####### !=、<>、 not in、in索引失效问题（例子是在遵循最左前缀的前提下进行）

```

	mysql> explain select * from person where name<>'张三';
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys    | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | idx_name_age_sex | NULL | NULL    | NULL |   24 | Using where |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select * from person where name!='张三';
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys    | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | idx_name_age_sex | NULL | NULL    | NULL |   24 | Using where |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
```

可以看出<> != 是索引失效的。

```

	mysql> explain select * from person where name in('张三');
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	1 row in set
	
	mysql> explain select * from person where name not in('张三');
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys    | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | idx_name_age_sex | NULL | NULL    | NULL |   24 | Using where |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select * from person where name in('张三','李四');
	+----+-------------+--------+-------+------------------+------------------+---------+------+------+-----------------------+
	| id | select_type | table  | type  | possible_keys    | key              | key_len | ref  | rows | Extra                 |
	+----+-------------+--------+-------+------------------+------------------+---------+------+------+-----------------------+
	|  1 | SIMPLE      | person | range | idx_name_age_sex | idx_name_age_sex | 768     | NULL |    2 | Using index condition |
	+----+-------------+--------+-------+------------------+------------------+---------+------+------+-----------------------+
	1 row in set
```
 
in、not in 仔细发现not in 是索引失效的，而in在里面参数超过两个变成了范围查询查询等级变成range，只有一个时等同于 = conts查询


```

	mysql> explain select * from person where name is null;
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | Extra                 |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	|  1 | SIMPLE      | person | ref  | idx_name_age_sex | idx_name_age_sex | 768     | const |    1 | Using index condition |
	+----+-------------+--------+------+------------------+------------------+---------+-------+------+-----------------------+
	1 row in set
	
	mysql> explain select * from person where name is not null;
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	| id | select_type | table  | type | possible_keys    | key  | key_len | ref  | rows | Extra       |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | ALL  | idx_name_age_sex | NULL | NULL    | NULL |   24 | Using where |
	+----+-------------+--------+------+------------------+------+---------+------+------+-------------+
	1 row in set
```

is null、 is not null，前者是没有失效的等同于= conts查询 ，后者是失效的。

---


########覆盖索引

```

	mysql> explain select * from person ;
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |   24 | NULL  |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	1 row in set
	
	mysql> explain select id,name,age,sex,phone from person ;
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |   24 | NULL  |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	1 row in set
	
	mysql> explain select name,age,sex,phone from person ;
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL |   24 | NULL  |
	+----+-------------+--------+------+---------------+------+---------+------+------+-------+
	1 row in set
	
	mysql> explain select name,age,sex from person ;
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	| id | select_type | table  | type  | possible_keys | key              | key_len | ref  | rows | Extra       |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | index | NULL          | idx_name_age_sex | 2304    | NULL |   24 | Using index |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	1 row in set

	mysql> explain select name,age from person;
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	| id | select_type | table  | type  | possible_keys | key              | key_len | ref  | rows | Extra       |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | index | NULL          | idx_name_age_sex | 2304    | NULL |   24 | Using index |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select age,name from person;
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	| id | select_type | table  | type  | possible_keys | key              | key_len | ref  | rows | Extra       |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | index | NULL          | idx_name_age_sex | 2304    | NULL |   24 | Using index |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	1 row in set
	
	mysql> explain select age,sex,name from person;
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	| id | select_type | table  | type  | possible_keys | key              | key_len | ref  | rows | Extra       |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | index | NULL          | idx_name_age_sex | 2304    | NULL |   24 | Using index |
	+----+-------------+--------+-------+---------------+------------------+---------+------+------+-------------+
	1 row in set
```

可以看出select 后面不写* 查询的字段在索引中有（顺序不重要），因为possible_keys并没有可以用的索引，所以sql优化器自己找到这个索引，提高速度。index虽然等级不高但总比all好。

order by 

```

	mysql> explain select * from person ORDER BY id;
	+----+-------------+--------+-------+---------------+---------+---------+------+------+-------+
	| id | select_type | table  | type  | possible_keys | key     | key_len | ref  | rows | Extra |
	+----+-------------+--------+-------+---------------+---------+---------+------+------+-------+
	|  1 | SIMPLE      | person | index | NULL          | PRIMARY | 4       | NULL |   24 | NULL  |
	+----+-------------+--------+-------+---------------+---------+---------+------+------+-------+
	1 row in set

	mysql> explain select * from person order by name,age,sex
	    -> ;
	+----+-------------+--------+------+---------------+------+---------+------+------+----------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra          |
	+----+-------------+--------+------+---------------+------+---------+------+------+----------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL | 3026 | Using filesort |
	+----+-------------+--------+------+---------------+------+---------+------+------+----------------+
	1 row in set
	mysql>  explain select name,age,sex from person order by name,age,sex;
	+----+-------------+--------+-------+---------------+-------------------+---------+------+------+-------------+
	| id | select_type | table  | type  | possible_keys | key               | key_len | ref  | rows | Extra       |
	+----+-------------+--------+-------+---------------+-------------------+---------+------+------+-------------+
	|  1 | SIMPLE      | person | index | NULL          | didx_name_age_sex | 2304    | NULL | 2649 | Using index |
	+----+-------------+--------+-------+---------------+-------------------+---------+------+------+-------------+
	1 row in set
	mysql>  explain select name,age,sex,phone from person order by name,age,sex;
	+----+-------------+--------+------+---------------+------+---------+------+------+----------------+
	| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra          |
	+----+-------------+--------+------+---------------+------+---------+------+------+----------------+
	|  1 | SIMPLE      | person | ALL  | NULL          | NULL | NULL    | NULL | 2649 | Using filesort |
	+----+-------------+--------+------+---------------+------+---------+------+------+----------------+
	1 row in set
```
可以看出order by ，select *查询出来id  其他的索引都是用不上的。必须在查询字段中包含索引字段。。但出现其他字段索引也是失效的。（select的字段，必须是索引字段。（主键查询除外））

```
		
		mysql>  explain select id,name from person order by id,name;
		+----+-------------+--------+-------+---------------+-------------------+---------+------+------+-----------------------------+
		| id | select_type | table  | type  | possible_keys | key               | key_len | ref  | rows | Extra                       |
		+----+-------------+--------+-------+---------------+-------------------+---------+------+------+-----------------------------+
		|  1 | SIMPLE      | person | index | NULL          | didx_name_age_sex | 2304    | NULL | 2649 | Using index; Using filesort |
		+----+-------------+--------+-------+---------------+-------------------+---------+------+------+-----------------------------+
		1 row in set
```

由于MySQL的每一条简单查询只应用一个索引，所以，这个时候使用order by 主键，主键的索引功能失效。

#######索引选择性与前缀索引

既然索引可以加快查询速度，那么是不是只要是查询语句需要，就建上索引？答案是否定的。因为索引虽然加快了查询速度，但索引也是有代价的：索引文件本身要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，另外，MySQL在运行时也要消耗资源维护索引，因此索引并不是越多越好。一般两种情况下不建议建索引。

第一种情况是表记录比较少，例如一两千条甚至只有几百条记录的表，没必要建索引，让查询做全表扫描就好了。至于多少条记录才算多，这个个人有个人的看法，我个人的经验是以2000作为分界线，记录数不超过 2000可以考虑不建索引，超过2000条可以酌情考虑索引。

另一种不建议建索引的情况是索引的选择性较低。所谓索引的选择性（Selectivity），是指不重复的索引值（也叫基数，Cardinality）与表记录数（#T）的比值：

Index Selectivity = Cardinality / #T

显然选择性的取值范围为(0, 1]，选择性越高的索引价值越大，这是由B+Tree的性质决定的。

```

	mysql> SELECT count(DISTINCT(phone))/count(*) AS Selectivity FROM person;
	+-------------+
	| Selectivity |
	+-------------+
	| 1.0000      |
	+-------------+
	1 row in set
```

phone的选择性已经到1了，所以实在必要为其单独建索引。（最大为1，越大约有必要建，自己斟酌）


有一种与索引选择性有关的索引优化策略叫做前缀索引，就是用列的前缀代替整个列作为索引key，当前缀长度合适时，可以做到既使得前缀索引的选择性接近全列索引，同时因为索引key变短而减少了索引文件的大小和维护开销。

可以看到employees表只有一个索引<emp_no>，那么如果我们想按名字搜索一个人，就只能全表扫描了


```

	EXPLAIN SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido';
	+----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
	| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
	+----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
	|  1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 300024 | Using where |
	+----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
```

如果频繁按名字搜索员工，这样显然效率很低，因此我们可以考虑建索引。有两种选择，建<first_name>或<first_name, last_name>，看下两个索引的选择性


```


	SELECT count(DISTINCT(first_name))/count(*) AS Selectivity FROM employees.employees;
	+-------------+
	| Selectivity |
	+-------------+
	|      0.0042 |
	+-------------+
	 
	SELECT count(DISTINCT(concat(first_name, last_name)))/count(*) AS Selectivity FROM employees.employees;
	+-------------+
	| Selectivity |
	+-------------+
	|      0.9313 |
	+-------------+
```


<first_name>显然选择性太低，<first_name, last_name>选择性很好，但是first_name和last_name加起来长度为30，有没有兼顾长度和选择性的办法？可以考虑用first_name和last_name的前几个字符建立索引，例如<first_name, left(last_name, 3)>，看看其选择性


```

	SELECT count(DISTINCT(concat(first_name, left(last_name, 3))))/count(*) AS Selectivity FROM employees.employees;
	+-------------+
	| Selectivity |
	+-------------+
	|      0.7879 |
	+-------------+
```



选择性还不错，但离0.9313还是有点距离，那么把last_name前缀加到4


```

	SELECT count(DISTINCT(concat(first_name, left(last_name, 4))))/count(*) AS Selectivity FROM employees.employees;
	+-------------+
	| Selectivity |
	+-------------+
	|      0.9007 |
	+-------------+
```

这时选择性已经很理想了，而这个索引的长度只有18，比<first_name, last_name>短了接近一半，我们把这个前缀索引 建上

```

	ALTER TABLE employees.employees
	ADD INDEX `first_name_last_name4` (first_name, last_name(4));
```

此时再执行一遍按名字查询，比较分析一下与建索引前的结果

```

	SHOW PROFILES;
	+----------+------------+---------------------------------------------------------------------------------+
	| Query_ID | Duration   | Query                                                                           |
	+----------+------------+---------------------------------------------------------------------------------+
	|       87 | 0.11941700 | SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido' |
	|       90 | 0.00092400 | SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido' |
	+----------+------------+---------------------------------------------------------------------------------+

```

性能的提升是显著的，查询速度提高了120多倍。

前缀索引兼顾索引大小和查询速度，但是其缺点是不能用于ORDER BY和GROUP BY操作，也不能用于Covering index（即当索引本身包含查询所需全部数据时，不再访问数据文件本身）。


## 后记

这是一两个星期的成果，用两天空余时间才写完。。还是蛮大的收获的。。这里也就只涉及优化入门吧。。在深入下去就的转行列做DBA了。。

学无止境。Tomorrow Will Be Better

—— Zhangxs 后记于 2017.0921
