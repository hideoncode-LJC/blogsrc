---
title: 子查询
date: 2023-11-08 20:33:41
tags: 数据库
categories: MySQL
---

# 子查询

## 为什么需要子查询

比如有一张salary table，你想要查询比员工A工资高的员工都有谁？此时该如何查询。利用前面的知识，可以使用以下两种方法。

```SQL
# 方法一
1. 先计算员工A的工资
2. 然后在查询比第一步得到的结果高的员工
SELECT salary
FROM salary
WHERE employ_name = 'A';

SELECT employ_name
FROM salary
WHERE salary > 10000;

# 方法二 子连接
1. 通过笛卡尔积将两张表连接在一块
2. 
```

以上两种方法在效率或者复杂程度上都有不足。所以就出现了子查询。

子查询是指一个查询语句嵌套在另外一个查询语句中。

```SQL
SELECT employ_name 
FROM salary
WHERE salary > (
    SELECT salary
    FROM salary
    WHERE employ_name = 'A';
);
```

## 子查询的基本使用

> 子查询的使用注意

1. 子查询要包含在括号中
2. 将子查询放在比较条件的右侧
3. 单行操作符应用于单行子查询，多行操作符应用于多行子查询。

## 子查询的分类

+ 根据子查询返回的结果集的记录个数，将子查询分为单行子查询和多行子查询
+ 根据子查询是否需要和外部查询交互，分为相关子查询和不相关子查询。

## 单行子查询

### 单行操作符

+ `<>` not equal to

### 常见题目

**查询工资大于149号员工工资的员工信息**

```SQL
SELECT * 
FROM employees
WHERE salary > (
    SELECT salary
    FROM employees
    WHERE employee_id = 149
);
```

**返回job_id与141号员工相同，salary比143号员工多的员工姓名，job_id和工资**

```SQL
SELECT first_name, last_name, job_id, salary
FROM employees
WHERE job_id = (
    SELECT job_id 
    FROM employees
    WHERE employee_id = 141
) and salary > (
    SELECT salary 
    FROM employees
    WHERE employee_id = 143
);
```

**返回公司工资最少的员工的last_name,job_id和salary**

```SQL
SELECT last_name, job_id, salary
FROM employees
WHERE salary = (
    SELECT MIN(salary)
    FROM employees
);
```

**查询与141号或174号员工的manager_id和department_id相同的其他员工的employee_id，manager_id，department_id**

```SQL
SELECT employee_id, manager_id, department_id
FROM employees
WHERE manager_id = (
    SELECT manager_id
    FROM employees
    WHERE employee_id = 141
) and department_id = (
    SELECT department_id
    FROM employees
    WHERE employee_id = 141
) or manager_id = (
    SELECT manager_id
    FROM employees
    WHERE employee_id = 174
) and department_id = (
    SELECT department_id
    FROM employees
    WHERE employee_id = 174
);

# 不成对比较方式,分别比较manager_id,department_id
SELECT employee_id, manager_id, department_id
FROM employees
WHERE manager_id IN (
    SELECT manager_id
    FROM employees
    WHERE employee_id IN (141, 174)
) or department_id IN (
    SELECT department_id
    FROM employees
    WHERE employee_id IN (141, 174)
);


# 成对比较方式 manager_id department_id 同时进行比较
SELECT employee_id, manager_id, department_id
FROM employees
WHERE (manager_id, department_id) IN (
    SELECT manager_id, department_id 
    FROM employees
    WHERE employee_id IN (141, 174)
) AND employee_id NOT IN (141, 174);
```

**查询最低工资大于50号部门最低工资的部门id和其最低工资**

```SQL
SELECT department_id, MIN(salary)
FROM employees
GROUP BY department_id
HAVING MIN(salary) > (
    SELECT MIN(salary)
    FROM employees
    WHERE department_id = 50
);
```

### case中的子查询

### 子查询中的空值问题

```SQL
SELECT last_name, job_id
FROM employees
WHERE job_id = (
    SELECT job_id
    FROM employees
    WHERE last_name = 'Haas'
);
```

此时无查询结果，子查询不返回任何行。

### 非法使用子查询

```SQL
SELECT employee_id, last_name
FROM employees
WHERE salary = (
    SELECT MIN(salary)
    FROM employees
    GROUP BY department_id
);
```
多行子查询使用单行比较符

## 多行子查询

+ 多行子查询也叫做集合比较子查询
+ 内查询返回多条记录
+ 使用多行比较操作符

### 多行比较操作符

+ `IN`,等于列表中的任意一个
+ `ANY`,需要和单行比较操作符一起使用,和子查询返回的某一个值比较
+ `ALL`,需要和单行比较操作符一起使用,和子查询返回的所有值进行比较
+ `SOME`, ANY的别名,作用相同,一般常使用ANY

### 代码示例

**返回其它job_id中比job_id为‘IT_PROG’部门任一工资低的员工的员工号、姓名、job_id 以及salary**

```SQL
SELECT employee_id, first_name, last_name, job_id, salary 
FROM employees
WHERE salary < ANY (
    SELECT salary
    FROM employees
    WHERE job_id = 'IT_PROG'
) AND job_id <> 'IT_PROG';
```

**返回其它job_id中比job_id为‘IT_PROG’部门所有工资都低的员工的员工号、姓名、job_id以及salary**

```SQL
SELECT employee_id, first_name, last_name, job_id, salary 
FROM employees
WHERE salary < ALL (
    SELECT salary
    FROM employees
    WHERE job_id = 'IT_PROG'
) AND job_id <> 'IT_PROG';
```

**查询平均工资最低的部门id**

```SQL
SELECT department_id, AVG(salary)
FROM employees
GROUP BY department_id
HAVING AVG(salary) <= ALL(
    SELECT AVG(salary)
    FROM employees
    GROUP BY department_id
);

# 其他方式
```

### 空值问题

```SQL
SELECT last_name
FROM employees
WHERE employee_id NOT IN(
    SELECT manager_id
    FROM employees
);
```

## 相关子查询

如果子查询的执行依赖于外部查询，通常情况下是因为子查询的表用到了外部的表，并进行了条件关联，因此每执行一次外部查询，子查询都要重新计算一次，这样的查询就称为
`子查询`。相关子查询按照一行接一行的顺序执行，主查询的每一次都执行一次子查询。

> 子查询中使用主查询中的列

### 代码示例

**查询员工中工资大于本部门平均工资的员工的last_name,salary和其department_id**

```SQL
SELECT last_name, salary, department_id
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e1.department_id
);
```

### 相关更新

**怎么使用**

通过相关子查询使用一个表的数据更新另外一个表

```SQL
UPDATE table1 alias1
SET column
```



## 思考题

> 自连接与子查询的比较

```SQL
# 自连接
SELECT e2.last_name, e2.salary
FROM employees e1, employees e2
WHERE e2.last_name = 'Abel' AND e1.salary > e2.salary

# 子查询
SELECT last_name, salary
FROM employees
WHERE salary > (
    SELECT salary 
    FROM employees
    WHERE last_name = 'Abel'
)
```

**自连接和子查询谁更快**

自连接方式好！
题目中可以使用子查询，也可以使用自连接。一般情况建议你使用自连接，因为在许多 DBMS 的处理过
程中，对于自连接的处理速度要比子查询快得多。
可以这样理解：子查询实际上是通过未知表进行查询后的条件判断，而自连接是通过已知的自身数据表
进行条件判断，因此在大部分 DBMS 中都对自连接处理进行了优化。

？？ 怎么进行优化的？

