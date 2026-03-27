# 一、Spring 核心概念（IOC / AOP / Bean）
## 1. 谈一下你对 Spring 的理解？核心是什么？
**15秒简答**  
Spring 是一个轻量级 Java 企业级开发框架，核心是 IOC（控制反转）和 AOP（面向切面编程），通过容器管理对象的生命周期和依赖关系，实现解耦和简化开发。  
**3分钟详答**  
Spring 最早是为了解决 J2EE/EJB 过于臃肿的问题而诞生，目标是“简化企业级 Java 开发”。它是一个分层的一站式框架，从表现层（Spring MVC）、业务层到持久层都提供解决方案，但核心只有两点：  
- IOC（Inversion of Control，控制反转）：  
  传统方式里，对象自己 new 依赖对象，控制权在对象手里；Spring 里，对象的创建、依赖关系都交给 IOC 容器，对象被动接收注入，所以叫“控制反转”。依赖注入（DI）是 IOC 的具体实现方式，常见有构造器注入、setter 注入等。  
- AOP（Aspect-Oriented Programming，面向切面编程）：  
  将横切逻辑（日志、事务、权限等）从业务代码中抽离出来，封装成切面，在运行时通过动态代理织入到目标方法中。这样可以减少重复代码，降低耦合，是对 OOP 的补充。  
Spring 框架包含 Core、Context、AOP、Web、ORM 等模块，现在我们日常说的 Spring 通常指 Spring Framework，再加上 Spring Boot、Spring Cloud 等构成“全家桶”。  
**追问**
1. **追问：IOC 和 DI 是什么关系？**  
   IOC 是思想，DI 是实现方式。IOC 强调“控制权交给容器”，DI 强调“容器在运行时把依赖注入给对象”，两者描述同一件事的不同侧面。  
2. **追问：Spring 用了哪些设计模式？**  
   工厂模式（BeanFactory/ApplicationContext）、单例模式（默认 Bean 作用域）、代理模式（AOP）、模板方法模式（JdbcTemplate/RedisTemplate）、观察者模式（ApplicationEvent）、适配器模式（HandlerAdapter）等。  
---
## 2. 什么是 IOC？有什么好处？
**15秒简答**  
IOC（控制反转）是把对象的创建和管理权交给 Spring 容器，对象不再自己 new 依赖，而是被动注入。好处是解耦、便于测试和维护，支持声明式配置和依赖管理。  
**3分钟详答**  
在传统程序中，A 依赖 B 时，通常在 A 中直接 `new B()`，A 和 B 强耦合。引入 IOC 后：  
- 对象不再负责创建依赖对象，而是通过 XML、注解或 Java 配置把依赖关系告诉容器。  
- 容器在启动或运行时，根据配置创建并注入依赖，A 只需要声明“我需要 B”，容器会把 B 注入进来。  
从设计模式角度看，IOC 容器就是一个大工厂，底层使用反射和工厂模式创建对象，通过依赖注入（构造器/setter）实现对象间解耦。  
**追问**
1. **追问：IOC 容器有哪些常见接口？**  
   最核心的是 `BeanFactory` 和 `ApplicationContext`。`BeanFactory` 是基础容器，延迟加载；`ApplicationContext` 继承自它，提供国际化、事件机制、自动扫描等更多企业特性，一般开发中常用 `ApplicationContext`。  
2. **追问：依赖注入有几种方式？**  
   常见三种：构造器注入、setter 方法注入、字段注入（@Autowired 字段）。构造器注入适合强制依赖；setter 注入适合可选依赖；字段注入最简单，但不利于单元测试。  
---
## 3. 什么是 AOP？解决了什么问题？
**15秒简答**  
AOP（面向切面编程）将横切关注点（日志、事务、权限）封装成切面，在运行时通过动态代理织入到目标方法。它解决了业务代码和系统级代码纠缠在一起的问题，降低重复耦合。  
**3分钟详答**  
在没有 AOP 时，日志、事务、权限等代码会散落在各个业务方法中，形成大量重复代码，业务逻辑难以看清。AOP 做的事情是：  
- 把这些横切逻辑抽取出来，定义成“切面”（@Aspect）。  
- 通过“切点”（Pointcut）指定在哪些类的哪些方法上生效。  
- 通过“通知”（Advice：@Before、@After、@Around 等）在方法执行前后织入增强逻辑。  
底层实现依靠动态代理：如果目标类实现了接口，Spring 默认使用 JDK 动态代理；如果没有接口，则使用 CGLIB 生成子类代理。  
**追问**
1. **追问：AOP 有哪些常见应用场景？**  
   事务管理（@Transactional）、日志记录、性能监控、权限校验、缓存处理、异常处理等。  
2. **追问：Spring AOP 和 AspectJ 有什么区别？**  
   Spring AOP 是运行时动态代理，只能拦截方法级别；AspectJ 是编译时/加载时织入，功能更强，可以拦截字段、构造器等。Spring 推荐使用 Spring AOP，足够满足大部分场景。  
---
## 4. Spring Bean 的生命周期是怎样的？
**15秒简答**  
Bean 生命周期大致包括：实例化 → 属性赋值 → 初始化（ Aware 接口 → BeanPostProcessor → @PostConstruct/init-method）→ 使用 → 销毁（@PreDestroy/destroy-method）。  
**3分钟详答**  
以单例 Bean 为例，Spring 容器管理 Bean 从创建到销毁的全过程：  
1. 实例化：根据 BeanDefinition，通过反射创建 Bean 实例。  
2. 属性赋值：依赖注入，将相关依赖注入到 Bean 中。  
3. 初始化：  
   - 如果实现各种 Aware 接口（BeanNameAware、ApplicationContextAware 等），容器会把 Bean 名字、容器实例注入进来。  
   - 执行 BeanPostProcessor 的前置方法 `postProcessBeforeInitialization`。  
   - 如果实现 InitializingBean，调用 `afterPropertiesSet()`，或执行自定义 init-method。  
   - 执行 BeanPostProcessor 的后置方法 `postProcessAfterInitialization`，很多 AOP 代理在此处生成。  
4. 使用：Bean 已经就绪，可以对外提供服务。  
5. 销毁：容器关闭时，执行 DisposableBean 的 `destroy()` 或自定义 destroy-method。  
**追问**
6. **追问：BeanPostProcessor 有什么用？**  
   它是 Spring 留给开发者的扩展点，可以在 Bean 初始化前后做自定义处理，比如生成代理对象、校验属性、注入额外信息等。AOP 代理、事务代理等都是通过 BeanPostProcessor 完成的。  
7. **追问：prototype Bean 的生命周期和 singleton 有什么不同？**  
   prototype Bean 在每次 getBean 时都会创建新实例，容器只负责创建和初始化，不负责销毁；因此不会执行 destroy 方法，生命周期交由客户端管理。  
---
## 5. Spring Bean 的作用域有哪些？
**15秒简答**  
常见作用域：singleton（默认，单例）、prototype（每次获取新建）、request（每次 HTTP 请求一个 Bean）、session（每个 HTTP Session 一个 Bean）、application（ServletContext 级别）。  
**3分钟详答**  
Spring 定义了多种 Bean 作用域，用来控制 Bean 实例的可见范围和生命周期：  
- singleton：默认作用域，每个 Spring 容器中只有一个 Bean 实例，所有请求共享该实例。  
- prototype：每次从容器获取 Bean 时都创建一个新实例，适合有状态且不能共享的对象。  
- request：每次 HTTP 请求创建一个 Bean，仅 Web 应用环境可用。  
- session：同一个 HTTP Session 共享一个 Bean，不同 Session 使用不同实例。  
- application：ServletContext 生命周期内一个 Bean，适用于全局共享组件。  
**追问**
1. **追问：有状态 Bean 和无状态 Bean 分别适合哪种作用域？**  
   无状态 Bean（如 Service、DAO）适合 singleton；有状态 Bean（如用户会话信息）适合 prototype 或 session/request。  
2. **追问：在 Web 环境中使用 request/session 作用域要注意什么？**  
   需要配置相应的 Web 环境容器（如 DispatcherServlet），并通过代理模式（`ScopedProxyMode.TARGET_CLASS`）解决将短生命周期 Bean 注入到长生命周期 Bean 的问题。  
---
## 6. Spring 是如何解决循环依赖的？
**15秒简答**  
Spring 通过“三级缓存 + 提前暴露半成品 Bean”解决单例 Bean 的 setter 循环依赖。构造器循环依赖无法自动解决。  
**3分钟详答**  
当两个 Bean 相互依赖时（A 依赖 B，B 依赖 A），Spring 使用三级缓存：  
- 一级缓存 `singletonObjects`：存放完全初始化好的单例 Bean。  
- 二级缓存 `earlySingletonObjects`：存放已经实例化但未完成属性填充和初始化的“半成品 Bean”。  
- 三级缓存 `singletonFactories`：存放 ObjectFactory，用于生成早期 Bean 引用（如果是代理对象，可以提前生成代理）。  
以 A → B → A 为例：  
1. 创建 A，实例化后放入三级缓存（工厂对象）。  
2. A 填充属性时发现需要 B，开始创建 B。  
3. B 填充属性时发现需要 A，从一级/二级缓存没找到，从三级缓存获取 A 的工厂，生成 A 的早期引用，放入二级缓存，B 拿到 A 的引用，继续完成初始化。  
4. B 完成后，A 继续完成初始化，最终把 A 从二级缓存移到一级缓存。  
**追问**
5. **追问：为什么需要三级缓存，二级不够吗？**  
   三级缓存主要是为了处理 AOP 场景：如果 Bean 需要代理，可以在三级缓存中通过 `getEarlyBeanReference` 提前生成代理对象，保证注入的是同一个代理实例，而不是先注入原始对象再生成代理导致不一致。  
6. **追问：哪些循环依赖无法解决？**  
   构造器注入的循环依赖、原型（prototype）作用域 Bean 的循环依赖，Spring 无法自动解决，需要改用 setter 注入或重新设计依赖结构。  
---
# 二、Spring 事务与 AOP 应用
## 7. Spring 事务的实现方式和原理是什么？
**15秒简答**  
Spring 事务分为编程式事务（编码控制）和声明式事务（基于 @Transactional）。底层通过 AOP 代理，在方法前后开启/提交/回滚事务，核心是 PlatformTransactionManager 和事务拦截器。  
**3分钟详答**  
Spring 提供两种事务管理方式：  
- 编程式事务：使用 TransactionTemplate 或 PlatformTransactionManager 在代码中显式控制事务边界，灵活性高但侵入业务代码。  
- 声明式事务：在方法或类上使用 @Transactional，由 Spring 通过 AOP 在方法调用前后自动开启、提交或回滚事务，代码侵入小，实际项目中最常用。  
实现原理：  
1. Spring 通过 AOP 为标注 @Transactional 的 Bean 生成代理对象。  
2. 代理在调用目标方法前，根据事务配置获取或创建事务（使用 PlatformTransactionManager，如 DataSourceTransactionManager）。  
3. 方法正常返回提交事务；抛出运行时异常或指定异常时回滚事务。  
4. 事务传播行为（如 REQUIRED、REQUIRES_NEW）控制方法嵌套时的事务边界。  
**追问**
5. **追问：Spring 事务用到了哪些设计模式？**  
   策略模式（不同事务管理器实现 PlatformTransactionManager）、代理模式（AOP 代理）、模板方法模式（TransactionTemplate）。  
6. **追问：声明式事务为什么只能用在 public 方法？**  
   Spring 默认使用代理方式实现事务，private/protected 方法无法被外部代理调用，事务增强无法生效，因此事务方法一般要求 public。  
---
## 8. @Transactional 的常见属性有哪些？默认值是什么？
**15秒简答**  
常见属性：propagation（传播行为）、isolation（隔离级别）、readOnly（是否只读）、timeout（超时时间）、rollbackFor / noRollbackFor（回滚异常类型）。默认传播行为 REQUIRED，隔离级别 DEFAULT（使用数据库默认）。  
**3分钟详答**  
- propagation：定义事务传播行为，默认 REQUIRED，有事务就加入，没有就新建。  
- isolation：事务隔离级别，默认 DEFAULT，使用底层数据库默认隔离级别。  
- readOnly：是否只读事务，对某些数据库优化有意义。  
- timeout：事务超时时间，单位秒。  
- rollbackFor：指定哪些异常触发回滚，默认只对 RuntimeException 和 Error 回滚。  
- noRollbackFor：指定哪些异常不回滚。  
**追问**
1. **追问：什么时候需要指定 rollbackFor = Exception.class？**  
   当业务方法抛出受检异常（checked exception）也希望事务回滚时，需要显式指定，否则默认只对 RuntimeException 回滚。  
2. **追问：readOnly=true 就一定不能写数据吗？**  
   不是绝对，只是提示数据库当前事务只读，某些数据库会优化；如果执行插入/更新，仍然可以执行，但可能违反业务预期或数据库约束。  
---
## 9. Spring 事务传播行为有哪些？常用哪些？
**15秒简答**  
七种传播行为：REQUIRED（默认）、SUPPORTS、MANDATORY、REQUIRES_NEW、NOT_SUPPORTED、NEVER、NESTED。常用 REQUIRED 和 REQUIRES_NEW。  
**3分钟详答**  
Spring 定义了七种传播行为，解决多个事务方法相互调用时的事务边界问题：  
- REQUIRED：有事务就加入，没有就新建（最常用）。  
- SUPPORTS：有事务就加入，没有就以非事务运行。  
- MANDATORY：必须在事务中运行，否则抛异常。  
- REQUIRES_NEW：总是新建事务，挂起当前事务（如果存在），新旧事务相互独立。  
- NOT_SUPPORTED：以非事务方式运行，挂起当前事务。  
- NEVER：以非事务方式运行，如果存在事务则抛异常。  
- NESTED：如果当前有事务，则在嵌套事务中运行；否则新建事务。  
**追问**
1. **追问：REQUIRED 和 REQUIRES_NEW 有什么区别？**  
   REQUIRED 使用同一个事务，只要有一个方法回滚，整个事务回滚；REQUIRES_NEW 新开一个独立事务，内部事务回滚不影响外部事务，适合日志、通知等“必须记录但不能因为业务失败而丢失”的场景。  
2. **追问：NESTED 和 REQUIRES_NEW 有什么区别？**  
   NESTED 是嵌套事务，依赖外层事务，部分回滚可以只回滚到保存点；外层回滚则整体回滚。REQUIRES_NEW 是完全独立的新事务，两者完全独立提交/回滚。  
---
## 10. Spring 事务在哪些情况下会失效？
**15秒简答**  
常见失效场景：数据库引擎不支持事务、类没有被 Spring 管理、方法非 public、自调用未走代理、异常类型不匹配、传播行为设置错误、多线程调用等。  
**3分钟详答**  
Spring 事务失效的原因很多，主要包括：  
1. 数据库层面：使用 MyISAM 等不支持事务的引擎。  
2. Bean 不在 Spring 管理：例如 @Service 注解没加，或 new 出来的对象。  
3. 方法访问权限：事务方法不是 public，代理无法拦截。  
4. 自调用问题：在同一个类中，一个非事务方法调用事务方法，通过 `this.method()` 调用，绕过代理对象，事务不生效。  
5. 异常处理不当：默认只对 RuntimeException 回滚，受检异常且未指定 rollbackFor 时不回滚；或者自己 catch 掉异常没有再次抛出。  
6. 传播行为错误：设置为 NOT_SUPPORTED 或 NEVER，事务根本不会生效。  
7. 多线程：事务上下文是 ThreadLocal 粒度，新线程中的操作不在同一个事务中。  
**追问**
8. **追问：同一个类中自调用导致事务失效，有哪些解决方式？**  
   - 将方法拆到另一个 Bean 中，通过外部调用触发代理。  
   - 注入当前 Bean 自己，通过代理调用 `self.method()`。  
   - 使用 AopContext.currentProxy() 获取当前代理对象调用。  
2. **追问：为什么多线程会导致事务失效？**  
   Spring 事务上下文绑定到当前线程，新线程有自己的 ThreadLocal，不会继承外层事务，因此新线程中的数据库操作不在同一个事务中。  
---
## 11. Spring AOP 中 JDK 动态代理和 CGLIB 有什么区别？
**15秒简答**  
JDK 动态代理基于接口，要求目标类实现接口；CGLIB 基于继承，生成目标类的子类，可以代理没有接口的类。Spring 会根据目标类是否实现接口自动选择。  
**3分钟详答**  
- JDK 动态代理：  
  - 通过反射机制生成代理类，代理类实现和目标对象相同的接口。  
  - 要求目标类必须实现至少一个接口。  
  - 只能对接口方法进行代理。  
- CGLIB：  
  - 通过生成目标类的子类，覆盖非 final 方法实现代理。  
  - 不要求目标类实现接口，但不能代理 final 类或 final 方法。  
  - 性能略高于 JDK 动态代理（在老版本 JDK 中，现在差距不大）。  
Spring 默认策略：如果目标类实现接口，优先使用 JDK 动态代理；否则使用 CGLIB。也可以强制使用 CGLIB（如配置 `proxyTargetClass=true`）。  
**追问**
1. **追问：什么时候需要强制使用 CGLIB？**  
   当目标类没有实现接口，或者需要代理类而不是接口时，可以强制使用 CGLIB。  
2. **追问：Spring AOP 和 AspectJ 在代理方式上有什么差异？**  
   Spring AOP 是运行时动态代理（JDK/CGLIB），只支持方法级连接点；AspectJ 可以在编译期或类加载时织入，支持更细粒度的连接点（字段、构造器等），性能更好但需要特殊编译器支持。  
---
# 三、Spring MVC 常见问题
## 12. 简述 Spring MVC 的执行流程。
**15秒简答**  
请求 → DispatcherServlet → HandlerMapping 找到 Handler → HandlerAdapter 执行 Handler → 返回 ModelAndView → ViewResolver 解析视图 → 渲染视图并响应。  
**3分钟详答**  
Spring MVC 是基于 MVC 设计模式的 Web 框架，核心是 DispatcherServlet 前端控制器。典型流程：  
1. 用户请求到达 DispatcherServlet。  
2. DispatcherServlet 调用 HandlerMapping，根据 URL 找到对应的 Controller 方法（Handler），并返回一个 HandlerExecutionChain（包含 Handler 和拦截器）。  
3. DispatcherServlet 调用 HandlerAdapter，由 HandlerAdapter 调用具体的 Handler 方法。  
4. Handler 执行业务逻辑，返回 ModelAndView（模型数据和视图名）。  
5. DispatcherServlet 将 ModelAndView 交给 ViewResolver 解析成具体 View（如 JSP/Thymeleaf/JSON）。  
6. View 进行渲染，将模型数据填充到视图中。  
7. DispatcherServlet 将响应返回给用户。  
**追问**
8. **追问：拦截器（Interceptor）的执行时机是什么？**  
   拦截器可以在 Handler 执行前（`preHandle`）、执行后但视图渲染前（`postHandle`）、视图渲染完成后（`afterCompletion`）进行拦截处理，常用于登录校验、日志记录、权限检查。  
9. **追问：Filter 和 Interceptor 有什么区别？**  
   Filter 是 Servlet 规范中的过滤器，由 Servlet 容器管理；Interceptor 是 Spring MVC 的组件，由 Spring 容器管理。Filter 作用范围更大，可以拦截所有请求；Interceptor 只能拦截到 DispatcherServlet 处理的请求。  
---
## 13. Spring MVC 常用注解有哪些？
**15秒简答**  
常用注解：@Controller、@RestController、@RequestMapping、@GetMapping/PostMapping、@RequestParam、@RequestBody、@ResponseBody、@PathVariable、@RequestHeader 等。  
**3分钟详答**  
- @Controller：标识控制层组件，配合 @RequestMapping 使用。  
- @RestController：相当于 @Controller + @ResponseBody，用于 RESTful 接口。  
- @RequestMapping：映射请求 URL 和方法，可标注在类和方法上。  
- @GetMapping/@PostMapping/@PutMapping/@DeleteMapping：简化写法，对应 HTTP 方法。  
- @RequestParam：绑定请求参数到方法参数。  
- @RequestBody：将请求体（如 JSON）绑定到方法参数。  
- @ResponseBody：将返回值直接写入响应体，常用于返回 JSON。  
- @PathVariable：绑定 URL 中的占位符参数。  
- @RequestHeader：获取请求头信息。  
**追问**
1. **追问：@RequestParam 和 @PathVariable 有什么区别？**  
   @RequestParam 获取查询参数（如 `?name=xxx`）；@PathVariable 获取路径参数（如 `/user/{id}` 中的 id）。  
2. **追问：如何接收 JSON 请求体并返回 JSON？**  
   使用 @RequestBody 将 JSON 绑定到对象，配合 @RestController 或 @ResponseBody 将返回对象自动序列化为 JSON。  
---
## 14. Spring MVC 如何处理异常？
**15秒简答**  
可以通过自定义 `@ControllerAdvice` + `@ExceptionHandler` 实现全局异常处理，统一返回错误信息；也可以实现 `HandlerExceptionResolver` 接口。  
**3分钟详答**  
Spring MVC 提供多种异常处理方式：  
- 局部异常处理：在 Controller 中使用 `@ExceptionHandler` 处理本类内部异常。  
- 全局异常处理：定义 `@ControllerAdvice` 类，在其中使用 `@ExceptionHandler` 统一处理所有 Controller 的异常，可以返回统一错误结构（如 JSON）。  
- 实现 `HandlerExceptionResolver`：通过重写 `resolveException` 方法自定义异常解析逻辑。  
**追问**
1. **追问：全局异常处理返回 JSON 如何实现？**  
   在 `@ControllerAdvice` 类中的 `@ExceptionHandler` 方法上标注 `@ResponseBody`，或者直接使用 `@RestControllerAdvice`，返回对象会被自动序列化为 JSON。  
2. **追问：如何区分不同异常返回不同 HTTP 状态码？**  
   在 `@ExceptionHandler` 方法参数中注入 `HttpServletResponse` 或使用 `ResponseEntity`，手动设置状态码。  
---
# 四、Spring Boot 基础（自动配置与使用）
## 15. Spring Boot 的自动配置原理是什么？
**15秒简答**  
Spring Boot 通过 @EnableAutoConfiguration + `spring.factories`/`AutoConfiguration.imports` 加载大量自动配置类，再根据 @ConditionalOnXxx 条件按需装配 Bean，实现“约定优于配置”。  
**3分钟详答**  
核心流程：  
1. 启动类上的 @SpringBootApplication 包含 @EnableAutoConfiguration。  
2. @EnableAutoConfiguration 通过 @Import 引入 AutoConfigurationImportSelector。  
3. AutoConfigurationImportSelector 使用 SpringFactoriesLoader 扫描所有 jar 包的 `META-INF/spring.factories`，找到 `EnableAutoConfiguration` 对应的自动配置类全名。  
4. 这些自动配置类（如 DataSourceAutoConfiguration、WebMvcAutoConfiguration）上都有 @ConditionalOnXxx 条件注解，根据类路径是否存在某些类、是否已有某个 Bean 等条件决定是否生效。  
5. 用户可以通过自定义配置或自定义 Bean 覆盖自动配置的默认行为。  
**追问**
6. **追问：@ConditionalOnMissingBean 有什么作用？**  
   当容器中不存在指定 Bean 时，自动配置才会生效；如果用户自己定义了该 Bean，自动配置就不会创建，从而实现“用户配置优先”。  
7. **追问：如何排除某个自动配置类？**  
   使用 `@SpringBootApplication(exclude = {XXXAutoConfiguration.class})` 或在配置文件中设置 `spring.autoconfigure.exclude`。  
---
## 16. Spring Boot 配置文件的加载顺序是怎样的？
**15秒简答**  
内部配置：`file:./config/` > `file:./` > `classpath:/config/` > `classpath:/`；外部配置：命令行参数 > 系统环境变量 > jar 外部配置文件 > jar 内部配置文件。高优先级覆盖低优先级。  
**3分钟详答**  
Spring Boot 支持多环境、多层级配置，优先级大致从高到低：  
1. 命令行参数，如 `--server.port=8081`。  
2. 操作系统环境变量。  
3. jar 包外部的 application-{profile}.yml。  
4. jar 包内部的 application-{profile}.yml。  
5. jar 包外部的 application.yml。  
6. jar 包内部的 application.yml。  
内部文件路径优先级：项目根目录下 `config/` > 项目根目录 > 类路径 `config/` > 类路径根目录。相同属性高优先级覆盖低优先级，未冲突则互补。  
**追问**
7. **追问：如何指定不同环境配置文件？**  
   使用 `application-{profile}.yml`，通过 `spring.profiles.active=dev` 激活指定环境。  
8. **追问：配置属性如何绑定到 Bean？**  
   使用 `@ConfigurationProperties(prefix = "xxx")` 将配置文件中的前缀属性注入到 Bean 字段中。  
---
## 17. Spring Boot 常用 Starter 有哪些？
**15秒简答**  
常用：spring-boot-starter-web（Web 开发）、starter-data-jpa（JPA）、starter-data-redis（Redis）、starter-security（安全）、starter-test（测试）、starter-actuator（监控）等。  
**3分钟详答**  
Starter 是 Spring Boot 提供的“依赖描述模块”，把某个功能所需的依赖和自动配置打包在一起，开发者只需引入一个 Starter，就自动引入了相关库和默认配置：  
- spring-boot-starter-web：引入 Spring MVC、内嵌 Tomcat、Jackson 等。  
- spring-boot-starter-data-jpa：引入 Hibernate、Spring Data JPA、数据源等。  
- spring-boot-starter-data-redis：引入 Redis 客户端（Lettuce/Jedis）和自动配置。  
- spring-boot-starter-security：引入 Spring Security，提供基础安全配置。  
- spring-boot-starter-test：引入 JUnit、Mockito 等测试依赖。  
- spring-boot-starter-actuator：提供生产就绪的监控端点。  
**追问**
1. **追问：Starter 和自动配置有什么关系？**  
   Starter 负责聚合依赖，自动配置负责根据这些依赖和条件自动装配 Bean，两者配合实现“开箱即用”。  
2. **追问：如何自定义一个 Starter？**  
   编写自动配置类（@Configuration + @ConditionalOnXxx），编写 `META-INF/spring.factories` 或 `AutoConfiguration.imports` 指定自动配置类，打包成独立模块供其他项目引用。  
---
# 五、综合与实战场景
## 18. 项目中是如何使用 Spring 事务的？有没有遇到事务失效的情况？
**15秒简答**  
通常在 Service 层方法上加 @Transactional，指定传播行为和回滚异常；遇到过自调用、异常被 catch、多线程等情况导致事务失效，通过拆分 Bean、显式指定 rollbackFor、使用编程式事务等方式解决。  
**3分钟详答**  
在项目中，常见做法：  
- 对涉及多表操作或需要保证一致性的 Service 方法标注 @Transactional。  
- 根据业务选择合适的传播行为，例如日志操作使用 REQUIRES_NEW，避免因为业务异常导致日志回滚。  
- 对于可能抛出受检异常的场景，指定 `rollbackFor = Exception.class`。  
事务失效的典型情况：  
- 在同一个类中，非事务方法直接调用事务方法（自调用）。  
- 在事务方法内部 catch 了异常但没有再次抛出。  
- 方法非 public，或者类不在 Spring 容器中。  
- 使用了不支持事务的数据库引擎或数据源未配置事务管理器。  
**追问**
1. **追问：如何排查事务是否生效？**  
   可以通过数据库事务日志、Debug 查看代理对象类型、打印事务管理器日志，或在方法中模拟异常观察是否回滚。  
2. **追问：长事务有什么问题？如何优化？**  
   长事务会导致数据库连接长时间占用、锁竞争增加、回滚成本高。可以通过拆分业务、减少事务方法粒度、使用编程式事务控制关键步骤等方式优化。  
---
## 19. 你在项目中是怎么使用 Spring AOP 的？
**15秒简答**  
主要用于日志记录、性能监控、权限校验等。定义切面类，使用 @Aspect + @Pointcut 指定切点，通过 @Around/@Before/@After 等通知在方法前后执行增强逻辑。  
**3分钟详答**  
以日志为例：  
- 定义切面类，使用 @Aspect 标注。  
- 使用 @Pointcut 定义切点表达式，例如 `execution(* com.example.service..*(..))`。  
- 使用 @Around 或 @Before/@After 在方法前后记录日志，包括请求参数、返回结果、执行时间等。  
- 在权限场景中，可以通过 AOP 校验用户是否有权访问某个资源。  
AOP 的好处是业务代码只需要关注业务逻辑，横切逻辑统一维护，便于扩展。  
**追问**
1. **追问：切点表达式有哪些常用写法？**  
   常用 `execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) throws-pattern?)`，如 `execution(public * com.example.service.*.*(..))`。  
2. **追问：环绕通知 @Around 与其他通知有什么区别？**  
   @Around 可以完全控制目标方法的执行，包括是否执行、何时执行、参数修改、返回值修改等；其他通知只能在特定位置执行，无法控制目标方法本身。  
---
## 20. 你是如何理解“约定优于配置”的？在 Spring/Spring Boot 中体现在哪些地方？
**15秒简答**  
“约定优于配置”就是框架提供一套默认约定，只要遵循这些约定，就无需显式配置；在 Spring Boot 中体现为默认包扫描、默认端口号、默认依赖管理、自动配置等。  
**3分钟详答**  
Spring Boot 通过约定优于配置，大幅减少样板代码：  
- 默认扫描启动类所在包及其子包下的所有 @Component/@Service/@Repository/@Controller 等 Bean。  
- 默认内嵌 Tomcat，默认端口 8080，默认使用 Jackson 序列化 JSON。  
- 默认提供 starter 依赖管理，通过引入 starter 自动拉取一组兼容依赖。  
- 自动配置根据类路径下的依赖自动配置数据源、Redis、消息队列等组件。  
只要项目结构、依赖、命名遵循约定，几乎不需要写额外配置；如果需要自定义，再通过配置文件或 Java 配显式覆盖。  
**追问**
1. **追问：如果你不遵守这些约定会怎样？**  
   需要显式配置，例如修改包扫描路径、修改端口、配置数据源等，但不会影响功能，只是需要多写一些配置。  
2. **追问：Spring Boot 和传统 Spring 项目相比，最大的优势是什么？**  
   简化配置、内嵌容器、自动配置、生产级监控 Actuator、依赖版本统一管理等，让开发者可以更专注于业务逻辑。  
---
## 21. 如何在 Spring 中实现多数据源事务？
**15秒简答**  
可以配置多个 DataSource 和对应的事务管理器，通过 @Transactional(“transactionManagerName”) 指定使用哪个事务管理器；或者使用 JTA 分布式事务（在简单场景下一般不用）。  
**3分钟详答**  
在单机多数据源场景下：  
1. 配置多个 DataSource（如 primaryDataSource、secondaryDataSource）。  
2. 为每个 DataSource 配置对应的 PlatformTransactionManager，如 primaryTransactionManager、secondaryTransactionManager。  
3. 在 Service 方法上使用 @Transactional(“primaryTransactionManager”) 指定事务管理器。  
需要注意：跨数据源事务不能通过简单的本地事务保证一致性，需要考虑分布式事务方案（如 Seata、基于消息的最终一致性等）。  
**追问**
4. **追问：多事务管理器情况下，默认事务管理器是哪一个？**  
   可以通过 `@Primary` 标注某个事务管理器为默认，或者显式指定事务管理器名称。  
5. **追问：分布式事务有哪些常见方案？**  
   常见：基于两阶段提交的 JTA、基于补偿的 TCC、基于本地消息表/事务消息的最终一致性、Seata 等。  
---
## 22. Spring 如何与 MyBatis/MyBatis-Plus 整合？
**15秒简答**  
Spring Boot 下引入 mybatis-spring-boot-starter，配置数据源和 mybatis 配置（mapper 文件位置、别名包等），在接口上标注 @Mapper 或使用 @MapperScan 扫描 Mapper 接口，由 Spring 创建代理实现类。  
**3分钟详答**  
整合步骤：  
1. 引入 starter：mybatis-spring-boot-starter 或 mybatis-plus-boot-starter。  
2. 在 application.yml 中配置数据源、mapper 文件位置、别名包等。  
3. 在启动类上使用 @MapperScan 指定 Mapper 接口所在包，或在每个 Mapper 接口上标注 @Mapper。  
4. Spring 会为每个 Mapper 接口创建代理对象，注入到 SqlSession 中，由 Spring 管理事务。  
MyBatis-Plus 在此基础上提供了通用 CRUD、分页插件、代码生成等功能，进一步简化开发。  
**追问**
5. **追问：#{} 和 ${} 有什么区别？**  
   `#{}` 使用预编译参数，安全防注入；`${}` 是直接拼接 SQL，有注入风险，一般用于动态表名、排序字段等。  
6. **追问：如何实现多租户或分库分表？**  
   可以通过 MyBatis 插件或中间件（如 ShardingSphere）实现，在执行 SQL 前根据租户 ID 或分片键路由到不同的数据源/表。  
---
## 23. Spring 中如何使用定时任务？
**15秒简答**  
可以使用 Spring 的 @Scheduled 注解，在启动类上 @EnableScheduling，然后在方法上标注 @Scheduled(cron/fixedRate/fixedDelay) 定义执行规则。  
**3分钟详答**  
Spring 提供对定时任务的支持：  
1. 在启动类或配置类上标注 @EnableScheduling。  
2. 在需要定时执行的方法上标注 @Scheduled，支持三种方式：  
   - cron：表达式定义时间。  
   - fixedRate：固定频率执行，从上次开始时间算起。  
   - fixedDelay：固定延迟执行，从上次结束时间算起。  
底层通过 TaskScheduler 调度线程池执行，适合轻量级定时任务；复杂场景可以使用 Quartz 等框架。  
**追问**
1. **追问：@Scheduled 默认是单线程执行，如何配置多线程？**  
   可以自定义 TaskScheduler 线程池，或配置 `spring.task.scheduling.pool.size`。  
2. **追问：分布式环境下如何避免任务重复执行？**  
   可以使用分布式锁（如 Redis 锁、Zookeeper 锁）保证同一时刻只有一个实例执行任务。  
---
## 24. Spring 中的事件机制是怎么用的？
**15秒简答**  
通过 ApplicationEventPublisher 发布事件，自定义事件继承 ApplicationEvent，监听器实现 ApplicationListener 或使用 @EventListener 注解，实现解耦的事件驱动模型。  
**3分钟详答**  
Spring 的事件机制基于观察者模式：  
1. 定义事件类，继承 ApplicationEvent，封装事件信息。  
2. 通过 ApplicationEventPublisher 发布事件。  
3. 监听器通过实现 ApplicationListener 或在方法上使用 @EventListener 监听事件。  
典型场景：用户注册后发送邮件、记录日志等，可以将这些操作拆成监听器，主流程只关注注册逻辑。  
**追问**
4. **追问：事件监听是同步还是异步？如何异步？**  
   默认同步，可以通过 @Async 注解或在事件发布线程池中配置异步执行，实现异步事件处理。  
5. **追问：Spring 事件和消息队列（MQ）有什么区别？**  
   Spring 事件是进程内消息，轻量但不可跨应用；MQ 是分布式消息，支持跨应用、可靠传输、重试等，适合复杂分布式场景。  
---
## 25. 如何理解 Spring 的“非侵入式”设计？
**15秒简答**  
“非侵入式”指业务代码不需要依赖 Spring 特定 API，可以通过 POJO + 注解/XML 配置完成依赖注入和事务管理，使代码更易测试、易迁移。  
**3分钟详答**  
在传统 EJB 等框架中，业务类往往需要实现框架接口或继承框架类，代码与框架强耦合。Spring 通过 IOC 和 AOP，让业务类保持简单 POJO 形式：  
- 只需要在类上加 @Component/@Service 等注解，或通过 XML 配置，就能让 Spring 管理其生命周期。  
- 事务、日志等通过代理织入，业务类本身不需要知道 Spring 的存在。  
这样业务代码可以脱离容器独立测试，也更容易切换到其他框架。  
**追问**
1. **追问：那 Spring 注解算不算侵入？**  
   从严格意义上说，注解也算一种侵入，但相比实现接口、继承类，侵入程度低很多，且可以通过 Java Config 或 XML 替代注解。  
2. **追问：如何让代码尽量不依赖 Spring？**  
   使用面向接口编程，业务依赖接口而非实现；通过构造器注入依赖，尽量少用 Spring 特有注解，保留 POJO 风格。  
---
## 26. Spring 中如何实现文件上传？
**15秒简答**  
在 Spring MVC 中，通过 MultipartFile 接收文件，配置 `spring.servlet.multipart.max-file-size` 等属性限制上传大小，Controller 方法中使用 `@RequestParam("file") MultipartFile file` 接收。  
**3分钟详答**  
Spring MVC 内置支持文件上传：  
1. 在表单中设置 `enctype="multipart/form-data"`。  
2. Controller 方法参数使用 `MultipartFile file` 或 `@RequestPart` 接收文件。  
3. 通过 `file.getInputStream()`、`file.transferTo()` 等方法保存文件。  
4. 通过配置限制文件大小、请求大小等。  
**追问**
5. **追问：如何防止文件上传漏洞？**  
   校验文件类型、大小、文件名，避免直接使用用户提供的文件名，限制上传目录权限，避免上传可执行文件。  
6. **追问：大文件上传如何优化？**  
   可以使用分片上传、断点续传，前端切片后端合并，或使用对象存储（OSS）提供的上传接口。  
---
## 27. Spring 如何支持国际化（i18n）？
**15秒简答**  
通过 ResourceBundleMessageSource 配置国际化资源文件，根据请求头中的 Accept-Language 或用户选择语言，在页面上通过 `#{}message}` 等方式显示对应语言信息。  
**3分钟详答**  
Spring 的国际化支持：  
1. 定义不同语言的资源文件，如 `messages_zh_CN.properties`、`messages_en_US.properties`。  
2. 在 Spring 配置 ResourceBundleMessageSource，设置 basename 和编码。  
3. 在页面（Thymeleaf/JSP）中通过特定语法获取消息。  
在 Web 环境中，可以使用 LocaleResolver 解析用户区域信息，支持 Cookie、Session、请求头等多种方式。  
**追问**
4. **追问：如何动态切换语言？**  
   可以通过拦截器或 LocaleChangeInterceptor，根据请求参数或用户设置修改 LocaleResolver 中的 Locale。  
5. **追问：Spring Boot 中如何配置国际化？**  
   配置 `spring.messages.basename`、`spring.messages.encoding` 等，自动注册 MessageSource。  
---
## 28. Spring 中如何使用缓存？
**15秒简答**  
Spring 提供缓存抽象，通过 @Cacheable/@CachePut/@CacheEvict 注解，配合 CacheManager（如 RedisCacheManager、ConcurrentMapCacheManager）实现方法级缓存。  
**3分钟详答**  
使用步骤：  
1. 在启动类或配置类上标注 @EnableCaching。  
2. 配置 CacheManager，如 RedisCacheManager。  
3. 在方法上标注 @Cacheable(value = “user”, key = “#id”) 表示缓存结果；@CachePut 更新缓存；@CacheEvict 删除缓存。  
Spring 缓存抽象支持多种后端，如 Redis、Ehcache、Caffeine 等，业务代码不依赖具体缓存实现。  
**追问**
4. **追问：@Cacheable 与 @CachePut 有什么区别？**  
   @Cacheable 会先查缓存，有则直接返回，没有则执行方法并缓存结果；@CachePut 每次都执行方法，并把结果更新到缓存。  
5. **追问：如何保证缓存一致性？**  
   常见策略：Cache-Aside（旁路缓存），先更新数据库再删除缓存；或使用消息队列保证最终一致性。  
---
## 29. Spring 中如何实现异步调用？
**15秒简答**  
通过 @Async 注解方法，在启动类上 @EnableAsync，Spring 会使用线程池异步执行该方法；可以自定义线程池配置线程数、队列等。  
**3分钟详答**  
Spring 对异步方法的支持：  
1. 在启动类或配置类上 @EnableAsync。  
2. 在需要异步执行的方法上标注 @Async。  
3. 可以通过实现 AsyncConfigurer 自定义线程池和异常处理器。  
适用于日志记录、发送邮件、调用外部接口等耗时但不影响主流程的场景。  
**追问**
4. **追问：@Async 方法有返回值怎么办？**  
   可以返回 `Future` 或 `CompletableFuture`，调用方通过 Future 获取结果。  
5. **追问：异步方法抛出异常如何处理？**  
   默认不会向外传播，可以通过实现 AsyncUncaughtExceptionHandler 处理异常，或使用 `CompletableFuture` 的 exceptionally 方法。  
---
## 30. 你在项目中遇到过哪些 Spring 相关的坑？
**15秒简答**  
常见坑：事务失效、循环依赖、Bean 作用域错误、AOP 自调用失效、多线程事务问题、配置覆盖等，需要结合源码和日志分析定位。  
**3分钟详答**  
结合前面内容，可以举几个例子：  
- 事务失效：自调用、异常类型不匹配、方法非 public 等。  
- 循环依赖：构造器注入导致的循环依赖无法自动解决，需要改为 setter 注入或重构依赖。  
- 作用域错误：在 singleton Bean 中注入 prototype Bean，导致始终使用同一个实例，需要通过代理或 ObjectFactory 每次获取。  
- AOP 自调用：同一类中调用方法不走代理，需要拆分 Bean 或通过 AopContext 获取代理。  
这些问题都可以通过理解 Spring 的生命周期、代理机制和容器行为来避免。  
**追问**
1. **追问：你是如何定位这些问题的？**  
   通过 Debug 查看代理对象类型、查看事务管理器是否生效、打印三级缓存状态、查看源码中的条件判断等。  
2. **追问：有没有读过 Spring 源码？对哪部分比较熟？**  
   可以结合自己情况回答，例如看过 IOC 容器初始化流程、Bean 生命周期、AOP 代理创建等，并结合前面回答举例说明。  
---
以上 30 个问题基本覆盖后端实习面试中 Spring 相关的高频考点，建议重点掌握 IOC/AOP、Bean 生命周期、事务传播与失效场景、Spring MVC 流程以及 Spring Boot 自动配置原理，并结合自己项目经历准备一些实战例子。
