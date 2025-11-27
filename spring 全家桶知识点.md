# spring aop
## aop的介绍
面向切面编程：不改变原有的代码 在其上添加一个代理的对像，使其具备其他的功能
## aop的组成部分
 * 切面 **Aspect** ：我的理解就是一个 java 类，它里面存储相关的 **切入点 （Pointcut）** 和 **通知 （Advice）**
* 连接点 **Join Point**：所有要进行切入方法集合
* 通知点 **Advice**：在连接点执行的动作
1.  @Before : 方法执行前
2.  @After：方法执行后，不管是否抛异常
3.  @After Returning：方法执成功执行后
4.  @After Throwing：方法执行抛出异常后
5.  @Around ：方法执行前后都会执行
* 切入点 **Pointcut** ：具体到哪一个连接点真的要进行切入，可以使用 AspectJ的语法来进行规定
* 目标对象 **Target Object** ：要切入点类
* 代理 **Proxy** :增强目标对象功能的代理对象
## aop实现方式
* jdk动态代理 ：基于接口来实现
* CGLIB: 基于继承来实现
#   




