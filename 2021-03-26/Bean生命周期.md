### Spring Bean生命周期

#### 概述

`Spring的ioc容器功能非常强大，负责Spring的Bean的创建和管理等功能。而Spring 的bean是整个Spring应用中很重要的一部分，了解Spring Bean的生命周期对我们了解整个spring框架会有很大的帮助。
BeanFactory和ApplicationContext是Spring两种很重要的容器,前者提供了最基本的依赖注入的支持，而后者在继承前者的基础进行了功能的拓展，例如增加了事件传播，资源访问和国际化的消息访问等功能。本文主要介绍了ApplicationContext和BeanFactory两种容器的Bean的生命周期。`

#### ApplicationContext Bean生命周期

##### 流程

![1](/Users/yinrui/repository/JavaInterview/2021-03-26/1.webp)

ApplicationContext容器中，Bean的生命周期流程如上图所示，流程大致如下：

1. 首先容器启动后，会对scope为singleton且非懒加载的bean进行实例化

2. 按照Bean定义信息配置信息，注入所有的属性

3. 如果Bean实现了BeanNameAware接口，会回调该接口的setBeanName()方法，传入该Bean的id，此时该Bean就获得了自己在配置文件中的id

4. 如果Bean实现了BeanFactoryAware接口,会回调该接口的setBeanFactory()方法，传入该Bean的BeanFactory，这样该Bean就获得了自己所在的BeanFactory

5. 如果Bean实现了ApplicationContextAware接口,会回调该接口的setApplicationContext()方法，传入该Bean的ApplicationContext，这样该Bean就获得了自己所在的ApplicationContext

6. 如果有Bean实现了BeanPostProcessor接口，则会回调该接口的postProcessBeforeInitialzation()方法

7. 如果Bean实现了InitializingBean接口，则会回调该接口的afterPropertiesSet()方法

8. 如果Bean配置了init-method方法，则会执行init-method配置的方法

9. 如果有Bean实现了BeanPostProcessor接口，则会回调该接口的postProcessAfterInitialization()方法

10. 经过流程9之后，就可以正式使用该Bean了,对于scope为singleton的Bean,Spring的ioc容器中会缓存一份该bean的实例，而对于scope为prototype的Bean,每次被调用都会new一个新的对象，期生命周期就交给调用方管理了，不再是Spring容器进行管理了

11. 容器关闭后，如果Bean实现了DisposableBean接口，则会回调该接口的destroy()方法

12. 如果Bean配置了destroy-method方法，则会执行destroy-method配置的方法，至此，整个Bean的生命周期结束

##### 示例

定义一个Person类，该类实现BeanNameAware，BeanFactoryAware，ApplicationContextAware，InitializingBean，DisposableBean五个接口，并且在applicationContext.xml文件中配置了该Bean的id为person1，并且配置了init-method和destroy-method，为该Bean配置了属性name为jack的值，然后定义了一个MyBeanPostProcessor方法，该方法实现了BeanPostProcessor接口，且在applicationContext.xml文件中配置了该方法的Bean，其代码如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
    xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
            http://www.springframework.org/schema/beans 
            http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context-3.2.xsd">
                    
     <bean id="person1" destroy-method="myDestroy" 
            init-method="myInit" class="com.test.spring.life.Person">
        <property name="name">
            <value>jack</value>
        </property>
    </bean>
    
    <!-- 配置自定义的后置处理器 -->
     <bean id="postProcessor" class="com.pingan.spring.life.MyBeanPostProcessor" />
</beans>
```

```java
public class Person implements BeanNameAware, BeanFactoryAware,
        ApplicationContextAware, InitializingBean, DisposableBean {

    private String name;
    
    public Person() {
        System.out.println("PersonService类构造方法");
    }
    
    
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
        System.out.println("set方法被调用");
    }

    //自定义的初始化函数
    public void myInit() {
        System.out.println("myInit被调用");
    }
    
    //自定义的销毁方法
    public void myDestroy() {
        System.out.println("myDestroy被调用");
    }

    public void destroy() throws Exception {
        // TODO Auto-generated method stub
     System.out.println("destory被调用");
    }

    public void afterPropertiesSet() throws Exception {
        // TODO Auto-generated method stub
        System.out.println("afterPropertiesSet被调用");
    }

    public void setApplicationContext(ApplicationContext applicationContext)
            throws BeansException {
        // TODO Auto-generated method stub
       System.out.println("setApplicationContext被调用");
    }

    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        // TODO Auto-generated method stub
         System.out.println("setBeanFactory被调用,beanFactory");
    }

    public void setBeanName(String beanName) {
        // TODO Auto-generated method stub
        System.out.println("setBeanName被调用,beanName:" + beanName);
    }
    
    public String toString() {
        return "name is :" + name;
    }
```

```java
public class MyBeanPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean,
            String beanName) throws BeansException {
        // TODO Auto-generated method stub
        
        System.out.println("postProcessBeforeInitialization被调用");
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean,
            String beanName) throws BeansException {
        // TODO Auto-generated method stub
        System.out.println("postProcessAfterInitialization被调用");
        return bean;
    }

}
```

```java
public class AcPersonServiceTest {

    public static void main(String[] args) {
        // TODO Auto-generated method stub

        System.out.println("开始初始化容器");
        ApplicationContext ac = new ClassPathXmlApplicationContext("com/test/spring/life/applicationContext.xml");
        
        System.out.println("xml加载完毕");
        Person person1 = (Person) ac.getBean("person1");
        System.out.println(person1);        
        System.out.println("关闭容器");
        ((ClassPathXmlApplicationContext)ac).close();
        
    }

}
```

我们启动容器，可以看到整个调用过程：

```
开始初始化容器
九月 25, 2016 10:44:50 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@b4aa453: startup date [Sun Sep 25 22:44:50 CST 2016]; root of context hierarchy
九月 25, 2016 10:44:50 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [com/test/spring/life/applicationContext.xml]
Person类构造方法
set方法被调用
setBeanName被调用,beanName:person1
setBeanFactory被调用,beanFactory
setApplicationContext被调用
postProcessBeforeInitialization被调用
afterPropertiesSet被调用
myInit被调用
postProcessAfterInitialization被调用
xml加载完毕
name is :jack
关闭容器
九月 25, 2016 10:44:51 下午 org.springframework.context.support.ClassPathXmlApplicationContext doClose
信息: Closing org.springframework.context.support.ClassPathXmlApplicationContext@b4aa453: startup date [Sun Sep 25 22:44:50 CST 2016]; root of context hierarchy
destory被调用
myDestroy被调用
```

#### BeanFactory Bean生命周期

##### 流程

![2](/Users/yinrui/repository/JavaInterview/2021-03-26/2.webp)

BeanFactoty容器中, Bean的生命周期如上图所示，与ApplicationContext相比，有如下几点不同:

1. BeanFactory容器中，不会调用ApplicationContextAware接口的setApplicationContext()方法，

2. BeanPostProcessor接口的postProcessBeforeInitialzation()方法和postProcessAfterInitialization()方法不会自动调用，必须自己通过代码手动注册

3. BeanFactory容器启动的时候，不会去实例化所有Bean,包括所有scope为singleton且非懒加载的Bean也是一样，而是在调用的时候去实例化。

##### 示例

我们还是使用前面定义好的Person类和MyBeanPostProcessor类，以及ApplicationContext.xml文件，main函数实现如下：

```java
public class BfPersonServiceTest {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        System.out.println("开始初始化容器");  
        ConfigurableBeanFactory bf = new XmlBeanFactory(new ClassPathResource("com/pingan/spring/life/applicationContext.xml"));
        System.out.println("xml加载完毕");      
        //beanFactory需要手动注册beanPostProcessor类的方法
        bf.addBeanPostProcessor(new MyBeanPostProcessor());
        Person person1 = (Person) bf.getBean("person1");
            System.out.println(person1);
        System.out.println("关闭容器");
        bf.destroySingletons();
    }

}
```

启动容器，我们可以看到整个调用流程:

```
开始初始化容器
九月 26, 2016 12:27:05 上午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [com/pingan/spring/life/applicationContext.xml]
xml加载完毕
PersonService类构造方法
set方法被调用
setBeanName被调用,beanName:person1
setBeanFactory被调用,beanFactory
postProcessBeforeInitialization被调用
afterPropertiesSet被调用
myInit被调用
postProcessAfterInitialization被调用
name is :jack
关闭容器
destory被调用
myDestroy被调用
```

