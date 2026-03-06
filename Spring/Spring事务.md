*==需要格外注意的是：**事务能否生效数据库引擎是否支持事务是关键。比如常用的 MySQL 数据库默认使用支持事务的 `innodb`引擎。但是，如果把数据库引擎变为 `myisam`，那么程序也就不再支持事务了！**==*
# 什么是事务
- **事务是逻辑上的一组操作，要么都执行，要么都不执行。**
```
- public void savePerson() {
    personDao.save(person);
    personDetailDao.save(personDetail);
  }
```
- 就像这个方法里面包含了多个原子性的数据库操作,它们要么都执行，要不就都不执行。
# 特性ACID
- **原子性**（`Atomicity`）：事务是最小的执行单位,事务的原子性保证了同一个事务里面的所有动作要不全部完成,要不全部失败
- **一致性**（`Consistency`）：执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；
- **隔离性**（`Isolation`）：并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
- **持久性**（`Durability`）：一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。
- **==只有保证了事务的持久性、原子性、隔离性之后，一致性才能得到保障。也就是说 A、I、D 是手段，C 是目的！==**

# Spring两种事务管理
## 编程式事务管理(不常用)
- 通过 `TransactionTemplate`或者`TransactionManager`手动管理事务

## 声明式事务管理(推荐)
- 基于 AOP 实现，使用 `@Transactional` 注解进行便捷的事务控制。 

# Spring事务接口
 - Spring 提供以下标准接口抽象事务操作：
	- `PlatformTransactionManager`(事务管理器接口)：管理事务的实现层。
	- `TransactionDefinition`(事务属性)：定义事务规则，如隔离级别、传播行为等。
	- `TransactionStatus`(事务状态)：记录事务运行状态
	
	## **事务属性**
	 - ==**事务传播行为**:定义方法在事务嵌套调用时如何运行(三种常用的)==
		 - `PROPAGATION_REQUIRED`：默认，加入当前事务或创建新事务。
		- `PROPAGATION_REQUIRES_NEW`：挂起当前事务，开启独立事务。
		- `PROPAGATION_NESTED`：父事务内部子事务，以嵌套方式执行。
	

	- ==**事务隔离级别**:控制不同事务间的可见性==
		- 不考虑隔离性会出现是==**三个读问题**==
			- 脏读
			- 不可重复读
			- 虚读（幻读）
		脏读（问题）：⼀个未提交事务读取到另⼀个未提交事务的数据
		不可重复读（现象）：⼀个未提交事务读取到另⼀个提交事务修改数据
		虚读（幻读）：⼀个未提交事务读取到另⼀个提交事务添加数据
		- **`TransactionDefinition.ISOLATION_DEFAULT`** :使用后端数据库默认的隔离级别，MySQL 默认采用的 `REPEATABLE_READ` 隔离级别 Oracle 默认采用的 `READ_COMMITTED` 隔离级别.
		
		-  从较低的 `READ_UNCOMMITTED` 到最高级的 `SERIALIZABLE`。
		- **`TransactionDefinition.ISOLATION_READ_UNCOMMITTED`**:允许读取尚未提交的数据变更,**可能会导致脏读、幻读或不可重复读**
		- **`TransactionDefinition.ISOLATION_READ_COMMITTED`**:允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
		- **`TransactionDefinition.ISOLATION_REPEATABLE_READ`**:对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
		- **`TransactionDefinition.ISOLATION_SERIALIZABLE`**:什么也不会发生,但是严重影响性能
		- ![[Pasted image 20251201170222.png]]
	
	- ==**事务超时属性 (Timeout)**：设置事务最大运行时间,超时则回滚,默认值为-1，即不超时==
	
	- ==事务只读属性:对于只有读取数据查询的事务，可以指定事务类型为 readonly，即只读事务。==
		- readOnly：是否只读
		- readOnly，默认FALSE，表示可以查询，修改，添加删除操作。
	
	- **==事务回滚规则:默认情况下，事务只有遇到运行期异常（`RuntimeException` 的子类）时才会回滚，`Error` 也会导致事务回滚，但是，在遇到检查型（Checked）异常时不会回滚。==**

# @Transactional注解使用
## @Transactional的作用范围

-  **方法**：推荐将注解使用于方法上，不过需要注意的是：**该注解只能应用到 public 方法上，否则不生效。**
-  **类**：如果这个注解使用在类上的话，表明该注解对该类中所有的 public 方法都生效。
-  **接口**：不推荐在接口上使用。
## 常用参数
![[Pasted image 20251201181017.png]]

## @Transactional事务注解原理
面试中在问 AOP 的时候可能会被问到的一个问题。简单说下吧！

我们知道，**`@Transactional` 的工作机制是基于 AOP 实现的，AOP 又是使用动态代理实现的。如果目标对象实现了接口，默认情况下会采用 [[JDK动态代理]]，如果目标对象没有实现了接口,会使用 CGLIB 动态代理。**

## 使用 @Transactional 的注意事项
- `@Transactional` 注解只有作用到 public 方法上事务才生效，不推荐在接口上使用；
- 避免同一个类中调用 `@Transactional` 注解的方法，这样会导致事务失效；
- 正确的设置 `@Transactional` 的 `rollbackFor` 和 `propagation` 属性，否则事务可能会回滚失败;
- 被 `@Transactional` 注解的方法所在的类必须被 Spring 管理，否则不生效；
- 底层使用的数据库必须支持事务机制，否则不生效；