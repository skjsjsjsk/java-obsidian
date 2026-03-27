# 一、面向对象与基础语法
## 1. 面向对象有哪些特征？分别说说你对封装、继承、多态的理解。
### 15秒简答
封装：把数据和操作数据的方法绑定在一起，通过访问修饰符控制外部访问，隐藏内部实现细节。  
继承：子类继承父类的属性和方法，实现代码复用，Java 只支持单继承。  
多态：同一操作在不同对象上有不同表现，通过方法重写（运行时多态）和方法重载（编译时多态）实现。  
抽象：通过抽象类/接口定义抽象概念，只关注“做什么”不关心“怎么做”。
### 3分钟详答
面向对象一般有四大特性：封装、继承、多态、抽象。
- **封装**：将成员变量设为 private，提供 public 的 getter/setter 或业务方法，让外部只能通过受限的接口访问对象内部状态。好处是提高安全性和可维护性，例如 BankAccount 类把余额设为 private，只通过 deposit/withdraw 操作余额。
- **继承**：使用 `extends` 让子类获得父类的属性和方法，子类可以扩展或重写。Java 类是单继承，接口可以多继承。继承解决代码复用问题，但过深的继承会降低可维护性。
- **多态**：
  - 编译时多态：方法重载（Overload），同一个类中方法名相同、参数列表不同。
  - 运行时多态：方法重写（Override），子类重写父类方法，通过父类引用指向子类对象，调用的是子类实现。
- **抽象**：通过抽象类和接口定义抽象层，只描述功能不实现细节，让调用者面向抽象编程，提高扩展性。
在面试中，可以先说“四大特征”，再逐条展开，并适当举一个简单例子（动物类 Animal、子类 Dog 重写 eat 方法等）。
### 可能追问
1. 多态在代码中一般怎么体现？  
   通过“父类引用指向子类对象，调用重写方法”体现，例如 `Animal a = new Dog(); a.eat();` 实际执行 Dog 的 eat。
2. 重载和重写的区别？  
   重载：同类中，方法名相同，参数列表不同，与返回类型无关，编译时多态。  
   重写：子类对父类方法重新实现，方法签名相同，返回类型相同或协变，运行时多态。
---
## 2. 重载（Overload）和重写（Override）有什么区别？
### 15秒简答
重载：同一个类中，方法名相同，参数列表不同，与返回类型无关，编译时多态。  
重写：子类重新实现父类的方法，方法名、参数列表相同，返回类型相同或协变，访问权限不能更严格，运行时多态。
### 3分钟详答
- **方法重载（Overloading）**  
  - 定义：一个类中定义多个同名方法，但参数个数或类型不同。  
  - 规则：方法名必须相同；参数列表必须不同；返回类型可以相同也可以不同。  
  - 本质：编译时多态，编译器根据参数类型选择具体方法。
- **方法重写（Overriding）**  
  - 定义：子类重新定义父类中已定义的方法。  
  - 规则：
    - 方法名、参数列表必须相同。
    - 返回类型相同，或是子类型（协变返回类型）。
    - 访问权限不能比父类更严格（可以更宽松）。
    - 不能抛出新的或更宽泛的受检异常。  
  - 本质：运行时多态，通过父类引用调用实际子类对象的方法。
在回答时，可以顺带提一句：构造方法可以重载，但不能重写（构造方法不属于子类继承范围）。
### 可能追问
1. 只有返回类型不同能构成重载吗？  
   不能。重载与返回类型无关，只看方法名和参数列表。
2. 父类的静态方法能被子类重写吗？  
   不能。静态方法属于类，子类可以定义同名静态方法，但这属于“隐藏”而不是重写，推荐使用“类名.方法名”调用。
---
## 3. 抽象类和接口有什么区别？什么时候用抽象类，什么时候用接口？
### 15秒简答
抽象类用 `abstract class` 定义，可以有构造方法、普通成员变量和非抽象方法，用于抽取子类共性，是一种 is-a 关系。  
接口用 `interface` 定义，JDK8 前只能有抽象方法和常量，JDK8 后支持 default/static 方法，表示一种能力或契约，是 has-a 关系。  
一个类可以实现多个接口，但只能继承一个抽象类。
### 3分钟详答
**抽象类：**
- 使用 `abstract` 修饰类和方法。
- 不能实例化，用于被继承。
- 可以有成员变量、构造方法、普通方法和抽象方法。
- 子类必须实现所有抽象方法，否则自身也是抽象类。
**接口：**
- 使用 `interface` 定义。
- 成员变量默认是 `public static final`，方法默认是 `public abstract`（JDK8 之前）。
- JDK8 开始支持 `default` 默认方法和 `static` 方法，允许在接口中提供实现。
- 一个类可以实现多个接口，实现多继承的效果。
**使用场景：**
- 抽象类：强调“是什么”的家族概念，多个子类有共同属性和方法实现时，用抽象类做模板（如 Animal 抽象类，包含 age/name 等字段和通用 sleep 方法）。
- 接口：强调“能做什么”的能力，如 Comparable、Serializable，用于定义行为契约，解耦调用者与实现。
### 可能追问
1. 接口能不能有构造方法？  
   不能。接口不能实例化，也不允许定义构造方法。
2. JDK8 之后接口有了默认方法，那抽象类还有存在的意义吗？  
   有。抽象类可以持有状态（成员变量）和提供部分实现，而接口仍然侧重行为契约；抽象类适合“模板方法模式”，接口适合定义多种能力。
---
## 4. Java 中有哪些访问权限修饰符？各自的可访问范围是什么？
### 15秒简答
- `public`：对所有类可见。
- `protected`：对本包和所有子类可见。
- 默认（无修饰符）：仅对本包可见。
- `private`：仅对本类可见。
### 3分钟详答
Java 使用访问权限修饰符控制类、成员变量、方法等的可见性：
- **public**：最大权限，任何类都可以访问。
- **protected**：同包内的类可访问，不同包的子类也可访问，常用于被子类继承的成员。
- **默认（包访问权限）**：不写修饰符，同包内的类可以访问，不同包不可见。
- **private**：最小权限，只能在当前类内部访问，用于封装内部实现细节。
在面试中，可以画一个简单的“包内/包子类/其他类”的表格来说明范围，或者给出一个示例类，展示不同修饰符成员的访问情况。
### 可能追问
1. 外部类能用哪些修饰符？  
   外部类可以使用 public 或默认权限，不能用 private/protected，因为外部类本身不是成员。
2. 构造方法可以用 private 吗？有什么用途？  
   可以，用于单例模式或工具类，限制外部直接创建实例，要求通过静态方法获取实例。
---
## 5. static 关键字有哪些用途？
### 15秒简答
`static` 可以修饰成员变量、成员方法、代码块和内部类。  
被 static 修饰的成员属于类，而不是实例，可以通过类名直接访问；静态代码块在类加载时执行，常做初始化工作。
### 3分钟详答
- **静态变量（类变量）**：属于类，所有实例共享一份，常用作全局常量或共享状态。
- **静态方法**：属于类，不能直接访问非静态成员（因为没有 this），常用作工具方法，如 `Math.sqrt`。
- **静态代码块**：类加载时执行一次，适合做复杂的静态初始化。
- **静态内部类**：不依赖外部类实例，可以直接创建，常用于构建器模式等。
注意：静态方法不能重写，但可以被继承；子类可以定义同签名的静态方法，这叫“隐藏”而不是重写。
### 可能追问
1. 静态方法里能不能使用 this 或 super？  
   不能，静态方法属于类，没有当前对象实例，因此不能使用 this 或 super。
2. 静态代码块和构造代码块（构造块）有什么区别？  
   静态代码块在类加载时执行一次；构造代码块在每次创建对象时执行一次，早于构造方法。
---
## 6. final 关键字有哪些用法？final、finally、finalize 有什么区别？
### 15秒简答
`final` 修饰类表示不能继承；修饰方法表示不能重写；修饰变量表示只能赋值一次（常量）。  
`finally` 是异常处理的一部分，`try-catch-finally` 中 finally 块一般总会执行。  
`finalize` 是 Object 的方法，在 GC 回收对象前调用，不推荐主动使用。
### 3分钟详答
- **final 类**：不能被继承，如 String、包装类。
- **final 方法**：不能被子类重写，常见于框架中防止子类改变关键逻辑。
- **final 变量**：只能赋值一次，既可以是编译时常量，也可以在构造方法/代码块中赋值。
**finally**：  
- 在 `try` 或 `catch` 块执行之后，finally 块中的代码通常会执行，用于释放资源（关闭流、连接等）。即使 try 或 catch 中有 return，也会先执行 finally。
**finalize**：  
- 是 `java.lang.Object` 的方法，在垃圾回收器认为对象不可达后，决定回收该对象前调用，用于释放本地资源等。  
- Java 9 已标记为 deprecated，不推荐依赖 finalize 做资源释放，应使用 try-with-resources 或显式 close。
### 可能追问
1. finally 块会不会不执行？  
   一般会执行，但如果在 try/catch 中调用 `System.exit()`，或 JVM 崩溃，finally 可能不执行。
2. final 修饰的引用变量，其指向的对象内容可变吗？  
   可变。final 只保证引用本身不变，对象内部字段仍可修改。
---
## 7. Java 是值传递还是引用传递？怎么理解？
### 15秒简答
Java 只有值传递。对于基本类型，传递的是值的副本；对于引用类型，传递的是对象引用的副本。方法内修改形参不会影响实参的引用，但可以通过引用修改对象内容。
### 3分钟详答
- **值传递**：方法调用时，实参的值被复制给形参，形参与实参相互独立。
- Java 的做法：
  - 基本类型：直接复制数值。
  - 引用类型：复制的是引用（地址），形参与实参指向同一个对象。
因此：
- 在方法内修改形参指向的对象（如 `arr[0] = 10`），外部会看到变化。
- 但在方法内让形参指向新对象，不会影响实参的引用。
常见错误说法：  
“基本类型是值传递，引用类型是引用传递”，这是不准确的；正确的表述是“Java 统一是值传递，引用类型传递的是引用的值”。
### 可能追问
1. 下面的代码输出是什么？
```java
public static void main(String[] args) {
    StringBuilder sb = new StringBuilder("hello");
    change(sb);
    System.out.println(sb);
}
public static void change(StringBuilder builder) {
    builder.append(" world");
    builder = new StringBuilder("new");
}
```
输出：`hello world`。方法内通过引用修改了对象内容，但重新赋值 builder 不影响原引用。
---
## 8. Java 中的基本数据类型有哪些？对应的包装类是什么？
### 15秒简答
8 种基本类型：byte、short、int、long、float、double、char、boolean。  
对应的包装类：Byte、Short、Integer、Long、Float、Double、Character、Boolean。
### 3分钟详答
Java 有两类类型：基本类型和引用类型。  
基本类型直接存储值，放在栈中；包装类是引用类型，对象放在堆中。  
自动装箱/拆箱是在基本类型和对应包装类之间自动转换的过程。
常见考点：
- 整数默认是 int，浮点默认是 double。
- long 型常量后加 L 或 l，float 型常量后加 F 或 f。
- 包装类提供了常用方法，如 `Integer.parseInt`、`Double.isNaN` 等。
### 可能追问
1. Integer a = 127, Integer b = 127，a == b 结果是什么？128 呢？  
   在 -128~127 之间，IntegerCache 会缓存对象，`a == b` 为 true；超出范围则为 false。
---
## 9. == 和 equals() 有什么区别？
### 15秒简答
`==`：对于基本类型比较值，对于引用类型比较引用地址。  
`equals()`：默认也是比较引用，但很多类（如 String、Integer）重写了 equals，用于比较内容是否相等。
### 3分钟详答
- `==`：
  - 基本类型：比较数值是否相等。
  - 引用类型：比较两个引用是否指向同一个对象。
- `equals()`：
  - Object 类中的 equals 等价于 `==`。
  - 常见类重写 equals，如 String 先比较地址，再逐个比较字符。
在面试中，可以给出：
```java
String s1 = new String("abc");
String s2 = new String("abc");
System.out.println(s1 == s2);       // false
System.out.println(s1.equals(s2)); // true
```
### 可能追问
1. 重写 equals 时为什么要重写 hashCode？  
   规定：equals 相等的对象，hashCode 必须相同。否则在 HashMap/HashSet 中会出现逻辑错误。
---
## 10. String、StringBuilder、StringBuffer 有什么区别？
### 15秒简答
String 是不可变字符串，每次拼接都会产生新对象；  
StringBuilder 是可变字符序列，非线程安全，性能好；  
StringBuffer 也是可变字符序列，线程安全，性能稍低。
### 3分钟详答
- **String**：
  - 底层使用 final 修饰的 char[]（JDK9 后是 byte[] + coder）。
  - 不可变，适合作为常量、HashMap 的 key 等。
- **StringBuilder**：
  - 继承 AbstractStringBuilder，内部维护可扩容的数组。
  - 常用 `append`、`insert` 等方法，在单线程环境中拼接字符串性能最好。
- **StringBuffer**：
  - 与 StringBuilder 类似，但方法大多使用 `synchronized` 修饰，线程安全。
使用建议：
- 少量、基本不变的字符串：String。
- 单线程大量拼接：StringBuilder。
- 多线程共享同一个字符串缓冲区：StringBuffer。
### 可能追问
1. 为什么 String 设计成不可变？  
   为了支持字符串常量池、缓存 hashCode、保证线程安全和安全性（如类加载、权限检查）。
---
## 11. 什么是自动装箱和拆箱？有什么注意点？
### 15秒简答
自动装箱：基本类型自动转换为对应的包装类（如 int -> Integer）。  
自动拆箱：包装类自动转换为基本类型（如 Integer -> int）。  
编译器在编译时插入 `Integer.valueOf` 和 `intValue` 调用。
### 3分钟详答
- 装箱示例：`Integer a = 10;` 实际编译为 `Integer a = Integer.valueOf(10);`。
- 拆箱示例：`int b = a;` 实际编译为 `int b = a.intValue();`。
注意点：
- 频繁拆箱/装箱会产生额外对象，可能影响性能。
- IntegerCache 缓存了 -128~127 的对象，在此范围内 `==` 比较可能“表现异常”。
### 可能追问
1. Integer a = null; int b = a; 会发生什么？  
   运行时抛出 NullPointerException，因为拆箱会调用 `a.intValue()`。
---
## 12. 什么是构造方法？构造方法可以重载吗？可以被重写吗？
### 15秒简答
构造方法用于初始化对象，方法名与类名相同，没有返回类型。  
构造方法可以重载（一个类可以有多个构造方法），但不能重写（不属于子类继承范围）。
### 3分钟详答
- 如果不显式定义构造方法，编译器会生成默认无参构造方法；一旦定义任意构造方法，默认构造方法不再生成。
- 构造方法重载：提供多种初始化方式，如无参、全参、部分参数构造方法。
- 构造方法不能被继承，因此不能重写；但子类构造方法默认调用父类构造方法（super()）。
### 可能追问
1. 子类构造方法一定会调用父类构造方法吗？  
   是的，如果子类构造方法没有显式调用 `super(...)`，编译器会默认调用 `super()`，要求父类有无参构造方法。
---
# 二、异常处理
## 13. 简述 Java 异常体系，Error 和 Exception 有什么区别？
### 15秒简答
Throwable 是所有错误/异常的父类，子类有 Error 和 Exception。  
Error 表示 JVM 级别严重错误，如 OutOfMemoryError，程序一般无法处理。  
Exception 表示程序需要捕获/声明的异常，分为受检异常（Checked）和运行时异常（RuntimeException）。
### 3分钟详答
- **Throwable**：所有错误和异常的超类。
- **Error**：JVM 内部错误或资源耗尽，如 VirtualMachineError、OutOfMemoryError，程序不应捕获。
- **Exception**：
  - 受检异常：编译期强制处理（IOException、SQLException）。
  - 运行时异常：RuntimeException 及其子类，如 NullPointerException、IndexOutOfBoundsException，编译期不强制处理。
在面试中，可以画出简单继承链：  
`Object -> Throwable -> Error / Exception`，再举例说明常见 Error 和 Exception。
### 可能追问
1. 列举 5 个常见的 RuntimeException？  
   NullPointerException、IndexOutOfBoundsException、ClassCastException、NumberFormatException、IllegalArgumentException。
---
## 14. throw 和 throws 有什么区别？
### 15秒简答
`throw` 用在方法体内，抛出一个具体的异常实例。  
`throws` 用在方法签名上，声明该方法可能抛出的异常类型，由调用者处理。
### 3分钟详答
- **throw**：
  - 语法：`throw new SomeException("msg");`
  - 执行到 throw 时，方法立即终止，异常向外传播。
- **throws**：
  - 语法：`void method() throws IOException, SQLException { ... }`
  - 表示方法可能抛出这些异常，调用者需要捕获或继续声明抛出。
常见用法：  
在方法内部检查到非法参数时，使用 `throw` 抛出 IllegalArgumentException；在方法签名上使用 `throws` 声明受检异常。
### 可能追问
1. 如果 throw 了一个受检异常，方法签名必须怎么处理？  
   要么用 try-catch 捕获，要么在方法签名上用 throws 声明，否则编译错误。
---
## 15. try-catch-finally 的执行顺序是怎样的？finally 一定会执行吗？
### 15秒简答
try 块中代码发生异常后，立即进入匹配的 catch 块；无论是否发生异常，finally 块一般都会执行。  
在 try/catch 中有 return 时，finally 也会执行，且如果 finally 中有 return，会覆盖之前的返回值。
### 3分钟详答
执行顺序：
1. 执行 try 块，遇到异常则转入对应 catch。
2. 执行 catch 块（如果有）。
3. 在返回前执行 finally 块。
特殊情况：
- 如果在 try/catch 中调用 `System.exit()`，JVM 直接退出，finally 可能不执行。
- 如果 finally 中有 return，会覆盖 try/catch 中的返回值，不推荐这样写。
### 可能追问
1. 下面代码的输出是什么？
```java
public static int test() {
    try {
        return 1;
    } finally {
        return 2;
    }
}
```
结果为 2，因为 finally 中的 return 会覆盖 try 中的 return。
---
## 16. 什么是自定义异常？如何实现？
### 15秒简答
自定义异常是根据业务需求定义的异常类，一般继承 Exception 或 RuntimeException。  
实现方式：定义类继承 Exception/RuntimeException，提供构造方法，使用 throw 抛出。
### 3分钟详答
- 步骤：
  1. 定义类继承 Exception 或 RuntimeException。
  2. 提供无参构造和带 message 的构造，调用 super(message)。
  3. 在业务代码中根据条件 `throw new CustomException("msg");`。
- 如果继承 Exception，则为受检异常，编译器强制处理；若继承 RuntimeException，则为运行时异常，使用更灵活。
### 可能追问
1. 你在项目中自定义过哪些异常？  
   常见如 UserNotFoundException、BizException（业务异常）、ParamInvalidException 等，用于封装业务错误信息。
---
# 三、IO 流与序列化
## 17. Java 中 IO 流分为哪几类？字节流和字符流有什么区别？
### 15秒简答
按方向分：输入流和输出流。  
按单位分：字节流（InputStream/OutputStream）和字符流（Reader/Writer）。  
字节流处理二进制数据，字符流处理文本数据，自动处理编码转换。
### 3分钟详答
- **字节流**：
  - 基类：InputStream/OutputStream。
  - 适合处理图片、音频、视频等二进制文件。
  - 常见实现：FileInputStream、FileOutputStream、BufferedInputStream 等。
- **字符流**：
  - 基类：Reader/Writer。
  - 以字符为单位读写文本文件，内部使用编码转换。
  - 常见实现：FileReader、FileWriter、BufferedReader、BufferedWriter。
选择原则：  
处理文本优先字符流，处理二进制必须使用字节流。
### 可能追问
1. 为什么有字节流还需要字符流？  
   字节流直接操作字节，处理文本时需要开发者自己处理编码，容易乱码；字符流封装了编码转换，更方便处理文本。
---
## 18. 什么是 Java 序列化？如何实现？
### 15秒简答
序列化是将对象转换为字节序列，便于存储到文件或进行网络传输；反序列化是将字节序列恢复为对象。  
实现方式：让类实现 `Serializable` 接口，使用 ObjectOutputStream/ObjectInputStream 进行序列化与反序列化。
### 3分钟详答
- `Serializable` 接口是一个标记接口，没有方法，表示对象可序列化。
- 步骤：
  1. 类实现 `Serializable`。
  2. 使用 `ObjectOutputStream.writeObject(obj)` 写对象。
  3. 使用 `ObjectInputStream.readObject()` 读对象。
- `transient` 关键字修饰的成员不参与序列化。
### 可能追问
1. 序列化 ID（serialVersionUID）有什么作用？  
   用于版本控制，反序列化时若 UID 不一致，会抛 InvalidClassException。
---
# 四、反射、泛型与注解
## 19. 什么是 Java 反射？有什么作用和缺点？
### 15秒简答
反射是在运行时动态获取类的信息（字段、方法、构造器等），并动态调用方法、创建对象。  
优点：提高灵活性，是框架的核心机制；缺点：性能较低，破坏封装，安全风险较大。
### 3分钟详答
- 核心类：Class、Field、Method、Constructor。
- 获取 Class 对象的三种方式：
  - `类名.class`
  - `对象.getClass()`
  - `Class.forName("全限定名")`。
- 通过 Class 对象可以：
  - 创建实例：`clazz.newInstance()` 或 `clazz.getDeclaredConstructor().newInstance()`。
  - 获取字段并修改：`Field.setAccessible(true)`。
  - 调用方法：`method.invoke(obj, args)`。
应用场景：框架（Spring、MyBatis）、ORM 映射、动态代理等。
### 可能追问
1. 反射能访问私有成员吗？  
   可以，通过 `setAccessible(true)` 暴力访问，但破坏封装，有安全限制。
---
## 20. Java 泛型的作用是什么？什么是类型擦除？
### 15秒简答
泛型让类/接口/方法可以定义类型参数，提高类型安全和代码复用。  
类型擦除是指编译器在编译时将泛型信息擦除，运行时只保留原始类型，泛型主要在编译期起作用。
### 3分钟详答
- 泛型作用：
  - 编译时类型检查，避免 ClassCastException。
  - 减少强制类型转换。
- 类型擦除：
  - 泛型代码编译后，`List<String>` 和 `List<Integer>` 在字节码层面都是 List，泛型类型信息被擦除。
  - 因此运行时无法通过泛型类型做反射等操作（除非使用特殊手段）。
### 可能追问
1. 泛型通配符 `? extends T` 和 `? super T` 有什么区别？  
   `? extends T`：上界通配符，适合读取，不适合写入。  
   `? super T`：下界通配符，适合写入，不适合读取。
---
## 21. 什么是注解？元注解有哪些？
### 15秒简答
注解是代码里的特殊标记，用于为代码提供元数据。  
元注解包括 @Retention、@Target、@Documented、@Inherited 等，用于定义注解的保留策略和使用位置。
### 3分钟详答
- 注解本身不影响程序运行，但可以通过反射读取并据此做处理。
- 常见内置注解：@Override、@Deprecated、@SuppressWarnings 等。
- 元注解：
  - @Retention：保留策略（SOURCE/CLASS/RUNTIME）。
  - @Target：使用位置（类、方法、字段等）。
  - @Documented：生成文档时包含该注解。
  - @Inherited：允许子类继承父类注解。
### 可能追问
1. 你自定义过注解吗？怎么用？  
   使用 `@interface` 定义注解，配合元注解，在运行时通过反射获取注解信息，实现权限校验、日志记录等功能。
---
## 22. 枚举（enum）有什么作用？和定义常量相比有什么优势？
### 15秒简答
枚举用于定义一组固定的常量实例，如季节、订单状态等。  
相比 `public static final` 常量，枚举类型安全，可以有字段和方法，switch 支持枚举，且能防止非法值。
### 3分钟详答
- 枚举本质是一个继承 `java.lang.Enum` 的类，枚举常量是此类的实例。
- 优点：
  - 编译期类型安全，只能使用已定义的枚举常量。
  - 可以添加字段、方法和构造方法。
  - 天然单例，可用于实现单例模式。
  - switch-case 支持枚举。
### 可能追问
1. 枚举可以实现接口吗？  
   可以，每个枚举实例可以单独实现接口方法，实现类似策略模式的效果。
---
# 五、其他高频基础题
## 23. Java 中创建对象有哪几种方式？
### 15秒简答
常见的有：使用 new 关键字、反射（Class.forName + newInstance）、克隆（实现 Cloneable）、反序列化（ObjectInputStream）。
### 3分钟详答
- new：最常见，调用构造方法。
- 反射：`Class.forName("ClassName").newInstance()` 或通过 Constructor 创建对象。
- 克隆：实现 Cloneable 接口，重写 `clone()` 方法。
- 反序列化：从字节流恢复对象。
### 可能追问
1. 深拷贝和浅拷贝有什么区别？  
   浅拷贝：只复制对象本身和基本类型字段，引用类型字段仍指向原对象。  
   深拷贝：递归复制所有引用对象，修改副本不影响原对象。
---
## 24. 什么是深拷贝和浅拷贝？如何实现深拷贝？
### 15秒简答
浅拷贝：复制对象本身，内部引用类型字段仍指向原对象。  
深拷贝：递归复制所有引用对象，副本与原对象完全独立。  
实现方式：重写 clone 方法递归克隆，或使用序列化/反序列化。
### 3分钟详答
- Object 的 clone 方法默认是浅拷贝。
- 深拷贝实现：
  - 让对象实现 Cloneable，在 clone 方法中手动克隆每个引用类型字段。
  - 或者将对象序列化再反序列化，得到一个全新的对象。
### 可能追问
1. 序列化实现深拷贝有什么注意点？  
   所有引用类型都必须实现 Serializable，否则会抛 NotSerializableException。
---
## 25. Java 中的值传递/引用传递你之前理解有误区吗？
### 15秒简答
很多人误以为“基本类型是值传递，引用类型是引用传递”，但 Java 实际上只有值传递。引用类型传递的是引用的副本，修改形参引用不会影响实参引用，但可以修改对象内容。
### 3分钟详答
- 基本类型：传递值的副本，形参修改不影响实参。
- 引用类型：传递引用的副本，两个引用指向同一个对象，通过引用修改对象内容会影响外部，但修改形参指向的对象不影响实参引用。
在面试中，可以用一个数组或 StringBuilder 示例说明“看起来像引用传递，实际是值传递”的情况。
### 可能追问
1. 你能写一个示例证明 Java 是值传递吗？  
   写一个方法，接收一个引用参数并让其指向新对象，方法外部引用不变，证明传递的是引用的副本。
---
## 26. 访问权限修饰符的范围，你能画一张表吗？
### 15秒简答
可以按“本类、同包、包子类、其他类”四个维度理解：  
- public：对所有类可见。  
- protected：本类 + 同包 + 子类可见。  
- 默认：本类 + 同包可见。  
- private：仅本类可见。
### 3分钟详答
可以画一个简单表格：
| 修饰符    | 本类 | 同包 | 包子类 | 其他 |
|-----------|------|------|--------|------|
| public    | ✓    | ✓    | ✓      | ✓    |
| protected | ✓    | ✓    | ✓      | ×    |
| 默认      | ✓    | ✓    | ×      | ×    |
| private   | ✓    | ×    | ×      | ×    |
面试时，可以直接口述这张表，或者用示例类说明不同修饰符成员的访问范围。
### 可能追问
1. 你在项目里怎么选择访问修饰符？  
   字段一般 private，对外暴露 public 方法；模板方法或需要子类重写的用 protected；包内工具方法用默认。
---
## 27. 你如何理解 Java 的“一次编写，到处运行”？
### 15秒简答
Java 源代码编译为字节码，运行在 JVM 上；不同平台有对应的 JVM 实现，屏蔽底层差异，因此同一份字节码可以在不同平台上运行。
### 3分钟详答
- 编译期：Java 源码编译为 .class 字节码。
- 运行期：JVM 将字节码解释/编译为本地机器码执行。
- 各主流平台（Windows、Linux、macOS）都有对应的 JVM，使得同一份程序可以跨平台运行。
### 可能追问
1. JVM、JRE、JDK 三者有什么区别？  
   JVM：虚拟机，执行字节码。  
   JRE：JVM + 核心类库，运行环境。  
   JDK：JRE + 工具（javac、javadoc 等），开发环境。
---
## 28. switch 语句能作用于哪些数据类型？
### 15秒简答
JDK7 前：byte、short、char、int 及其包装类，以及枚举。  
JDK7 开始：支持 String。  
long、float、double、boolean 不能用于 switch。
### 3分钟详答
- switch 表达式类型：
  - byte、short、char、int 及对应包装类。
  - 枚举类型。
  - String（JDK7+）。
- 不支持 long、float、double、boolean。
在面试中，可以举一个 String 的 switch 示例，说明 JDK7 的增强。
### 可能追问
1. switch 是否能作用在 String 上？  
   JDK7 开始可以，编译器将 String 转换为 hashCode + if-else 实现。
---
## 29. 你在项目中是如何避免空指针异常（NullPointerException）的？
### 15秒简答
使用前做非空检查，使用 Optional（JDK8+），避免链式调用空对象，尽量使用“常量.equals(变量)”等写法。
### 3分钟详答
常见做法：
- 显式判空：`if (obj != null) { ... }`。
- 使用 `Objects.requireNonNull(obj, "msg")` 快速失败。
- 使用 Optional：`Optional.ofNullable(obj).ifPresent(...)`.
- 字符串比较：`"constant".equals(variable)` 避免变量为 null。
- 使用 IDE 的 @Nullable/@NotNull 注解做静态分析。
### 可能追问
1. Optional 能完全替代 null 吗？  
   不能。Optional 适合作为返回值类型，不建议在字段、方法参数中使用。
---
## 30. 你对“Java 面向对象”的理解，除了语法，还有哪些体会？
### 15秒简答
面向对象不仅是语法，更是一种建模方式：通过抽象、封装、继承、多态将现实世界映射为类和对象，降低耦合，提高复用性和可维护性。
### 3分钟详答
可以从几个角度回答：
- 设计层面：合理划分职责，单一职责、开闭原则、依赖倒置等原则都围绕面向对象展开。
- 代码层面：通过接口/抽象类解耦，组合优于继承，避免过深的继承层次。
- 工程层面：面向对象使得代码更易扩展、易测试，为框架和模式提供基础。
### 可能追问
1. 你在项目中遇到过哪些“糟糕的面向对象设计”？  
   过深的继承链、大类、大类中大量静态方法等，都可以作为例子，说明你如何重构为更合理的结构。
