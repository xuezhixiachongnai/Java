#### 定义切点（拦截器）

- 在 AOP 中，Joinpoint 代表了程序执行的某个具体位置，比如方法的调用、异常的抛出等。AOP 框架通过拦截这些 Joinpoint 来插入额外的逻辑，实现横切关注点的功能。
- 而实现拦截器 MethodInterceptor 接口比较特殊，它会将所有的 @AspectJ 定义的通知最终都交给 MethodInvocation（子类 ReflectiveMethodInvocation ）去执行。