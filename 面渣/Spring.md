- Spring使用了哪些设计模式: 常见的有工厂模式, 单例模式, 还有代理模式
	- 工厂模式: 这是Spring中一个使用非常多的模式, 像BeanFactory就是一个工厂, 它负责创建和管理所有的Bean示例.  通常通过@Autowired注解获取一个Bean时, 底层就是通过工厂来创建的
	- 单例模式: Spring的默认模式. 所有的Bean都是单例的, 它保证了在应用中只有一个实例, 节省内存. 也可以将@Scope注解设置为prototype, 这样就会在获取一次Bean时就创建一个Bean实例
	- 代理模式: 有两种代理模式, 分别为JDK动态代理和CGLIB代理, 当当前类实现接口时就会使用JDK动态代理, 否则就用CGLIB创建子类来进行代理
	- 还有模板方法: 比如 JdbcTemplate, 定义了数据库的基本流程: 连接数据库, 执行SQL, 处理结果, 关闭连接 
- Bean的生命周期: 总共分为5个阶段
	- 实例化: Spring容器会通过反射调用Bean的构造方法来创建对象实例, 这时候它没有属性
	- 属性注入: 对象创建好后, Spring会进行依赖注入(也就是根据Bean的属性赋值), 就是通过@Autowired, @Resource注解注入依赖对象, 或者将xml里面的配置填充进去
	- 初始化: 
		- 回调Aware接口: 如果 Bean 实现了 `BeanNameAware` 等接口，Spring 会把 Bean 的名字、Factory 等信息注入进去。
		- 进行BeanPostProcessor前置处理
		- 执行初始化方法: 
			- 首先执行@postConstruct下的方法
			- 在执行InitializingBean接口下的afterPropertiesSet方法
			- 最后执行自定的init-method
		- 进行BeanPostProcessor后置处理, 通常在这一步创建好了代理对象, 比如AOP代理
	- 使用Bean: 比如Controller调用Services, Services调用DAO
	- 销毁: 当容器关闭或者Bean被移除时, 依次执行
		- @preDestory下的方法
		- DisposaableBean接口的destory方法
		- 执行自定义的destory-method
- 三级缓存来解决循环依赖问题: 
	- 三级缓存: 
		- 一级缓存: 用来存储已经实例化初始化完全的Bean实例
		- 二级缓存: 用来存储已经暴露的Bean实例(进行了实例化, 没有初始化的)
		- 三级缓存: 存储Bean工厂, 用来生成提取暴露的Bean
	- 举个例子: 
		1. 创建Bean A, Spring调用A的构造方法创建了A实例. 将A的对象工厂放到三级缓存里面, 对A进行属性注入B
		2. A注入B: A要注入B, 然后发现B不存在, 此时开始创建B.  调用B的构造方法创建了B实例, 将B的对象工厂放到三级缓存里面, 对B进行属性注入A
		3. B注入A: B要注入A, 那么获取A, 先从一级缓存获取, 不存在, 二级缓存, 不存在, 三级缓存, 找到了A的对象工厂, 此时调用工厂的`getObject()`方法得到了A实例, 此时A进行了实例化, 将A从三级缓存升级到二级缓存
		4. B初始化完成: B属性注入完成, B完成了初始化, 将B放到一级缓存里面
		5. A重新zhu'ru