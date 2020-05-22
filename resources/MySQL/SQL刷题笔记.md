# SQL刷题笔记

### 1、第二高薪水

<img src="E:\Typora\MyNote\resources\MySQL\SQL刷题\第二高薪水.png" style="zoom:60%;" />

注意事项：

1. 注意有可能重复，所以要用`distinct`

```sql
select (select distinct(Salary) from Employee order by Salary desc limit 1,1) as SecondHighestSalary;
```

### 2、分数排名

<img src="E:\Typora\MyNote\resources\MySQL\SQL刷题\分数排名.png" style="zoom:60%;" />

注意事项:

1. `count(DISTINCT score)`和`count(1) groud by`的用法区别
2. 注意排名是从高到低,所以是`score >= s.score`

```sql
SELECT Score, (SELECT count(DISTINCT score) FROM Scores WHERE score >= s.score) AS Rank FROM Scores s ORDER BY Score DESC ;
```

### 3、连续出现的数字

<img src="E:\Typora\MyNote\resources\MySQL\SQL刷题\连续出现的数字.png" style="zoom:60%;" />

没什么好说的,当扩宽思路吧。

```xml
select c.Num as ConsecutiveNums from Logs a,Logs b,Logs c
where a.id+1=b.id and a.Num=b.Num and b.id+1=c.id and b.Num=c.Num group by c.Num;
```

