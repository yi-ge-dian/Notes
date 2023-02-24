# hw01-SQL

## 0. 准备工作

跟着官网的步骤一步一步做即可。

{% embed url="https://15445.courses.cs.cmu.edu/fall2022/homework1/" %}
CMU官网地址
{% endembed %}

但是如果使用DataGrip 来连接 Windows 下的 WSL2 启动时，会报错误。

{% hint style="danger" %}
SQL错误 \[5]: \[SQLITE\_BUSY] The database file is locked (database is locked) \[SQLITE\_BUSY] The database file is locked (database is locked) \[SQLITE\_BUSY] The database file is locked (database is locked)
{% endhint %}

解决方法就是把数据库文件放到 Windows 下面来，不要放在 WSL2 的路径。

SQLite 教程地址

{% embed url="https://www.sqlitetutorial.net/" %}
SQLite官网地址
{% endembed %}

## 1. 记录步骤

### Q1 \[0 points] (q1\_sample):

这个主要就是用来热身的，代码已经给我们了，要求我们按**字母表顺序列出所有类别名称。**

```sql
SELECT DISTINCT(language)
FROM akas
ORDER BY language
LIMIT 10;
```

### Q2 \[5 points] (q2\_sci\_fi):

输出作品的名字、首映日、时长。时长应以 `(mins)` 为后缀，例如时长为 12 分钟，输出为 `12(mins)`。如果作品的流派有多个，只要包含科幻即属于科幻作品。

结果的第一行应为：`Cicak-Man 2: Planet Hitam|2008|999 (mins)`

```sql
SELECT primary_title,premiered, runtime_minutes || ' (mins)'
FROM titles
WHERE genres like '%Sci-Fi%'
ORDER BY runtime_minutes DESC
LIMIT 10;
```

* CAST 可以把 integer 转为 string
* mins 前面有个空格

### Q3 \[5 points] (q3\_oldest\_people):

找到出生于 1990 及以后最年长的人，如果死亡年份为空说明还在世。

输出每个人的姓名和年龄，输出结果首先按年龄降序，其次按姓名升序，返回前 20 个结果。

```sql
SELECT name,age
FROM (
    SELECT name, (2022 - born) AS age
    FROM people
    WHERE born >=1900 AND died IS NULL
    UNION
    SELECT name, (died - born) AS age
    FROM people
    WHERE born >=1900 AND died IS NOT NULL
    )
ORDER BY age DESC,name
LIMIT 20;
```

### Q4 \[10 points] (q4\_crew\_appears\_most):

找到出现次数最多的 20 名剧组人员，输出姓名和出场次数，按照出场次数降序。

```sql
WITH temp(count , person_id) AS (
    SELECT count(*) AS count, person_id
    FROM crew
    GROUP BY  person_id
    ORDER BY  count DESC
    LIMIT 20
)

SELECT name, count
FROM temp JOIN people
ON temp.person_id = people.person_id
```

### Q5 \[10 points] (q5\_decade\_ratings):

输出每十年的作品发行量、平均分(四舍五入到两位小数)、最高分、最低分，忽略尚未上映的作品，即 `premiered` 为 `null` 的作品。

打印十年的时候以一种更优雅的方式：如 1990s，输出的时候按照平均分降序，然后再根据年份升序。

```sql
WITH temp(decade,title_id) AS (
    SELECT (premiered-premiered%10) AS decade ,title_id
    FROM titles
    WHERE premiered IS  NOT NULL
)
SELECT decade || 's',round(avg(rating),2) AS avg,max(rating),min(rating),count(1)
FROM temp
JOIN ratings ON temp.title_id = ratings.title_id
GROUP BY decade
ORDER BY round(avg(rating),2) DESC, decade;
```

### Q6 \[10 points] (q6\_cruiseing\_altitude):

找到名字包含 "Cruise" 且出生于 1962 年的人参与的票数最高的作品，输出作品名和票数。

列出 10 个按照票数从高到底的顺序。



