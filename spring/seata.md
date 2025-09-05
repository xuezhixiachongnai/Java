# [Seata](https://www.techgrow.cn/posts/e8b71fbe.html)

# 概念

分布式事务

Seata 分 TC、TM 和 RM 三个角色，TC（Server 端）需要单独作为服务端部署，TM 和 RM（Client 端）由业务系统集成

seata的三大组件

- TC：事务协调器，维护全局和分支事务的状态，负责协调并驱动全局事务的提交或回滚
- TM：事务管理器，控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议
- RM：资源管理器，负责管理分支事务上的资源，向 TC 注册分支事务，上报分支事务的状态，接受 TC 的命令来提交或者回滚分支事务

![](https://www.techgrow.cn/asset/2020/12/seata-modules.png)

> XID 是全局事务的唯一标识，它可以在服务的调用链路中传递，绑定到服务的事务上下文中

## [Seata的四大事务模式](https://www.techgrow.cn/posts/84d3f3e6.html)

### @GlobalTransactional

@GlobalTransactional的作用时开启一个全局事务

当在某个方法上加上@GlobalTransactional，这个事务就是全局事务的发起方。TC就会为这个方法开启一个全局事务XID，这个方法调用的其他微服务如果用到@Transactional或@GlobalTransactional就会被seata拦截并纳入之歌全局事务。一旦方法抛出异常，就会触发全局回滚，所有分支事务都会撤销。