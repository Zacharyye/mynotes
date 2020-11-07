> Spring针对Java Transaction API（JTA）、JDBC、Hibernate、Java Persistence API（JPA）等事务API，实现了一致的变成模型，而Spring的声明式事务功能更是提供了极其方便的事物配置方式，配合Spring Boot的自动配置，大多数Spring Boot项目只需要在方法上标记@Transactional注解，即可一键开启方法的事务性配置。

# 小心Spring的事务可能没有生效

> - 除非特殊配置（比如使用AspectJ静态织入实现AOP），否则只有定义在public方法上的@Transactional才能生效。原因是，Spring默认通过动态代理的方式实现AOP，对目标方法进行增强，private方法无法代理到，Spring自然也无法动态增强事务处理逻辑
> - 必须通过代理过的类从外部调用目标方法才能生效

# 事务即便生效也不一定能回滚

> 通过AOP实现事务处理可以理解为，使用try...catch...来包裹标记了@Transactional注解的方法，当方法出现了异常并且满足一定条件的时候，在catch里面可以设置事务回滚，没有异常则直接提交事务。
>
> **一定条件**
>
> - 只有异常传播出了标记了@Transactional注解的方法，事务才能回滚。在Spring的TransactionAspectSupport里有个invokeWithinTransaction方法，里面就是处理事务的逻辑。可以看到，只有捕获到异常才能进行后续事务处理：
>
>   ```java
>   
>   try {
>      // This is an around advice: Invoke the next interceptor in the chain.
>      // This will normally result in a target object being invoked.
>      retVal = invocation.proceedWithInvocation();
>   }
>   catch (Throwable ex) {
>      // target invocation exception
>      completeTransactionAfterThrowing(txInfo, ex);
>      throw ex;
>   }
>   finally {
>      cleanupTransactionInfo(txInfo);
>   }
>   ```
>
> - 默认情况下，出现RuntimeException（非受检异常）或Error的时候，Spring才会回滚事务。
>
>   - 可以自己捕获异常，手动设置让当前事务处于回滚状态
>   - 在注解中声明中，期望遇到所有的Exception都回滚事务（来突破默认不回滚受检异常的限制）
>
> - 确认事务传播设置是否符合自己的业务逻辑
>
>   - 注意事务隔离级别 
>     - 读未提交 - read-uncommitted
>     - 不可重复读 - read-committed
>     - 可重复读 - repeatable-read
>     - 串行化 - serializable
>   - 事务传播行为
>     - PROPAGATION_REQUIRED
>     - PROPAGATION_SUPPORTS
>     - PROPAGATION_MANDATORY
>     - PROPAGATION_REQUIRED_NEW
>     - PROPAGATION_NOT_SUPPORTED
>     - PROPAGATION_NEVER
>     - PROPAGATION_NESTED
>     - ![这里写图片描述](https://img-blog.csdn.net/20170420212829825?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc29vbmZseQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
>
> **总结**
>
> - 务必确认调用@Transactional注解标记的方法是public的，并且是通过Spring注入的Bean进行调用的
> - 因为异常处理不正确，导致事务虽然生效但出现异常时没回滚。Spring默认只会对标记@Transactional注解的方法出现了Ru难题么E相册菩提哦你和Error的时候回滚，如果方法捕获了异常，那么需要通过手动编码处理事务回滚。如果希望Spring针对其他异常也可以回滚，那么可以相应配置@Transactional注解的rollbackFor和noRollbackFor属性来覆盖其默认设置。
> - 如果方法涉及多次数据库操作，并希望将它们作为独立的事务进行提交或者回滚，那么需要考虑进一步细化配置事务传播方法，也就是@Transactional注解的Propagation属性
> - Spring的部分Debug日志

