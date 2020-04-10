title: 事务性
author: Dylan
tags:
  - 事务
categories:
  - 并发
date: 2018-10-07 10:34:00
---
### ACID
&emsp;&emsp;事务有以下几个属性:
1. 原子性(Atomicity)

&emsp;&emsp;在一个事务中，所有的操作都是原子的，要么所有的操作都生效，要么所有的操作都失效，不存在部分操作生效，部分操作失效。

2. 一致性(Consistency)

&emsp;&emsp;在事务开始和结束时，系统的资源处于一至的，没有被破坏的状态。

3. 隔离性(Isolation)

&emsp;&emsp;一个事务，在它被提交之后，它的所有操作结果才会被其他事务可见。比如在一个事务中，对数据库做了多次修改操作，但在事务提交之前，这些修改对其他事务来说是不可见的。

4. 持久性(Durability)

&emsp;&emsp;一个已提交的事务的任何结果必须是永久性的，哪怕系统崩溃了也不能影响已提交事务的持久化。


### 事务级别
&emsp;&emsp;事务有4个级别，分别是"可串行化"，"可重复读(幻读)"，"不可重复读(读已提交)"和"脏读(读未提交)"

1. 可串行化

&emsp;&emsp;事务串行化能保证，第一个事务如果在第二个事务之前打开，那么第二个事务做的任何修改对第一个事务都没有影响。第一个事务在第二事务提交后打开，那么第一个事务读取到的是第二个事务修改后的最新数据。

&emsp;&emsp;比如有两条记录A，B，A的值为1,B的值为2。有两个事务，一个事务要读取A，B两条记录，另一个事务要修改A为2，B为3。第一个事务先打开，读取A值为1,然后第二个事务打开，并修改了A为2,B为3并提交了事务。这时，第一个事务再读取B值，读取到的是2而不是3。如果第一个事务在第二个事务提交后再打开，那么读取到A的值为2,B的值为3.

2. 可重复读

&emsp;&emsp;还是拿上面的例子做说明。首先第一个事务被打开，读取了A的值为1,这时第二个事务打开，修改了A的值为2,B的值为3并提交了事务，这是第一个事务再读取B的值为3而不是2.

3. 不可重复读

&emsp;&emsp; 第一个事务打开，读取了A的值为1,第二个事务打开，修改了A的值并提交事务。当第一个事务在事务中再次读取A值时，每一次读取都可能跟上一次不一样。

4. 脏读

&emsp;&emsp;如果第二个事务先打开，对A的值修改成了2,但第二个事务还没有提交。在脏读这个事务级别上，哪怕事务还没有提交，事务中所做的修改对其他事务是可见的。这时第一个事务打开，读取到A的值是2。当第二个事务想修改B的值为3时发生了异常，第二个事务回滚，A的值为1。但对于第一个事务来说，读取到A的值为2，这就出现了第一个事务对数据脏读。