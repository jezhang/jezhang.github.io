---
layout:     post
title:      "PostgresSQL SQL调优技巧"
date:       2017-06-06 21:44:00
author:     "jezhang"
header-img: "https://asdfex.oss-cn-qingdao.aliyuncs.com/jezhang.github.io/img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - 数据库
    - 学习笔记
---


> PostgresSQL SQL调优技巧


### SQL调优范式

#### 开发范式一：不要轻易把字段嵌入到表达式

在salary列上有index，但是条件语句中把salary列放在了表达式当中，导致索引被压抑，因为索引里面存储的是salary的值，而不是salary加上100以后的值。
```sql
explain select * from emp where salary + 100 = 2000; 
--------------------------------------------------------------------
Gather (cost=1000.00..7796.60 rows=2294 width=36)
  Workers Planned: 2
  -> Parallel Seq Scan on emp (cost=0.00..6567.20 rows=956 width=36)
       Filter: ((salary + 100) = 2000)
(4 rows)
```
通过等式转换，把salary列从表达式中剥离出来，就会用到索引。
```sql
explain select * from emp where salary = 2000 - 100;
--------------------------------------------------------------------
Index Scan using emp_salary_index on emp (cost=0.42..8.44 rows=1 width=36)
  Index Cond: (salary = 1900)
(2 rows)
```

#### 开发范式二：不要轻易把字段嵌入到函数中
在hiredate列上有index，但是条件语句中把该列放在了函数当中，导致索引被压抑，因为索引里面存储的是该列的值，而不是函数处理以后的值。
```sql
explain select * from emp where to_char(hiredate, ‘dd-mm-yyyy’)=’22-05-2022’;
--------------------------------------------------------------------
Seq Scan on emp (cost=0.00..289.32 rows=50 width=62)
  Filter: (to_char((hiredate)::timestamp with time zone, 'dd-mm-yyyy'::text) = '22-05-2022'::text)
```
改写成（通过等式转换，把列从函数中剥离出来，就会用到索引，比较成本，差别较大）
```sql
explain select * from emp where hiredate = to_date(’22-05-2022’, ‘dd-mm-yyyy’);
--------------------------------------------------------------------
Index Scan using emp_hiredate on emp (cost=0.29..8.30 rows=1 width=62)
  Index Cond: (hiredate = to_date('22-05-2022'::text, 'dd-mm-yyyy'::text))
```

#### 开发范式三

如果查询中比较固定查询某些列，可以基于这几个列建符合索引，直接查询索引，避免**回表扫描**。

```sql
create index emp_empno on emp(empno, salary);
explain select empno, salary from emp where empno=7788;
--------------------------------------------------------------------
Index Only Scan using emp_empno on emp (cost=0.29..10.09 rows=2 width=8)
  Index Cond: (empno=7788)
```

### 多表查询调优技巧

OLTP应用SQL调优指导方针

 - 驱动表上有很好的条件限制，同时，驱动表上的限制性条件字段上应该有索引，包括主键、唯一索引或其它索引、复合索引等。
 - 在每次连接操作之后尽量保证返回记录数最少，传递给下一个连接操作根据返回的行的数量对应正确的连接方式。
 - 尽量通过在被驱动表的连接字段上的索引，访问被驱动表。
 - 单表扫描应该有效率，如果被驱动表上还有其它限制条件，可以遵循复合索引创建原则，创建合适的复合索引（连接字段与条件字段）。
 - 全表扫描也许是合理的，例如若干小表、代码表的访问。
 - 依次类推，顺序完成所有表的连接操作。

#### 多表查询指导方针

##### 多表连接优化案例一

```sql
explain select e.*, d.*
        from emp e, dept d
        where d.deptno = e.deptno
        and e.empno=7499;
--------------------------------------------------------------------
Nested Loop(cost=0.30..16.36 rows=1 width=192)
 -> Index Scan using pk_emp on emp e (cost=0.15..8.17 rows=1 width=98)
    Index Cond:(empno = 7499)
 -> Index Scan using pk_dept on dept d (cost=0.15..8.17 rows=1 width=94)
    Index Cond: (deptno = e.deptno)
```

**执行计划解读**

 1. 先按照建立在empno字段上的索引去emp表查询empno为7499的员工信息。
 2. 再根据7499所在的部门号（deptno）去dept表查询该部门的详细信息，而且dept表的deptno字段上应该有索引。
 3. 最后使用嵌套循环连接方式处理数据。

>建议：“如果是多表连接sql语句，注意驱动表的连接字段是否需要创建索引”。
在上例中，被驱动表是dept，dept表的连接字段是deptno，而emp的deptno字段是可以不需要建索引的，因为已经根据条件字段上列访问驱动表。 


##### 多表连接优化案例二

```sql
explain select e.*, d.*
        from emp e, dept d
        where d.deptno = e.deptno
        and e.empno=7499
        and d.dname='DALLAS';
--------------------------------------------------------------------
Nested Loop (cost=0.30..20.35 rows=1 width=192)
 -> Index Scan using pk_emp on emp e (cost=0.15..8.17 rows=1 width=98)
    Index Cond:(empno = 7499)
 -> Index Scan using pk_dept on dept d (cost=0.15..8.17 rows=1 width=94)
    Index Cond: (deptno = e.deptno)
    Filter: ((dname)::text = 'DALLAS'::text)
```

**执行计划解读**
 1. 先按照建立在empno字段上的索引去emp表查询empno为7499的员工信息。
 2. 再根据7499所在的部门号（deptno）去dept表查询该部门的详细信息。此时dept表还有一个条件字段loc='DALLAS'，因此可考虑按（deptno，loc）复合索引方式去查询dept表，效率更高，即可建立（deptno，loc）字段上的复合索引（idx_dept_2）。
 3. 最后以嵌套循环的连接方式处理数据。

> 建议：“如果是多表连接sql语句，注意是否可以在被驱动表的连接字段与该表的其它约束条件字段上创建复合索引”。索引可以在dept表上创建（deptno与dname）字段的复合索引。

**应该遵循关于复合索引创建时的建议**

“如果单个字段是主键或者唯一字段，或者可选性非常高的字段，尽管约束条件字段比较固定，也不一定要建成复合索引，可建成单字段索引，降低复合索引开销”。

而且通过比较发现这种情况创建单列索引比创建复合索引查询的时候代价要低的多。
所以在本例中，不应该创建复合索引。


### 多表查询应用案例

#### 5张表查询应用案例

```sql
select emp.last_name, emp.first_name, j.job_title, d.department_name, l.city, l.state, l.state_province, l.postal_code, l.street_address, emp.email, emp.phone_number, emp.hire_date, emp.salary, mgr.last_name
from hr.employees emp, hr.employees mgr, hr.departments d, hr.locations l, hr.jobs j
where l.city='South San Fransisco'
and emp.manager_id=mgr.employee_id
and emp.department_id=d.department_id
and d.location_id=l.location_id
and emp.job_id=j.job_id
```
