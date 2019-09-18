[TOC]
# spring多线程事务
spring事务是线程级别的，通过threadLocal保存，因此事务的传播也只能在同一个线程调用多个事务方法时才有意义。如果在事务方法的主线程中，开启子线程执行新的事务方法，此时子线程的事务是新开一个事务，而不会和主线程共享一个事务。
如果采用编程式事务，需要手动开启事务、提交和回滚。

# spring事务原理
如何保证事务的开启、提交和回滚使用同一个数据源连接呢？--threadLocal。
对声明式注解@Transactional标记的方法，那代理方法首先执行增强，判断当前线程是否存在connection，不存在则新建并绑定到当前线程(通过threadLocal进行绑定)，然后通过反射执行目标方法，最后回到代理方法执行增强，如提交回滚和归还连接到连接池等。

# 一个类中的方法互相调用导致事务失效
spring中的一个类多个方法互相调用，会导致被调用的方法事务失效；这是因为，调用自身方法时，其实是调用原对象的方法，而不是通过调用代理类的方法来执行的，那么事务自然就会失效。事务生效的前提必须是通过代理类，代理类中会通过反射调用原先的方法来执行，在反射调用前后指向代理方法的逻辑。
如何解决呢？
1. 可以手动拿到代理类对象，业务上直接调用代理类对象的方法；context.getBean(ServiceA.class)。因ServiceA含有Transactional注解，容器生成的bean就是代理对象。或者，((ServiceA)AopContext.currentProxy()).methodB()，需要设置<aop:aspectj-autoproxy expose-proxy="true"></aop:aspectj-autoproxy>
2. 将方法放在不同的service类中。