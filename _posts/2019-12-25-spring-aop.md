---
layout: post
title:  "spring aop"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}











## Java动态代理

[Java动态代理](https://qigangzhong.github.io/2019/03/18/proxy/#%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)的方式有两种：

* JDK动态代理
* CGLib动态代理

## spring aop

### aop基本概念

* Aspect：切面类

* Join point：是在应用执行过程中能够插入切面的一个点。这个点可以是调用方法时、抛出异常时、甚至修改一个字段时。切面代码可以利用这些点插入到应用的正常流程之中，并添加新的行为。

* Advice：切入前后执行的建议

  * before：方法执行之前执行
  * after：方法执行之后执行
  * after-returning：仅在方法成功完成时，在方法执行后执行
  * after-throwing：仅在方法通过抛出异常退出时，才在方法执行后执行
  * around：在调用方法之前和之后执行，是before、after的组合

* PointCut：一个切面并不需要通知应用的所有连接点。切点有助于缩小切面所通知的连接点范围。切点的定义会匹配通知所要织入的一个或多个连接点。我们通常使用明确的类和方法名称，或是利用正则表达式定义所匹配的类和方法名称来指定这些切点。另外，有些AOP框架是允许我们创建动态的切点，可以根据运行时的决策（比如方法的参数值）来决定是否应用通知。

  > 【PointCut的表达式】
  >
  > 1. 方法标签匹配方式
  >
  > 假设定义了EmployeeManager接口。
  >
  > 1）
  > execution(* com.howtodoinjava.EmployeeManager.*(..))
  > 以上切入点表达式可以匹配EmployeeManger接口中所有的方法。
  > 2）
  > 当切面方法和EmployeeManager接口在相同的包内，如果切入点表达式匹配所有所有方法，则表达式可以改成：
  > execution(* EmployeeManager.*(..))
  > 3）匹配EmployeeManager接口的所有public方法。
  > execution(public * EmployeeManager.*(..))
  > 4) 匹配EmployeeManager接口中权限为public并返回类型为EmployeeDTO的所有方法。
  > execution(public EmployeeDTO EmployeeManager.*(..))
  > 5） 匹配EmployeeManager接口中权限为public并返回类型为EmployeeDTO，第一个参数为EmployeeDTO类型的所有方法。
  > execution(public EmployeeDTO EmployeeManager.*(EmployeeDTO, ..))
  > 6） 匹配EmployeeManager接口中权限为public、返回类型为EmployeeDTO，参数明确定义为EmployeeDTO,Integer的所有方法。
  > execution(public EmployeeDTO EmployeeManager.*(EmployeeDTO, Integer))
  >
  >
  > 2. 类型标签匹配模式
  > 1）匹配在com.howtodoinjava包下所有类型中所有的方法。
  > within(com.howtodoinjava.*)
  > 2）匹配在com.howtodoinjava包以及其子包下所有类型中所有的方法。
  > within(com.howtodoinjava..*)
  > 3）匹配其他包一个类下的所有方法。
  > within(com.howtodoinjava.EmployeeManagerImpl)
  > 4）匹配同一个包下一个类下的所有方法。
  > within(EmployeeManagerImpl)
  > 5）匹配一个接口下的所有继承者的所有方法。
  > within(EmployeeManagerImpl+)
  >
  >
  > 3. bean名字匹配模式
  > 匹配所有以Manager结尾的beans中的所有方法。
  > bean(*Manager)
  >
  >
  > 4. 切入点表达式拼接
  > 在AspectJ中，切入点表达式可以通过&&，||，!等操作符进行拼接。
  > bean(*Manager) || bean(*DAO)

* Introduction：给原有的类引入新的接口功能，参考[示例](https://gitee.com/qigangzhong/springdemo/tree/master/aop/src/main/java/com/qigang/spring_aop/introduction)

* Target object：切入的目标对象

* Weaving：织入，将切面应用到目标对象从而创建一个新的代理对象的过程，这个过程可以发生在编译期、类装载期及运行期。

### spring aop的实现方式

#### xml方式（ProxyFactoryBean）

定义一个业务服务类：

```java
public class CustomerService {
    private String name;
    private String url;

    public void setName(String name) {
        this.name = name;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public void printName() {
        System.out.println("Customer name : " + this.name);
    }

    public void printURL() {
        System.out.println("Customer website : " + this.url);
    }

    public void printThrowException() {
        throw new IllegalArgumentException();
    }

}
```

定义切面类，在业务类方法执行前后做一些事情

```java
public class CustomerInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        try {
            System.out.println("方法调用之前做一些事情....");

            Object result = methodInvocation.proceed();

            System.out.println("方法调用之后做一些事情");

            return result;
        }catch (Exception ex){
            System.out.println("执行方法抛出异常");
            throw ex;
        }
    }
}

public class MyBeforeAdvice implements MethodBeforeAdvice {

    @Override
    public void before(Method method, Object[] objects, @Nullable Object o) throws Throwable{
        System.out.println(o.getClass() + ":"+method.getName()+" 方法准备执行");
    }
}

public class MyAfterAdvice implements AfterReturningAdvice {
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        System.out.println(target.getClass()+":"+method.getName()+" 方法执行完成，返回结果:"+returnValue);
    }
}
```

XML配置

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-2.5.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!--示例1:拦截所有方法-->
    <bean id="customerService" class="com.qigang.spring_aop.CustomerService">
        <property name="name" value="nameeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"/>
        <property name="url" value="urlllllllllllllllllllllllllllllllllllll"/>
    </bean>
    <bean id="customerInterceptor" class="com.qigang.spring_aop.CustomerInterceptor"></bean>
    <bean id="beforeAdvice" class="com.qigang.spring_aop.MyBeforeAdvice"></bean>
    <bean id="afterAdvice" class="com.qigang.spring_aop.MyAfterAdvice"></bean>
    <bean id="customerServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target" ref="customerService"></property>
        <property name="interceptorNames">
            <list>
                <value>customerInterceptor</value>
                <value>beforeAdvice</value>
                <value>afterAdvice</value>
            </list>
        </property>
    </bean>
    
    <!--示例2:拦截pointcut中指定的方法-->
    <!--<bean id="customerService" class="com.qigang.spring_aop.CustomerService">
        <property name="name" value="nameeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"/>
        <property name="url" value="urlllllllllllllllllllllllllllllllllllll"/>
    </bean>
    <bean id="customerInterceptor" class="com.qigang.spring_aop.CustomerInterceptor"></bean>
    <bean id="beforeAdvice" class="com.qigang.spring_aop.MyBeforeAdvice"></bean>
    <bean id="afterAdvice" class="com.qigang.spring_aop.MyAfterAdvice"></bean>
    <bean id="customerServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target" ref="customerService"></property>
        <property name="interceptorNames">
            <list>
                <value>customerAdvisor</value>
                <value>beforeAdvice</value>
                <value>afterAdvice</value>
            </list>
        </property>
    </bean>
    <bean id="customerPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
        <property name="mappedName" value="printName"></property>
    </bean>
    <bean id="customerAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
        <property name="advice" ref="customerInterceptor"></property>
        <property name="pointcut" ref="customerPointcut"></property>
    </bean>-->
</beans>
```

测试方法

```java
ApplicationContext appContext = new ClassPathXmlApplicationContext("applicationContext.xml");
CustomerService cust = (CustomerService) appContext.getBean("customerServiceProxy");
cust.printName();
cust.printURL();
try {
    cust.printThrowException();
} catch (Exception e) {
    System.out.println("出现异常了...");
}
```

#### 代码方式（ProxyFactory）

利用代码方式可以实现上面一样的功能

```java
ProxyFactory proxyFactory = new ProxyFactory();
NameMatchMethodPointcut pointcut=new NameMatchMethodPointcut();
pointcut.setMappedName("*");
proxyFactory.addAdvisor(new DefaultPointcutAdvisor(pointcut, new CustomerInterceptor()));
proxyFactory.addAdvisor(new DefaultPointcutAdvisor(pointcut, new MyBeforeAdvice()));
proxyFactory.addAdvisor(new DefaultPointcutAdvisor(pointcut, new MyAfterAdvice()));
proxyFactory.setTarget(new CustomerService());
CustomerService cust = (CustomerService)proxyFactory.getProxy();
cust.printName();
cust.printURL();
try {
    cust.printThrowException();
} catch (Exception e) {
    System.out.println("出现异常了...");
}
```

#### aspectj+xml

```java
public class Logging {
    public void beforeAdvice() {
        System.out.println("beforeAdvice");
    }
    public void afterAdvice() {
        System.out.println("afterAdvice");
    }
    public void afterReturningAdvice(Object retVal){
        System.out.println("afterReturningAdvice:" + retVal.toString() );
    }
    public void AfterThrowingAdvice(IllegalArgumentException ex) {
        System.out.println("exception: " + ex.toString());
    }
}
```

```java
public class Student {
    private Integer age;
    private String name;
    public void setAge(Integer age) {
        this.age = age;
    }
    public Integer getAge() {
        System.out.println("Age : " + age );
        return age;
    }
    public void setName(String name) {
        this.name = name;
    }
    public String getName() {
        System.out.println("Name : " + name );
        return name;
    }
    public void printThrowException(){
        System.out.println("Exception raised");
        throw new IllegalArgumentException();
    }
}
```

```java
ApplicationContext context = new ClassPathXmlApplicationContext("aspectj.xml");
Student student = (Student) context.getBean("student");
student.getName();
student.getAge();
student.printThrowException();
```

xml配置文件内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <aop:config>
        <aop:aspect id = "log" ref = "logging">
            <aop:pointcut id = "allMethods" expression = "execution(* com.qigang.aspectj.*.*(..))"/>
            <aop:before pointcut-ref = "allMethods" method = "beforeAdvice"/>
            <aop:after pointcut-ref = "allMethods" method = "afterAdvice"/>
            <aop:after-returning pointcut-ref = "allMethods" returning = "retVal" method = "afterReturningAdvice"/>
            <aop:after-throwing pointcut-ref = "allMethods" throwing = "ex" method = "AfterThrowingAdvice"/>
        </aop:aspect>
    </aop:config>

    <bean id = "student" class = "com.qigang.aspectj.Student">
        <property name = "name"  value = "Zara" />
        <property name = "age"  value = "11"/>
    </bean>

    <bean id = "logging" class = "com.qigang.aspectj.Logging"/>
</beans>
```

#### aspectj+annotation

```java
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class Logging2 {

    @Pointcut("execution(* com.qigang.aspectj.*.*(..))")
    public void allMethods(){}

    @Before("allMethods()")
    public void beforeAdvice() {
        System.out.println("beforeAdvice");
    }

    @After("allMethods()")
    public void afterAdvice() {
        System.out.println("afterAdvice");
    }

    @AfterReturning(pointcut = "allMethods()", returning = "retVal")
    public void afterReturningAdvice(Object retVal){
        System.out.println("afterReturningAdvice:" + retVal );
    }

    @AfterThrowing(pointcut = "allMethods()", throwing = "ex")
    public void AfterThrowingAdvice(IllegalArgumentException ex) {
        System.out.println("exception: " + ex.toString());
    }
}
```

```java
@Component("student")
public class Student {
    //...
}
```

```java
ApplicationContext context = new ClassPathXmlApplicationContext("aspectj.xml");
Student student = (Student) context.getBean("student");
student.getName();
student.getAge();
student.printThrowException();
```

xml配置文件内容只保留两行

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!--测试AspectJ-->
    <aop:aspectj-autoproxy/>
    <context:component-scan base-package="com.qigang.aspectj"/>
</beans>
```

## 参考
