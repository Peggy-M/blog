# Java 基础面试题

## Java 中的序列化与反序列化

- **为什么会有序列化与反序列化这种技术**

序列化这种技术的诞生之初就是为了 **解决对象或数据结构的存储与传输** ，将对象或数据结构 **通过编码的方式存储为二进制或 文件的形式** ，而反序列化则是对之间序列化之后的数据的一种反向解码还原的过程。

- **在 Java 当中序列化与反序列化时如何体现的**

由于对于序列化而言采用的编码方式不同，其序列化的形式也存在差异，比如为了更好的节省空间采用二进制序列化，比如为了有较强的可读性使用文件序列化为 JSON，XML，YAML 格式。

在 Java 当中继承 **Serializable** 接口，通过 **ObjectOutputStream** 就可以序列化为二进制对象文件，主要用户对中间的一些对象结果存储。

- **有哪些实际开发应用场景**

假设有这样的一个场景，比如开发一个电商系统，在用户完成下单之后需要分别进行:

1. 将订单数据持久化到本地文件作为备份,方便日后对于时间跨度较长的订单详细信息进行追溯
2. 将订单发送到消息队列供其他服务处理，比如支付流程、库存管理、物流配送、订单分析等
3. 把订单数据存入Redis缓存加速查询，对近期的数据进行缓存命中

## Java 中的 Exception 与 Error 有什么区别

- **为什么会有 Exception  与 Error 两种**

首先对于 Exception  和 Error 其都是继承于 Throwable 该类，都是配合**try-catch** 一起使用，其 **核心目的就是为了在程序运行当中对于一些异常进行抛出**，进行错误的定位。当然也有一些特殊的情况，比如对于一个判断预检，直接通过 **throw** 的方式抛出。

- **Excetion 与 Error 的详细区别**

虽然说 Excetion 与 Error  都是继承与 Throwable 但是本身两者在用途方面还是存在很大的区别的。对于 **Exception ** 而言，主要是用于来处理应用层面的程序代码，这种错误通常是可以恢复，不影响 JVM 服务的运行，比如 IOException, SQLException，而 **Error** 的错误来自于 JVM 或底层系统资源出现的错误，这种错误一旦报出，会导致整个 JVM 服务宕机停止运行，比如最常见的  OutOfMemoryError, StackOverflowError 等错误。

- **有哪些实际的开发和应用场景**

对于 Error 实际的应用场景其实并不多，其主要是为了方便分析定位错误，而 **Excetion ** 这种类型的错误，一般用于对一些异常的捕获处理比如 **I/O失败、数据库访问错误、类加载失败** 或者进行继承自定义一些边界检查的异常信息。

## Java 中的参数传递是按值还是按引用？

这与 C++ 存在一定的区别，因为 C++ 的参数传递是支持引用传递和指针传递的，而 Java 当中所有的传递都 **按照值传递的**，但是根据其传递的是基本类型还是对象类型表现会有所不同。

在基本类型传递过程当中，如果在方法内修改参数的值，由于参数是一份拷贝的变量值副本，其并不会影响到后续其他方法的调用。但对于对象来说，是有些许不同的，对于对象作为参数而言，由于对象拷贝的是一份引用的副本，所以在修改对象属性值的时候，其由于这个副本指向的还是原来对象的本身，所以属性值的修改会影响到原理对象本身的值。但如果只是修改的是参数副本的引用其并不会对原本的对象引用造成影响。

这一点与 C++ 通过指针作为参数传递直接修改指针的指向是存在差异的。

- 基本类型的传递

~~~ java
void modify(int x) {
    x = 10; // 只修改局部副本
}

public static void main(String[] args) {
    int num = 5;
    modify(num);
    System.out.println(num); // 输出 5（原值不变）
}
~~~

- 对象类型的传递

~~~ java
class Person {
    String name;
    Person(String name) { this.name = name; }
}

void modifyObject(Person p) {
    p.name = "Alice";  // 修改的是原对象
    p = new Person("Bob"); // 只改变局部引用
}

public static void main(String[] args) {
    Person person = new Person("Charlie");
    modifyObject(person);
    System.out.println(person.name); // 输出 "Alice"（不是Bob）
}
~~~

## 什么是 Java 的多态性

对于面向对象的编程语言都是具有 **封装、继承、多态** 最基本的三大特性的，这三大特性使得面向对象的这种语言变得更加的灵活，更容易抽象成一个个可以管理的模块。而多态简单描述就是 **同一行为具有的多种不同的表现形式或形态能力** ，但这一特性对于 C 语言这种低级语言而言，本身是没有这种特性的，要实现类似的模拟特性就需要通过结构体进行封装。

- **在 Java 层面如何体现这种多态性的**

先给结论，Java 层面的多态依赖于接口的实现以及继承关系，同一行为的不同具体的表现形式，在实际开放当中也就是通过一个抽象接口定义统一的一套方法规则，针对于不同的具体业务来实现其共同的抽象接口，比如在业务当中需要进行模型的训练，但对于不同的模型其训练的细节可能会有所不同，可以通过扩展的具体实现类去实现共同的一套接口。

还比如不同数据库厂家针对于 JDBC 的实现就是一种经典的多态性体现，官方提供统一的实现接口，厂家针对于接口开发自己的数据库驱动。

~~~ java
// 同一行为：数据库连接
Connection conn = DriverManager.getConnection(url, user, pass);

// 不同表现（根据驱动不同）：
// MySQL的实现：com.mysql.cj.jdbc.ConnectionImpl
// Oracle的实现：oracle.jdbc.driver.T4CConnection
// PostgreSQL的实现：org.postgresql.jdbc.PgConnection
~~~

~~~ java
//在指向查询语句的时候
public void queryData(Connection conn) throws SQLException {
    // 多态的Statement
    Statement stmt = conn.createStatement(); 
    
    // 多态的ResultSet
    ResultSet rs = stmt.executeQuery("SELECT * FROM users");
    
    while(rs.next()) {
        // 多态的getString方法（不同驱动实现不同）
        System.out.println(rs.getString("username"));
    }
}
// 同一段代码可以适用于任何JDBC兼容的数据库
~~~

## 为什么 Java 不支持多继承

C++ 诞生于80年 Java 比C++ 晚了10年，当初设计 Java 语言的时候詹姆斯高斯林的想法，就是为了对应 C++ 的复杂性，因此引入了许多特性（内存自动管理、单继承），C++ 当中多继承面临一个较为痛疼的 **菱形继承问题**，

- C++ 的菱形继承问题

~~~ c++
class A {
public:
    int value;
    void foo() { cout << "A::foo()" << endl; }
};

class B : public A {};  // B 继承 A
class C : public A {};  // C 继承 A
class D : public B, public C {};  // D 继承 B 和 C
~~~

~~~ c++
// D 会包含 两份 A 的副本（分别来自 B 和 C）,如果直接访问 D 的 value 或 foo()，编译器会报错，因为不知道应该选择 B::A::value 还是 C::A::value。
D d;
d.value = 10;  // 错误：ambiguous access of 'value'
d.foo();       // 错误：ambiguous access of 'foo'
~~~

- 通过虚继承确定指向的实例

~~~ c++
class A {
public:
    int value;
    void foo() { cout << "A::foo()" << endl; }
};

class B : virtual public A {};  // B 虚继承 A
class C : virtual public A {};  // C 虚继承 A
class D : public B, public C {};  // D 只包含一份 A
~~~

~~~ c++
//B 和 C 使用 virtual public A：确保 D 只继承 一份 A，而不是两份。D 的存储结构：D 现在只包含 一个 A 的实例，B 和 C 共享它。
D d;
d.value = 10;  // OK，只有一份 A::value
d.foo();       // OK，调用 A::foo()
~~~

~~~ mermaid
classDiagram
    class A { +int value +foo() }
    class B { }
    class C { }
    class D { }

    B --|> A
    C --|> A
    D --|> B
    D --|> C
~~~

- **为什么接口可以多实现**

  ~~~ java
  //接口 B 定义了 print 方法
  public interface B { void print(); }
  //接口 C 定义了 print 方法
  public interface C { void print(); }
  //接口 D 多继承了 接口 B 与 C
  public interface D extends C, B {  void test(); }
  ~~~

  ~~~ java
  public class AB implements D {
      public void test() {}
      public void print() {}
  }
  ~~~

  可以看到对于接口 D 而言是完全支持多继承的，那么这里就有一个问题了，**为什么类不支持而接口支持呢？**

  类不支持的原因是因为，对于不同的类内部可能拥有相同的方法，多继承就会导致子类无法确定到底调用哪一个方法，而接口却不一样，接口只是提供定义了行为规范，不包含 **具体的实现** 或 **状态**。即便是多继承了，最终的实现是子类完成，所以接口的对象进行调用的时候，最终的指向是明确的。不会出现调用的混乱。

## 面向对象编程与面向过程编程的区别是什么

面向对象更多的想是一条流水线，将一个具体问题进行拆解，用不同的子函数去实现，再通过 **main** 方法作为入口将整个子函数线性串联起来。更多的是适合于数学计算与简单脚本的处理。

而面向对象更多是通过协作解决一个体量比较大的问题，每一个模块独立之间不存在特别强的关联性，最终将整个模块组合拼装起来。比如对于一个Web应用程序来说，定义一个通用的接口，并交给不同的具体业务细节进行实现，最终根据不同的业务模块进行灵活的组合。

## Java 方法重载和方法重写之间的区别是什么

- **为什么会有方法的重载**

重载在平时开发当中经常见到，无论是构造对象的重载比如线程池核心参数的给定，还是调用方法的重载比如将不同的对象通过` String.valueOf()` 转为 String 对象。其实正是由于重载的存在使用语言变得更加的 **灵活**，通过方法与参数就可以知道其方法的核心功能有更 **强的可读性**。

- **为什么会有方法的重写**

方法重新的前提一定是继承，只有在继承的基础之上子类才能对父类的原有的方法进行重写，其重写的意义在更多的是 Java 这种语言的一种 **多态的体现**。比如一个父类针对于不同的业务功能有不同众多不同的具体实现子类，而子类会对父类原有的方法进行扩展。比如在 Spring MVC 拦截器当中通过继承 `HandlerInterceptorAdapter ` 接口重写拦截方法，进行日记记录、或权限校验等。

- 两者的本质区别

上面其实已经从具体功能以及用法方面做了区分，还有一点是需要注意的，一个对象对应的重载方法时唯一可以确定的，因此在编译期就可以动态的绑定。而方法的重写时一种多态的体现，对于多态而言其父类到底指向的时哪一个子类是不确定的，只有在运行时才能最终有 JVM 确定。比如所有的对象都归属于 `Object`  超父类对象，而 `haseCode ` 的调用要在运行时才能最终确定调用的是属于那个对象的 `haseCode` 方法。

## JDK8 有哪些新特性

- **语言核心特性层面**

1. Lambda 表达式

   允许将将函数作为方法参数或将代码作为形参数据进行编码，比如一个类对象的构造方法当中定义了一个接口属性值，就可以将代码作为参数进行传递，最典型的就是在创建线程的时候通过函数实现 `Runnable` 接口。还有比如通过 Lambda 来静态方法、方法、特定类型任意对象方法于构造器的引用。

2. 默认方法

   允许接口包含其具体的实现方法，但必须使用 `default` 关键字修改或该方法时一个静态方法

3. 函数式接口

   通过 `@FunctionalInterface` 注解标记，显示的标记一个接口时函数式接口

- **新 API 和库**