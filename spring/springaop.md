# Spring AOP

Spring 框架通过定义切面，通过拦截切点实现了不同业务模块的解耦，这个就叫**AOP 面向切面编程**

AOP 是一种设计思想，本质上是为了解耦

AOP 理念：将分散在各个业务逻辑代码中相同的代码通过**横向切割**的方式抽取到一个独立的模块中。它所面对的是处理过程的某个步骤或阶段，以获得逻辑过程的中各部分之间低耦合的隔离效果。

比如要实现一个日志功能，我们如果在每一个需要调用日志的地方加入日志代码，这样会很耦合和冗余。如果我们把日志单独抽取出来，在调用需要日志的方法的时候拦截该方法，调用日志模块。这样就实现了解耦。

#### AOP 术语

**连接点（Jointpoint）**：表示需要在程序中插入横切关注点的扩展点，**连接点可能是类初始化、方法执行、方法调用、字段调用或处理异常等等**，Spring只支持方法执行连接点，在AOP中表示为**在哪里干**；

**切入点（Pointcut）**： 选择一组相关连接点的模式，即可以认为连接点的集合，Spring支持perl5正则表达式和AspectJ切入点模式，Spring默认使用AspectJ语法，在AOP中表示为**在哪里干的集合**；

**通知（Advice）**：在连接点上执行的行为，通知提供了在AOP中需要在切入点所选择的连接点处进行扩展现有行为的手段；包括前置通知（before advice）、后置通知(after advice)、环绕通知（around advice），在Spring中通过代理模式实现AOP，并通过拦截器模式以环绕连接点的拦截器链织入通知；在AOP中表示为**干什么**；

**方面/切面（Aspect）**：横切关注点的模块化，比如上边提到的日志组件。可以认为是通知、引入和切入点的组合；在Spring中可以使用Schema和@AspectJ方式进行组织实现；在AOP中表示为**在哪干和干什么集合**；

**引入（inter-type declaration）**：也称为内部类型声明，为已有的类添加额外新的字段或方法，Spring允许引入新的接口（必须对应一个实现）到所有被代理对象（目标对象）, 在AOP中表示为**干什么（引入什么）**；

**目标对象（Target Object）**：需要被织入横切关注点的对象，即该对象是切入点选择的对象，需要被通知的对象，从而也可称为被通知对象；由于Spring AOP 通过代理模式实现，从而这个对象永远是被代理对象，在AOP中表示为**对谁干**；

**织入（Weaving）**：把切面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。这些可以在编译时（例如使用AspectJ编译器），类加载时和运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入。在AOP中表示为**怎么实现的**；

**AOP代理（AOP Proxy）**：AOP框架使用代理模式创建的对象，从而实现在连接点处插入通知（即应用切面），就是通过代理来对目标对象应用切面。在Spring中，AOP代理可以用JDK动态代理或CGLIB代理实现，而通过拦截器模型应用切面。在AOP中表示为**怎么实现的一种典型方式**；

##### 通知类型

**前置通知（Before advice）**：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）。

**后置通知（After returning advice）**：在某连接点正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回。

**异常通知（After throwing advice）**：在方法抛出异常退出时执行的通知。

**最终通知（After (finally) advice）**：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。

**环绕通知（Around Advice）**：包围一个连接点的通知，如方法调用。这是最强大的一种通知类型。环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行。

**Spring AOP** 如何将需要的逻辑代码织入对应的方法呢。

它使用的是动态织入的方式。在程序运行时将要增强的代码织入到目标类中。它是通过动态代理的方式实现的，如 JDK 通过反射实现的动态代理、底层通过继承实现的CGLIB的动态代理

> **Spring 在实现 AOP 时的默认策略**：
>
> - 如果目标对象实现接口 → 使用 JDK 动态代理
> - 如果目标对象没有接口 → 使用 CGLIB 代理
> - 可通过 `@EnableAspectJAutoProxy(proxyTargetClass = true)` 强制使用 CGLIB

而另一种则是静态织入的方式，ApectJ 采用的就是这种。在编译其间，使用AspectJ的acj编译器(类似javac)把aspect类编译成class字节码后，在java目标类编译时织入，即先编译aspect类再编译目标类。

### 配置 AOP

在这里，我们只介绍使用注解的方式配置 AOP

**开启注解 AOP**

```java
@Configuration
@EnableAspectJAutoProxy  // 开启 AOP 注解支持
public class AppConfig {}
```

在配置类中使用 **@EnableAspectJAutoProxy** 启用 Spring AOP 注解支持

可选属性：

- `proxyTargetClass = true`：强制使用 CGLIB 代理（即使目标对象实现了接口）

**定义切面类**

```java
@Aspect               // 标记这是一个切面类
@Component            // 将切面注册到 Spring 容器
public class LogAspect {
    ...
}
```

**定义通知（Advice）**

Spring 提供的通知类型

| 注解              | 作用                                                     |
| ----------------- | -------------------------------------------------------- |
| `@Before`         | 前置通知，方法调用前执行                                 |
| `@After`          | 后置通知，方法调用后执行（无论异常与否）                 |
| `@AfterReturning` | 返回通知，方法正常返回后执行                             |
| `@AfterThrowing`  | 异常通知，方法抛出异常后执行                             |
| `@Around`         | 环绕通知，可在方法前后执行逻辑，甚至决定是否执行目标方法 |

**切点表达式（Pointcut）**

**格式**：`execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)`

```java
@Before("execution(* com.example.service.UserService.*(..))")
```

定义可重用切点

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceMethods() {}

@Before("serviceMethods()")
public void beforeService() {
    System.out.println("方法执行前日志");
}
```

> 切面类必须是 **Spring 管理的 Bean**（`@Component` 或通过 `@Bean` 注册）
>
> 目标对象也必须是 **Spring 管理的 Bean**
>
> 注入 AOP 后：
>
> - 对目标对象的调用会被代理
> - 执行切面逻辑
>
> 多种增强通知的顺序：
>
> 当有**多个切面**同时作用于同一个方法时，Spring 会按照**切面的优先级**执行
>
> 我们可以通过`@Order`注解设置优先级（值越小，优先级越高）
>
> ```java
> @Aspect
> @Component
> @Order(1) // 优先级高
> public class LogAspect {}
> 
> @Aspect
> @Component
> @Order(2) // 优先级低
> public class TransactionAspect {}
> 
> ```

接下来我们使用 AOP 来实现一个 Redis 缓存



#### 定义切点（拦截器）

- 在 AOP 中，Joinpoint 代表了程序执行的某个具体位置，比如方法的调用、异常的抛出等。AOP 框架通过拦截这些 Joinpoint 来插入额外的逻辑，实现横切关注点的功能。
- 而实现拦截器 MethodInterceptor 接口比较特殊，它会将所有的 @AspectJ 定义的通知最终都交给 MethodInvocation（子类 ReflectiveMethodInvocation ）去执行。