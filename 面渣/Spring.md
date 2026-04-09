- Spring使用了哪些设计模式: 常见的有工厂模式, 单例模式, 还有代理模式
	- 工厂模式: 这是Spring中一个使用非常多的模式, 像BeanFactory就是一个工厂, 它负责创建和管理所有的Bean示例.  通常通过@Autowired注解获取一个Bean时, 底层就是通过工厂来创建的
	- 单例模式: Spring的默认模式. 所有的Bean都是单例的, 它保证了在应用中只有一个实例, 节省内存. 也可以将@Scope注解设置为prototype, 这样就会在获取一次Bean时就创建一个Bean实例
	- 代理模式: 有两种代理模式, 分别为JDK动态代理和CGLIB代理, 当当前类实现接口时就会使用JDK动态代理, 否则就用CGLIB创建子类来进行代理
	- 还有模板方法: 比如 JdbcTemplate, 定义了数据库的基本流程: 连接数据库, 执行SQL, 处理结果, 关闭连接 

- ==Bean的生命周期==: 
	Spring 中 Bean 的生命周期，大致可以分为 **实例化、属性注入、初始化、使用、销毁** 这几个阶段。
	1. 实例化
		Spring 容器首先会通过反射调用构造方法创造出Bean对象, 此时Bean没有属性
	2. 属性注入 
		Spring 会对 Bean 进行依赖注入，把@Autowired、@Value这些标注的依赖注入进去
	3. 初始化
	- 回调 Aware 接口
	- 执行 BeanPostProcessor 前置处理
	- 执行初始化方法
	- 执行 BeanPostProcessor 后置处理
	
	- 其中：
		- 3.1 回调 Aware 接口
			- Aware 接口主要分为两类：
				- BeanFactory 级别, 在`invokeAwareMethods()`方法里直接调用，不经过任何后处理器：
			    
			    - `BeanNameAware`
			    - `BeanClassLoaderAware`
			    - `BeanFactoryAware`
		
			- ApplicationContext 级别, 通过`ApplicationContextAwareProcessor`这个后处理器调用。两者的调用时机和依赖层级不同。
			    
			    - `ApplicationContextAware`
			    - `EnvironmentAware`
			    - `ResourceLoaderAware`
			    - `ApplicationEventPublisherAware`
			    - `MessageSourceAware`
			
			- 它的作用是让 Bean 感知 Spring 容器提供的一些基础信息和能力。
			
		- 3.2 BeanPostProcessor 前置处理
			- 会执行 Bean 后置处理器中的前置方法，也就是：
			- `postProcessBeforeInitialization()`
			- @PostConstruct注解的处理就是在这一步完成的
			
		-  3.3 执行初始化方法
			- 依次执行`InitializingBean`接口的`afterPropertiesSet`方法和自定义的`init-method`
			
		-  3.4 BeanPostProcessor 后置处理
			- 所有BeanPostProcessor的`postProcessAfterInitialization()`执行
			- Spring AOP 的代理对象，通常就是在这个阶段创建出来的。上面方法的返回值会替换掉原始Bean，所以你注入的对象和Spring实例化的对象可能不是同一个。
			
	4. 使用 Bean
	- 初始化完成后，Bean 就可以被业务代码正常使用了，比如 Controller 调用 Service，Service 调用 DAO。
	
	 5. 销毁
		- 当容器关闭时，Spring 会销毁 Bean，执行顺序一般是：
			- `@PreDestroy`注解方法
			- `DisposableBean` 接口的 `destroy()`
			- 自定义的 `destroy-method`

- ==三级缓存来解决循环依赖问题==: 
	- 循环依赖就是两个Bean或者多个Bean互相依赖, 形成一个环, 比如A依赖B, B依赖A
	- 三级缓存: 
		- 一级缓存: 用来存储已经实例化初始化完全的Bean实例
		- 二级缓存: 用来存储已经暴露的Bean实例(进行了实例化, 没有初始化的)
		- 三级缓存: 存储Bean工厂, 用来生成提取暴露的Bean
	- 举个例子: 
		1. 创建Bean A, Spring调用A的构造方法创建了A实例. 将A的对象工厂放到三级缓存里面, 对A进行属性注入B
		2. A注入B: A要注入B, 然后发现B不存在, 此时开始创建B.  调用B的构造方法创建了B实例, 将B的对象工厂放到三级缓存里面, 对B进行属性注入A
		3. B注入A: B要注入A, 那么获取A, 先从一级缓存获取, 不存在, 二级缓存, 不存在, 三级缓存, 找到了A的对象工厂, 此时调用工厂的`getObject()`方法得到了A实例, 此时A进行了实例化, 将A从三级缓存升级到二级缓存
		4. B初始化完成: B属性注入完成, B完成了初始化, 将B放到一级缓存里面
		5. A重新注入B并完成初始化: 和B一样, 完成初始化, A从二级缓存删除, 放到一级缓存里面
		6. 最后A,B都完成了初始化且都在一级缓存里面
	
		- 为什么不是二级缓存而是三级缓存: 主要为了解决AOP代理的问题
			- 如果是二级缓存, 那么Bean进行实例化以后就要决定给二级缓存是存储原对象还是代理对象, 这违背了Spring的设计原则, 代理对象应该在BeanPostProcessor后置处理阶段创建
			- 如果三级缓存, 三季缓存里面的工厂去判断是生成一个原对象还是代理对象, 这样保证了不会出现两个Bean对象

		- 哪种循环依赖无法解决: 
			- 通过构造器进行注入的, 构造器注入发生在实例化阶段, 这时候Bean还没有创建出来，自然没办法放到缓存里提前暴露。
			- 非单例（prototype）作用域的 Bean. 对于 `scope="prototype"` 的 Bean，Spring 每次都会创建一个新的实例，并且不会将其放入缓存中。因此，循环依赖的机制完全失效。
		- Spring Boot从2.6版本开始默认禁止循环依赖

- ==关于AOP==:AOP（面向切面编程）把与业务逻辑无关的横切关注点（日志、事务、权限）封装成切面，在运行时通过动态代理织入到目标方法。它解决了业务代码和系统级代码纠缠在一起的问题，降低重复耦合。  
	- 应用场景: 生活派秒杀下单场景里，业务代码通过 `AopContext.currentProxy()` 取到代理对象，以保证事务方法在“自调用”场景下依然生效。

- ==Spring的JDK 动态代理和 CGLIB 代理的区别(Spring AOP是如何实现的(通过动态代理实现))==: 
	- JDK动态代理要求目标类必须实现接口, 通过反射机制创建一个实现了目标类的接口的匿名类, 调用方法时被转发到`InvocationHandler`的`invoke`方法里面, 在这里面织入内容, 同时它是JDK原生支持的
	- CGLIB代理是基于字节码的, 通过ASM字节码动态生成目标类的子类来创建代理对象, 子类可以重写父类的方法, 在子类方法里面织入内容, 因为它是继承, 所以无法代理final类
- ==选择策略==: 
	- 如果设置了proxyTargetClass=true，或者目标类没有实现任何接口，使用CGLIB
	- 如果目标类实现了接口, 就是用JDK动态代理
	- 如果目标本身就是接口或者已经是代理类了, 即使proxyTargetClass=true, 也是用JDK动态代理
	- spring boot2.x默认将spring.aop.proxy-target-class设为true, 所以默认使用CGLIB代理
- ==Spring AOP和AspectJ的区别==: Spring AOP是运行时通过动态代理实现的, 只能代理Spring管理的Bean, 只支持到方法级别, 而AspectJ 是编译期织入的, 支持到字段级别和构造方法级别, 但是需要额外的编译器

- ==说一下IOC与DI==: 
	- IOC就是控制反转, 传统的写法, 如果A类需要用B类, 那就在A类的代码里直接new一个B出来, 创建与管理对象的权力在自己手里, 而**IOC它把对象创建和依赖管理的权力交给了外部容器进**行, 需要B的时候, 容器帮你把B注入
	- DI就是实现IOC的具体手段, 有三种:
		- 构造器注入: 通过构造方法注入, 注入的字段可以声明为final, 保证创建出来的对象完整可用
		- setter注入: 适合可选依赖的场景
		- 字段注入: 加@Autowired, 但是它无法声明final字段, 不推荐
	- @Autowired和@Resource的区别: 
		- @Autowired是Spring提供的注解, 默认按照类型匹配. 如果同类型的Bean有多个, 那么就按照字段名匹配
		- @Resource是JDK提供的注解, 默认按名称匹配, 如果按名称找不到就按类型匹配
	- IOC成功实现了解耦, 让对象之间不再依赖具体实现, 降低了代码耦合度

- ==Spring事务==: Spring事务分为编程式事务和声明式事务, 编程式事务可以自己控制事务的开启, 结束, 回滚等等, 比较灵活, 而声明式事务就是在方法上加上一个注解, @Transactional, Spring会自动帮我管理该事务的整个生命周期
	- 如何实现事务的: 通过AOP来实现的, 当在方法上加上@Transactional注解时, Spring会自动创建一个当前Bean的代理对象, 通过这个代理对象来管理事务的进行;          
	- 声明式事务不需要在代理里面添加额外的代码逻辑, 但是粒度只能到方法级别, 而不能到代码块
	- 事务传播机制: (生效的前提就是通过代理对象调用)就是多个事务方法互相调用时, 事务如何传播;  Spring中有7种事务传播机制, 其中REQUIRED是默认的传播行为
		- **REQUIRED**: 如果当前存在事务, 就加入进去; 不存在就新建事务
		- SUPPORTS: 支持当前事务, 如果没有, 就以非事务的方式执行
		- MANDATORY: 使用当前事务, 如果当前没有事务就报错
		- **REQUIRES_NEW**: 新建事务, 如果存在事务, 就将该事务挂起
		- NOT_SUPPORTED: 以非事务方式执行操作。如果当前存在事务，把当前事务挂起。
		- **NESTED**: 如果存在事务, 就在当前事务中创建一个嵌套事务. 如果没有, 就与REQUIRED类似, 嵌套事务回滚不影响外层事务, 通常只回滚到保存点
		- NEVER: 以非事务的方式运行, 如果当前存在事务就报错
	- @Transactional失效的常见场景: 
		- 自调用: 同一个类里面, 一个非事务的方法调用了另一个加了该注解的方法, 事务不生效, 需要使用代理对象去调用. 因为Spring事务是通过AOP代理实现的, 自调用使用的是this, 不走代理对象
		- Spring默认只对public生效, 该注解加在private, protected这些方法上不生效
		- 异常类型不匹配, 该注解默认只对RuntimeException和Error回滚, 如果事务抛出受检异常（checked exception），需要通过rollbackFor显式指定，否则事务不会回滚
		- 方法内部使用try-catch捕获了异常, 但是没有抛出, Spring感知不到异常，事务自然不会回滚
		- Bean没有被Spring管理, 类没有加@Component或@Service等注解，不是Spring管理的Bean，@Transactional不起作用

- 什么是SpringBoot: SpringBoot就是一个基于Spring的快速开发工具包, 在传统的Spring开发种, 我们需要进行大量的xml配置文件, 还要手动管理各自jar包的依赖关系, 非常繁琐, 而SpringBoot通过起步依赖和自动装配解决了这些问题. 举个例子, 我在做RAG的问答助手这个项目的时候, 我在xml文件种引入spring-boot-starter-web, spring-boot-starter-data-redis依赖就自动完成了web和redis的连接, 不需要任何繁琐的配置代码. 同时SpringBoot预设了很多默认的配置, 比如内置Tomcat服务器, 可以直接打包成jar包运行等等. 它解决了传统Spring配置复杂, 管理依赖麻烦的问题.
- ==SpringBoot自动装配==: 在@SpringBootApplication中有三个核心注解, 分别是@SpringBootConfiguration(标记这是个SpringBoot配置类), @EnableAutoConfiguration(启动自动装配)和@ComponentScan(启用组件扫描).                                                                 其中@EnableAutoConfiguration是自动装配的关键, 它通过@Import导入了AutoConfigurationImportSelector类, 这个类的getCondidateConfigurations方法从spring.factories和AutoConfiguration.imports加载配置类, 加载完之后, 还要进行去重->排除用户指定排除的类->通过条件注解过滤
	- 条件注解: 
		- @ConditionalOnClass：类路径上存在某个类时才生效
		- @ConditionalOnMissingBean：容器中不存在某个Bean时才生效
		- @ConditionalOnProperty：配置文件中某个属性满足条件时才生效

- Spring不推荐用@Autowired: 现在一般使用
	- 一个final关键字；    
	- 一个@RequiredArgsConstructor注解(放到这个类的最上面)