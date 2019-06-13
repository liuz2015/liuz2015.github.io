---
layout:     post
title:      "SQL知识点总结"
subtitle:   "《SQL必知必会》阅读笔记"
date:       2018-09-09
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 数据库
    - SQL

---

《SQL必知必会》是一本学习SQL的优秀书籍，本篇文章对其核心知识点进行了总结。

1.IN操作符一般比一组OR操作符执行得更快。IN的最大优点是可以包含其他SELECT语句，能够更动态地建立WHERE子句。

2.ORDER BY 如果想在多个列上进行降序排序，必须对每一列指定DESC关键字。
```sql
SELECT C 
FROM T 
ORDER BY A DESC, B DESC;
```

3.like查找中通配符有 % 和 _ 和 [] ，第二个只能匹配单个字符，第三个匹配指定位置（通配符的位置）的一个字符（sqlserver）。
```sql
SELECT C 
FROM T 
WHERE A LIKE '%asd';
```

4.使用 as 来使用别名，以及使用 + - * /来计算列值。

5.聚集函数：
- AVG() 返回某列的平均值
- COUNT() 返回某列的行数
- MAX() 返回某列的最大值
- MIN() 返回某列的最小值
- SUM() 返回某列值之和
```sql
SELECT COUNT(*), MAX(C1) 
FROM T
```

如果指定列名，则DISTINCT只能用于COUNT()。DISTINCT不能用于COUNT(*)。类似地，DISTINCT必须使用列名，不能用于计算或表达式。

6.日期、文本、数值的处理函数，各类dbms（数据库管理系统）实现不同。

7.GROUP BY：

GROUP BY子句可以包含任意数目的列，因而可以对分组进行嵌套，更细致地进行数据分组。

如果在GROUP BY子句中嵌套了分组，数据将在最后指定的分组上进行汇总。换句话说，在建立分组时，指定的所有列都一起计算（所以不能从个别的列取回数据）。

GROUP BY子句中列出的每一列都必须是检索列或有效的表达式（但不能是聚集函数）。如果在SELECT中使用表达式，则必须在GROUP BY子句中指定相同的表达式。不能使用别名。

大多数SQL实现不允许GROUP BY列带有长度可变的数据类型（如文本或备注型字段）。

除聚集计算语句外，SELECT语句中的每一列都必须在GROUP BY子句中给出。
如果分组列中包含具有NULL值的行，则NULL将作为一个分组返回。如果列中有多行NULL值，它们将分为一组。

GROUP BY子句必须出现在WHERE子句之后，ORDER BY子句之前。

```sql
SELECT C1, COUNT(C2) 
FROM T 
WHERE A>1 
GROUP BY B;
```

8.HAVING：

有关WHERE的所有技术和选项都适用于HAVING。它们的句法是相同的，只是关键字有差别。

WHERE在数据分组前进行过滤，HAVING在数据分组后进行过滤。这是一个重要的区别，WHERE排除的行不包括在分组中。这可能会改变计算值，从而影响HAVING子句中基于这些值过滤掉的分组。

```sql
SELECT C1, COUNT(C2) AS CC 
FROM T 
WHERE A>1 
GROUP BY B 
HAVING C2>100;
```

9.作为**子查询**的SELECT语句只能查询单个列。企图检索多个列将返回错误。

10.内联结（inner join）：查找有对应的行；

外联结（left/right outer join）:查找左/右表的所有行；

自联结（self join）：一个表自己和自己连接。

注意所使用的联结类型。

关于确切的联结语法，应该查看具体的文档，看相应的DBMS支持何种语法（大多数DBMS使用这两课中描述的某种语法）。

保证使用正确的联结条件（不管采用哪种语法），否则会返回不正确的数据。
应该总是提供联结条件，否则会得出笛卡儿积。

在一个联结中可以包含多个表，甚至可以对每个联结采用不同的联结类型。虽然这样做是合法的，一般也很有用，但应该在一起测试它们前分别测试每个联结。这会使故障排除更为简单。

```sql
SELECT T1.C1, T2.C2 
FROM T1 LEFT JOIN T2 ON T1.A=T2.A 
WHERE T1.B>1;
```

11.组合查询：

UNION必须由两条或两条以上的SELECT语句组成，语句之间用关键字UNION分隔（因此，如果组合四条SELECT语句，将要使用三个UNION关键字）。

UNION中的每个查询必须包含相同的列、表达式或聚集函数（不过，各个列不需要以相同的次序列出）。

列数据类型必须兼容：类型不必完全相同，但必须是DBMS可以隐含转换的类型（例如，不同的数值类型或不同的日期类型）。

UNION从查询结果集中自动去除了重复的行；如果想返回所有的匹配行，可使用UNION ALL而不是UNION。

```sql
SELECT C
FROM T1
WHERE A1>1
UNION
SELECT C
FROM T2
WHERE A2>10;
```

12.在用UNION组合查询时，只能使用一条ORDER BY子句，它必须位于最后一条SELECT语句之后，DBMS将用它来排序所有SELECT语句返回的所有结果。

13.如果从另一张表中取得数据插入到这张表中：
```sql
INSERT INTO T2 (
    SELECT *
    FROM T1
    WHERE A>1
);
CREATE TABLE T2 SELECT * FROM T1;#创建新表，同时把数据复制过去
CREATE TABLE T2 SELECT * FROM T1 WHERE 1=2;#只复制结构，不复制数据。
```
该方法要要注意以下几点：
- 任何SELECT选项和子句都可以使用，包括WHERE和GROUP BY；
- 可利用联结从多个表插入数据；
- 不管从多少个表中检索数据，数据都只能插入到一个表中。

14.如果想从表中删除所有行，不要使用DELETE。可使用TRUNCATE TABLE语句，它完成相同的工作，而速度更快（因为不记录数据的变动）。除非确实打算更新和删除每一行，否则绝对不要使用不带WHERE子句的UPDATE或DELETE语句。

15.保证每个表都有主键，尽可能像WHERE子句那样使用它（可以指定各主键、多个值或值的范围）。

16.在UPDATE或DELETE语句使用WHERE子句前，应该先用SELECT进行测试，保证它过滤的是正确的记录，以防编写的WHERE子句不正确。

17.使用强制实施引用完整性的数据库，这样DBMS将不允许删除其数据与其他表相关联的行。有的DBMS允许数据库管理员施加约束，防止执行不带WHERE子句的UPDATE或DELETE语句。如果所采用的DBMS支持这个特性，应该使用它。

18.修改表结构：
```sql
ALTER TABLE T 
ADD/DROP C CHAR(20)/COLUMN C;
```

19.视图：视图是虚拟的表。与包含数据的表不一样，视图只包含使用时动态检索数据的查询。

数据库只存放视图的定义，而不存放视图对应的数据，这些数据仍存放在原来的基本表中。所以基本表中的数据发生变化，从视图中查询出的数据也就随之改变了。从这个意义上讲，视图就像一个窗口，透过它可以看到数据库中自己感兴趣的数据及其变化。

1）重用SQL语句。简化复杂的SQL操作。在编写查询后，可以方便地重用它而不必知道其基本查询细节。

2）使用表的一部分而不是整个表或者是多张表，而用户不必知道具体细节。

3）保护数据。可以授予用户访问表的特定部分的权限，而不是整个表的访问权限。

4）更改数据格式和表示。视图可返回与底层表的表示和格式不同的数据。
```sql
CREATE VIEW ProductCustomers AS
SELECT cust_name, cust_contact, prod_id
FROM Customers, Orders, OrderItems
WHERE Customers.cust_id = Orders.cust_id
AND OrderItems.order_num = Orders.order_num;
DROP VIEW xxx;
```

20.局部变量/全局变量：SQL语言也跟其他编程语言一样，拥有变量，在SQL语言里面把变量分为局部变量和全局变量，全局变量又称系统变量。@value是局部变量，@@value是全局变量。
```sql
declare @SEL_TYPE char(2) --定义局部变量
declare @SEL_CUNT numeric(10) 
set @SEL_TYPE = 'U'
set @SEL_CUNT = 10 

select @SEL_CUNT = COUNT(*) --使用局部变量
from sysobjects 
where type = @SEL_TYPE;
```

21.存储过程：一组为了完成特定功能的SQL语句集，它存储在数据库中，一次编译后永久有效，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。
```sql
#新增学生信息
IF OBJECT_ID (N'PROC_INSERT_STUDENT', N'P') IS NOT NULL
    DROP procedure PROC_INSERT_STUDENT;
GO
CREATE procedure PROC_INSERT_STUDENT
    @id int,
    @name nvarchar(20),
    @age int,
    @city nvarchar(20)
AS
    INSERT INTO Students(ID,Name,Age,City) VALUES(@id,@name,@age,@city)
GO

EXEC PROC_INSERT_STUDENT 1001,N'张三',19,'ShangHai'

#根据学生ID，更新学生信息
IF OBJECT_ID (N'PROC_UPDATE_STUDENT', N'P') IS NOT NULL
    DROP procedure PROC_UPDATE_STUDENT;
GO
CREATE procedure PROC_UPDATE_STUDENT
    @id int,
    @name nvarchar(20),
    @age int,
    @city nvarchar(20)
AS
    UPDATE Students SET Name=@name,Age=@age,City=@city WHERE ID=@id
GO

EXEC PROC_UPDATE_STUDENT 1001,N'张思',20,'ShangHai'


#根据ID，删除某学生记录
IF OBJECT_ID (N'PROC_DELETE_STUDENT_BY_ID', N'P') IS NOT NULL
    DROP procedure PROC_DELETE_STUDENT_BY_ID;
GO
CREATE procedure PROC_DELETE_STUDENT_BY_ID
    @id int
AS
    DELETE FROM  Students WHERE ID=@id
GO

EXEC PROC_DELETE_STUDENT_BY_ID 1001
```

22.事务：事务是一种机制、一种操作序列，它包含了一组数据库操作命令，这组命令要么全部执行，要么全部不执行。因此事务是一个不可分割的工作逻辑单元。在数据库系统上执行并发操作时事务是作为最小的控制单元来使用的。

事务四大属性：
- 原子性(Atomicity):事务是一个完整的操作。
- 一致性(Consistency)：当事务完成时，数据必须处于一致状态。
- 隔离性(Isolation):对数据进行修改的所有并发事务是彼此隔离的。
- 持久性(Durability):事务完成后，它对于系统的影响是永久性的。

```sql
begin transaction#开始事务 
commit transaction#提交事务
rollback transaction#回滚事务 
```

23.游标：DECLARE命名游标，并定义相应的SELECT语句，根据需要带WHERE和其他子句。在一个游标被打开后，可以使用FETCH语句分别访问它的每一行。FETCH指定检索什么数据（所需的列），检索出来的数据存储在什么地方。
```sql
--创建一个游标
declare my_cursor cursor for     --my_cursor为游标的名称
select id,name from my_user      --这是游标my_cursor的值
--打开游标
open my_cursor                  
--变量
declare   @id int               --声明变量  ‘declare’为声明变量 ‘@name’为变量名称 后面为变量类型
declare   @name varchar(50)     --这里是两个变量用来接收游标的值
--循环游标
fetch next from my_cursor into @id,@name  --获取my_cursor的下一条数据，其中为两个字段分别赋值给@id,@name
while @@FETCH_STATUS=0 --假如检索到了数据继续执行
begin
print(@name) --print()打印变量
select * from my_user where id=@id --再执行一次查询 
fetch next from my_cursor into @id,@name --获取下一条数据并赋值给变量
end--关闭释放游标
close my_cursor
deallocate my_cursor
```
24.索引：

索引改善检索操作的性能，但降低了数据插入、修改和删除的性能。在执行这些操作时，DBMS必须动态地更新索引。

索引数据可能要占用大量的存储空间。

并非所有数据都适合做索引。取值不多的数据（如州）不如具有更多可能值的数据（如姓或名），能通过索引得到那么多的好处。

索引用于数据过滤和数据排序。如果你经常以某种特定的顺序排序数据，则该数据可能适合做索引。

可以在索引中定义多个列（例如，州加上城市）。这样的索引仅在以州加城市的顺序排序时有用。如果想按城市排序，则这种索引没有用处。

25.触发器：一种特殊类型的存储过程，它在指定的表中的数据发生变化时自动生效。
```sql
CREATE TRIGGER trigger_name
ON table|view
FOR|INSTEAD OF|AFTER [DELETE][,INSERT][,UPDATE]
AS
Sql_statement[…n]
```
创建名为trigger_name的新触发器。触发器可在删除、插入、更新操作发生之前（FOR）或之后（AFTER）执行，或者代替执行（INSTEAD OF）。
```sql
#在表Student中建立删除触发器，实现表Student和表SC的级联删除，也就是只要删除表Student中的元组学号为s1，则表SC中SNO为s1的元组也要删除；建立完触发器后用企业管理器删除Student中学号为30的元组，看看表SC中SNO为30的选课记录是否也一起删除。
create trigger t_std2 on student
instead of delete
as
begin
  declare @id char(5)
  select @id=sno from deleted
  delete from sc where SNo =@id
  delete from student where SNo=@id
  end
  go
 
delete from Student where SNo='00002'
```

26.正则表达式：
```sql
 SELECT C FROM T WHERE A regexp 'xxx'
```

27.全文本搜索，两个最常使用的引擎为MyISAM和InnoDB，前者支持全文本搜索，而后者不支持。
要使用全文本搜索的话，创建表时，engine=MyISAM；使用两个函数Match()和Against()执行全文本搜索，其中Match()指定被搜索的列，Against()指定要使用的搜索表达式。

28.使用CASE关键字进行判断操作：
```sql
update salary 
set male=(
    case 'f' 
        then 'm' 
        else 'f' 
    end
    );
```

29.mod()可用于对数值求模

30.IFNULL() 可以对NULL值进行处理：
```sql
SELECT 
ProductName,UnitPrice*(UnitsInStock+IFNULL(UnitsOnOrder,0))#如果是NULL就设为0
FROM Products
```

## 参考资料
- 《SQL必知必会》

