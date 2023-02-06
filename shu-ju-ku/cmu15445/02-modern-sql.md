# 02-Modern SQL

## 1. SQL History

1971 年，IBM 创建了它的第一个关系查询语言，名为 _SQUARE_。&#x20;

随后，IBM 在 1972 年为 IBM 系统 R 原型 DBMS 创建了 "SEQUEL：结构化英语查询语言 (Structured English Query Language)

之后 IBM 又发布了基于 SQL 的商业 DBMS：System/38（1979），SQL/DS（1981）和 DB2 (1983)。

1986年，ANSI 标准。1987年为 ISO 标准：结构化查询语言 (Structured Query Language)

目前的标准是 SQL:2016&#x20;

* SQL:2016 → JSON, Polymorphic tables&#x20;
* SQL:2011 → Temporal DBs, Pipelined DML
* SQL:2008 → Truncation, Fancy Sorting
* SQL:2003 → XML, Windows, Sequences, Auto-Gen IDs.
* SQL:1999 → Regex, Triggers, OO

一个系统需要说它支持 SQL 的最小语言语法是 SQL-92。

## 2. Relational Languages

该语言由不同类别的命令组成：

* Data Manipulation Language: `SELECT, INSERT, UPDATE, DELETE`
* Data Definition Language： Schema definitions for tables, indexes, views, and other objects.
* Data Control Language： Security, access controls.

SQL是不断发展的，关系代数基于 sets (unordered, no duplicates)。 SQL基于 bags (unordered, allows duplicates)。

一个数据库例子：

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p>database example</p></figcaption></figure>

```sql

CREATE TABLE student (
    sid INT PRIMARY KEY,
    name VARCHAR(16),
    login VARCHAR(32) UNIQUE,
    age SMALLINT,
    gpa FLOAT
);
CREATE TABLE course (
    cid VARCHAR(32) PRIMARY KEY,
    name VARCHAR(32) NOT NULL
);
CREATE TABLE enrolled (
    sid INT REFERENCES student (sid),
    cid VARCHAR(32) REFERENCES course (cid),
    grade CHAR(1)
);
```

## 3. Aggregates <a href="#tid-tiwyap" id="tid-tiwyap"></a>

聚合函数接受一组列表，然后产生一个单一的标量值作为其输出。基本上只能在SELECT输出列表中使用。

* AVG(col)→ Return the average col value.
* MIN(col)→ Return minimum col value.&#x20;
* MAX(col)→ Return maximum col value.
* SUM(col)→ Return sum of values in col.&#x20;
* COUNT(col)→ Return # of values for col.

1. 得到`@cs`登录的学生的人数。

```sql
SELECT COUNT(*) FROM student WHERE login LIKE '%@cs';
-- *换成其他都行
```

2. &#x20;得到`@cs`登录的学生的人数和平均GPA

```sql
SELECT AVG(gpa), COUNT(sid) FROM student WHERE login LIKE '%@cs';
```

{% hint style="info" %}
有些聚合函数支持`DISTINCT`关键字。
{% endhint %}

3. 得到通过`@cs`登录的学生数量，以及他们的GPA, 要求学生不能重复

```sql
SELECT COUNT(DISTINCT login) FROM student WHERE login LIKE '%@cs';
```

## 4. Group By

得到在每个课上的学生的平均GPA，出现聚合函数和其它字段时，后面必须接 GROUP BY 字段。

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid;
```

下面是错误的

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
```

## 5. Having

HAVING 子句在聚合计算的基础上过滤输出结果。这使得 HAVING 的行为像一个 GROUP BY 的 WHERE 子句。&#x20;

获取学生平均GPA大于3.9的课程。

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING avg_gpa > 3.9;
```

上述查询语法被许多主要的数据库系统所支持，但不符合 SQL 标准。为了使查询符合标准，我们必须在AVG(S.GPA)的主体中使用 HAVING 子句.

```sql
SELECT AVG(s.gpa), e.cid
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING AVG(s.gpa) > 3.9;
```

## 6. String Operations <a href="#tid-3zxr2f" id="tid-3zxr2f"></a>

SQL-92 标准是区分大小写的，而且只能是单引号。函数可以处理字符串，可以在查询的任何部分使用。

* **Pattern Matching:** `LIKE`关键字
  * %：匹配全部
  * \_：匹配单个
* **String Function：**
  * SUBSTRING(S, B, E)
  * UPPER(S)
* **Concatenation:**
  * ||：

## 7. Date/Time Operations <a href="#tid-dtwht2" id="tid-dtwht2"></a>

可以在输出和谓词中使用。 支持/语法有很大的不同...

## 8. Output Redirections

你可以告诉 DBMS 将查询结果存储到另一个表中，而不是将查询结果返回给客户端（例如，终端）。结果存储到另一个表中。然后你可以在随后的查询中访问这些数据.

1. 表不能已经被定义，表将具有与输入相同的列数和类型。

```sql
SELECT DISTINCT cid INTO CourseIds FROM enrolled;
```

2. 从查询中插入元组到另一个表中。内部 SELECT 必须产生与目标表相同的列。 DBMS有不同的选项/语法来处理完整性违反（例如，无效的重复）。

```sql
INSERT INTO CourseIds (SELECT DISTINCT cid FROM enrolled)
```

## 9. Output Controls

因为 SQL 是无序的，我们可以用 ORDER BY 来对输出进行排序。

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721' ORDER BY grade;
```

后面可以加`DESC（降序）`, `ASC（生序）`来指定排序策略

输出的数量可以用`LIMIT n` 进行指定，也可以用`OFFSET` 来提供一个bias。

## 10. Nested Queries

在其他查询中调用查询，在单个查询中执行更复杂的逻辑。嵌套查询往往难以优化。内部查询可以出现在（几乎）所有的查询。

获取在15-445中注册的学生名字

```sql
SELECT name FROM student
    WHERE sid IN (
        SELECT sid FROM enrolled
        WHERE cid = '15-445'
);
```

* ALL→ Must satisfy expression for all rows in the sub-query.&#x20;
* ANY→ Must satisfy expression for at least one row in the sub-query.&#x20;
* IN→ Equivalent to '=ANY()' .&#x20;
* EXISTS→ At least one row is returned without comparing it to an attribute in outer query.

上面的另一种写法，把 IN 换成 =ANY

```sql
SELECT name FROM student
    WHERE sid = ANY(
        SELECT sid FROM enrolled
        WHERE cid = '15-445'
)
```

## 11. Window Functions

A window function perform “sliding” calculation across a set of tuples that are related. Like an aggregation but tuples are not grouped into a single output tuple.

函数： 窗口函数可以是我们上面讨论的任何一个聚合函数。也有一些特殊的窗口函数。

1. `ROW_NUMBER`: 当前列的数字
2. `RANK`: 当前列的顺序

```sql
SELECT *, ROW_NUMBER() OVER () AS row_num FROM enrolled
```

**OVER 子句指定了在计算窗口函数时如何对图元进行分组**。使用 PARTITION BY来指定分组

```sql
SELECT cid, sid, ROW_NUMBER() OVER (PARTITION BY cid)
FROM enrolled ORDER BY cid;
```

我们也可以在OVER中放入ORDER BY，以确保结果的确定性排序，即使数据库内部发生变化。

```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY cid)
	FROM enrolled ORDER BY cid;
```

{% hint style="info" %}
DBMS在窗函数排序后计算`RANK()`，而在排序前计算`ROW_NUMBER()`
{% endhint %}

找到每门课程中成绩第二高的学生

```sql
SELECT * FROM (
    SELECT *, RANK() OVER (
        PARTITION BY cid
        ORDER BY grade ASC) AS rank
    FROM enrolled) AS ranking
WHERE ranking.rank = 2;
```

## 12. Commom Table Expressions

在编写更复杂的查询时，通用表表达式（CTE）是窗口或嵌套查询的一种替代方法。复杂的查询时，可以替代窗口或嵌套查询。它们提供了一种方法来为用户在一个更大的查询中编写辅助语句.

可以理解为一个辅助表。

`WITH`子句将内部查询的输出与一个具有该名称的临时结果绑定。

1. 生成一个名为 cteName 的 CTE，其中包含一个单一属性设置为 "1 "的元组。从这个 CTE 中选择所有属性。

```sql
WITH cteName AS (
	SELECT 1
)
SELECT * FROM cteName;
```

2. 我们可以在AS之前将输出列绑定到名称上

```sql
WITH cteName (col1, col2) AS (
	SELECT 1, 2
)
SELECT col1 + col2 FROM cteName;
```

3. 一个查询可能包含多个CTE声明

```sql
WITH cte1 (col1) AS (SELECT 1), cte2 (col2) AS (SELECT 2)
SELECT * FROM cte1, cte2;
```

4. 查找至少注册一门课程的学生，并且具有学生id最大的学生记录。

```sql
WITH cteSource (maxId) AS (
    SELECT MAX(sid) FROM enrolled
)
SELECT name FROM student, cteSource
WHERE student.sid = cteSource.maxId
```

5. 递归能力。在 WITH 后面添加 RECURSIVE 关键字允许 CTE 引用自己。这使得在 SQL 查询中可以实现递归。有了递归的 CTE，SQL 被证明是图灵完备的，这意味着它在计算上的表现力不亚于更多的通用编程语言。

```sql
WITH RECURSIVE cteSource (counter) AS (
    ( SELECT 1 )
    UNION
    ( SELECT counter + 1 FROM cteSource
    	WHERE counter < 10 )
)
SELECT * FROM cteSource;
```

## 13. Conclusion

SQL不是一种死的语言。&#x20;

你应该（几乎）总是努力将你的答案作为一个单一的 SQL 语句来计算，剩下的交给 DBMS。
