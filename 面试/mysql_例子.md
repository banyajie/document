###### MYSQL JOIN



left Join  right Join  inner Join

```mysql
# 表一：Person
+-------------+---------+
| 列名         | 类型     |
+-------------+---------+
| PersonId    | int     |
| FirstName   | varchar |
| LastName    | varchar |
+-------------+---------+
# PersonId 是上表主键

# 表二：Address
+-------------+---------+
| 列名         | 类型    |
+-------------+---------+
| AddressId   | int     |
| PersonId    | int     |
| City        | varchar |
| State       | varchar |
+-------------+---------+
#AddressId 是上表主键


#编写一个 SQL 查询，满足条件：无论 person 是否有地址信息，都需要基于上述两表提供 person 的以下信息：
# FirstName, LastName, City, State


select p.FirstName, p.LastName, a.City, a.State from Person p left join Address a on a.PersonId = p.PsersonId;

left join（左联结）:返回左表中的所有记录，和右表中联结字段相等的记录
right join(右联结):同理
inner join(等值联结):只返回两表中联结字段相等的行

```



###### 第二高的薪水

```shell
# Employee
+----+--------+
| Id | Salary |
+----+--------+
| 1  | 100    |
| 2  | 200    |
| 3  | 300    |
+----+--------+

SELECT (SELECT DISTINCT Salary FROM Employee ORDER BY Salary DESC LIMIT 1 OFFSET 1) AS SecondHighestSalary;

SELECT IFNULL((SELECT DISTINCT Salary FROM Employee ORDER BY Salary DESC LIMIT 1 OFFSET 1), NULL) AS SecondHighestSalary;




```

