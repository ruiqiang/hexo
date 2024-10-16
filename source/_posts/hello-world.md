---
title: MYSQL8.0数据库外键分享
---
在mysql3.23.44版本中InnoDB引擎首先支持了外键.

### 为什么要引入外键
{% note [warning]%}
为了维护数据库的完整性和一致性。‌
{% endnote %}

### 外键的创建

``` bash
# 1. 建表语句式直接创建
CREATE TABLE child_table (

    id INT PRIMARY KEY AUTO_INCREMENT,

    parent_id INT,

    -- 其他字段...

    FOREIGN KEY (parent_id) REFERENCES parent_table(id)

) ENGINE=InnoDB;

# 2.增量式单独加外键
ALTER TABLE child_table

ADD CONSTRAINT fk_child_parent FOREIGN KEY (parent_id) REFERENCES parent_table(id)

# 这部分可以不加
ON DELETE CASCADE ON UPDATE CASCADE;
```

### 准备DEMO数据

``` bash
# 班级表
create table t_class (
	id bigint not null auto_increment,
	class_name varchar(50) not null,
	primary key (id)
) engine = innodb auto_increment = 1 default charset = utf8mb4 collate = utf8mb4_general_ci;

# 学生表
create table t_student (
	id bigint not null auto_increment,
	stu_name varchar(50) not null,
	stu_age int not null,
	class_id bigint not null,
	primary key (id)
) engine = innodb auto_increment = 1 default charset = utf8mb4 collate = utf8mb4_general_ci;
```

### 要建立的外键关系

![foreign_key_table_relation.png](/images/foreign_key_table_relation.png)

{% note [warning]%}
1. t_class 是外键关系中的主表, t_student 是外键关系中的子表
2. 我们现在要创建一个外键关系,t_student(class_id) -> t_class(id)
3. 按照mysql的规定,外键的关系是由子表来创建的
{% endnote %}

### 外键的限制

| 序号 | 规则                                                                                                        
|----|:----------------------------------------------------------------------------------------------------------|
| 1  | 主表和子表的存储引擎必须一致,并且不能是临时表；                                                                                  |
| 2  | 主表外键字段和子表外键字段必须有兼容的类型；<br/>相似数字类型的长度和小数点位数必须一致；<br/>字符串类型的长度不需要一致,但是字符集必须一致；                              |
| 3  | 建立外键的主表和子表的字段上必须存在索引,以防止全表扫描；<br/>创建外键关系时，子表字段无索引时mysql会自动创建一个索引；<br/>并且在后续这个字段被加上可用的其他索引时静默的删掉这个自动创建的索引； |
| 4  | BLOB和TEXT字段不允许作为外键关联的字段；                                                                                  |
| 5  | 当表已经存在外键关系时，不可修改表的引擎；                                                                                     |

### 外键的级联特性

| 序号 | 约束名称                 | 解释                                                                                    |
|----|:---------------------|:--------------------------------------------------------------------------------------|
| 1  | CASCADE              | 主表中的某行数据被删除或更新时，所有与之关联的子表中的行也将联动被删除或更新。                                               |
| 2  | NO ACTION(默认)        | 主表删除或更新一行数据时，如果该行在子表中有对应关联项，操作会被拒绝，抛出错误;<br/>子表删除或更新一行数据时，如果该行在主表中有对应关联项，操作会被拒绝，抛出错误; |
| 3  | RESTRICT(=NO ACTION) | 主表删除或更新一行数据时，如果该行在子表中有对应关联项，操作会被拒绝，抛出错误;<br/>子表删除或更新一行数据时，如果该行在主表中有对应关联项，操作会被拒绝，抛出错误; |
| 4  | SET NULL             | 主表删除或更新一行数据时，如果该行在子表中有对应关联项，子表中对应的外键字段会被设置为NULL。                                              |

### 一些测试(针对RESTRICT约束)
```
# 创建外键关系
mysql> ALTER TABLE t_student ADD CONSTRAINT fk_class_id FOREIGN KEY (class_id) REFERENCES t_class(id);
Query OK, 0 rows affected (0.02 sec)

# 查询子表索引,多了一条自动给字段class_id创建的索引
mysql> show index from t_student;
+-----------+------------+-------------+--------------+-------------+-----------+------------+
| Table     | Non_unique | Key_name    | Seq_in_index | Column_name | Collation | Index_type |
+-----------+------------+-------------+--------------+-------------+-----------+------------+
| t_student | 0          | PRIMARY     | 1            | id          | A         | BTREE      |
| t_student | 1          | fk_class_id | 1            | class_id    | A         | BTREE      |
+-----------+------------+-------------+--------------+-------------+-----------+------------+

# 查询外键(只列出了部分字段)
mysql> select * from INFORMATION_SCHEMA.KEY_COLUMN_USAGE where REFERENCED_TABLE_NAME = 't_class'
+-----------------+------------+-------------+-----------------------+------------------------+
| CONSTRAINT_NAME | TABLE_NAME | COLUMN_NAME | REFERENCED_TABLE_NAME | REFERENCED_COLUMN_NAME |
+-----------------+------------+-------------+-----------------------+------------------------+
| fk_class_id     | t_student  | class_id    | t_class               | id                     |
+-----------------+------------+-------------+-----------------------+------------------------+
```
### 建立外键后的数据约束
{% note [warning]%}
<b style="color:Orange;">约束1: 子表创建的外键字段值超出了主键表数据对应外键的值</b>
{% endnote %}
```
# 此时主表子表都是空表
mysql> insert into t_student (id, stu_name, stu_age, class_id) values (1, "小苹果", 4, 1);
SQL 错误 [1452] [23000]: Cannot add or update a child row: a foreign key constraint fails
(`test`.`t_student`, CONSTRAINT `fk_class_id` FOREIGN KEY (`class id`) REFERENCES `t_class`(`id`))

# 添加外键后主键外键字段要增加新的值必须先添加主表记录
mysql> insert into t_class (id, class_name) values (1, "2年级1班");
Query OK, 1 rows affected (0.02 sec)
mysql>insert into t_student (id, stu_name, stu_age, class_id) values (1, "小苹果", 4, 1);
Query OK, 1 rows affected (0.02 sec)
```

{% note [warning]%}
<b style="color:Orange;">约束2: 主表外键记录被子表引用时无法删除，必须先删除子表引用记录后才能删除</b>
{% endnote %}
```
mysql> delete from t_class where id = 1;
SOL错误[1451] [23000]: Cannot delete or update a parent row: a foreign key constraint 
fails (`test`.`t_tudent`, CONSTRAINT `fk_class_id` FOREIGN KEY (`class_id`)
REFERENCES `t_class`(`id`))

mysql> delete from t_student where id = 1;
Query OK, 1 rows affected (0.02 sec)
mysql> delete from t_class where id = 1;
Query OK, 1 rows affected (0.02 sec)
```

{% note [warning]%}
<b style="color:Orange;">总结: 外键一旦创建
新增记录的顺序按照主表->子表的顺序来创建.
删除的顺序按照子表->主表的顺序来删除</b>
{% endnote %}