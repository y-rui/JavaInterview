### 简述 Java 的反射机制及其应用场景

#### 1.什么是反射机制？

Java反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法；这种动态获取的以及动态调用对象的方法的功能称为Java的反射机制。

- 通俗理解：通过反射，任何类对我们来说都是透明的，想要获取任何东西都可以，破坏程序安全性？

#### 2.反射机制能够获取哪些信息？

在运行时判定任意一个对象所属的类;
在运行时构造任意一个类的对象；
在运行时判定任意一个类所具有的成员变量和方法；
在运行时调用任意一个对象的方法；
生成动态代理；

- 主要的反射机制类：

```java
java.lang.Class; //类               
java.lang.reflect.Constructor;//构造方法 
java.lang.reflect.Field; //类的成员变量       
java.lang.reflect.Method;//类的方法
java.lang.reflect.Modifier;//访问权限
```

##### 2.1class对象的获取

```java
//第一种方式 通过对象getClass方法
Person person = new Person();
Class<?> class1 = person.getClass();
//第二种方式 通过类的class属性
class1 = Person.class;
try {
    //第三种方式 通过Class类的静态方法——forName()来实现
    class1 = Class.forName("club.sscai.Person");
} catch (ClassNotFoundException e) {
    e.printStackTrace();
}
```

##### 2.2获取class对象的属性、方法、构造函数等

```java
Field[] allFields = class1.getDeclaredFields();//获取class对象的所有属性
Field[] publicFields = class1.getFields();//获取class对象的public属性
try {
    Field ageField = class1.getDeclaredField("age");//获取class指定属性
    Field desField = class1.getField("des");//获取class指定的public属性
} catch (NoSuchFieldException e) {
    e.printStackTrace();
}

Method[] methods = class1.getDeclaredMethods();//获取class对象的所有声明方法
Method[] allMethods = class1.getMethods();//获取class对象的所有方法 包括父类的方法

Class parentClass = class1.getSuperclass();//获取class对象的父类
Class<?>[] interfaceClasses = class1.getInterfaces();//获取class对象的所有接口

Constructor<?>[] allConstructors = class1.getDeclaredConstructors();//获取class对象的所有声明构造函数
Constructor<?>[] publicConstructors = class1.getConstructors();//获取class对象public构造函数
try {
    Constructor<?> constructor = class1.getDeclaredConstructor(new Class[]{String.class});//获取指定声明构造函数
    Constructor publicConstructor = class1.getConstructor(new Class[]{});//获取指定声明的public构造函数
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}

Annotation[] annotations = class1.getAnnotations();//获取class对象的所有注解
Annotation annotation = class1.getAnnotation(Deprecated.class);//获取class对象指定注解

Type genericSuperclass = class1.getGenericSuperclass();//获取class对象的直接超类的 Type
Type[] interfaceTypes = class1.getGenericInterfaces();//获取class对象的所有接口的type集合
```

##### 2.3反射机制获取泛型类型

示例：

```java
//People类public class People<T> {}
//Person类继承People类public class Person<T> extends People<String> implements PersonInterface<Integer> {}
//PersonInterface接口public interface PersonInterface<T> {}
```

获取泛型类型：

```java
Person<String> person = new Person<>();
//第一种方式 通过对象getClass方法
Class<?> class1 = person.getClass();
Type genericSuperclass = class1.getGenericSuperclass();//获取class对象的直接超类的 Type
Type[] interfaceTypes = class1.getGenericInterfaces();//获取class对象的所有接口的Type集合
getComponentType(genericSuperclass);
getComponentType(interfaceTypes[0]);
```

getComponentType具体实现

```java
private Class<?> getComponentType(Type type) {
Class<?> componentType = null;
if (type instanceof ParameterizedType) {
    //getActualTypeArguments()返回表示此类型实际类型参数的 Type 对象的数组。
    Type[] actualTypeArguments = ((ParameterizedType) type).getActualTypeArguments();
    if (actualTypeArguments != null && actualTypeArguments.length > 0) {
    componentType = (Class<?>) actualTypeArguments[0];
    }
} else if (type instanceof GenericArrayType) {
    // 表示一种元素类型是参数化类型或者类型变量的数组类型
    componentType = (Class<?>) ((GenericArrayType) type).getGenericComponentType();
} else {
    componentType = (Class<?>) type;
}
return componentType;
}
```

##### 2.4通过反射机制获取注解信息(知识点)

这种场景，经常在 Aop 使用，这里重点以获取Method的注解信息为例。

```java
try {
    //首先需要获得与该方法对应的Method对象
    Method method = class1.getDeclaredMethod("jumpToGoodsDetail", new Class[]{String.class, String.class});
    Annotation[] annotations1 = method.getAnnotations();//获取所有的方法注解信息
    Annotation annotation1 = method.getAnnotation(RouterUri.class);//获取指定的注解信息
    TypeVariable[] typeVariables1 = method.getTypeParameters();
    Annotation[][] parameterAnnotationsArray = method.getParameterAnnotations();//拿到所有参数注解信息
    Class<?>[] parameterTypes = method.getParameterTypes();//获取所有参数class类型
    Type[] genericParameterTypes = method.getGenericParameterTypes();//获取所有参数的type类型
    Class<?> returnType = method.getReturnType();//获取方法的返回类型
    int modifiers = method.getModifiers();//获取方法的访问权限
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}
```

#### 3.反射机制的应用实例

大家是不是经常遇到一种情况，比如两个对象，Member 和 MemberView，很多时候我们都有可能进行相互转换，那么我们常用的方法就是，把其中一个中的值挨个 get 出来，然后再挨个 set 到另一个中去，接下来我介绍的这种方法就可以解决这种问题造成的困扰:

```java
public class TestClass{

    public double eachOrtherToAdd(Integer one,Double two,Integer three){
        return one + two + three;
    }
}
```

```java
public class ReflectionDemo{

    public static void main(String args[]){
        String className = "initLoadDemo.TestClass";
        String methodName = "eachOrtherToAdd";
        String[] paramTypes = new String[]{"Integer","Double","int"};
        String[] paramValues = new String[]{"1","4.3321","5"};

        // 动态加载对象并执行方法
        initLoadClass(className, methodName, paramTypes, paramValues);

    }

    @SuppressWarnings("rawtypes")
    private static void initLoadClass(String className,String methodName,String[] paramTypes,String[] paramValues){
        try{
            // 根据calssName得到class对象
            Class cls = Class.forName(className);

            // 实例化对象
            Object obj = cls.newInstance();

            // 根据参数类型数组得到参数类型的Class数组
            Class[] parameterTypes = constructTypes(paramTypes);

            // 得到方法
            Method method = cls.getMethod(methodName, parameterTypes);

            // 根据参数类型数组和参数值数组得到参数值的obj数组
            Object[] parameterValues = constructValues(paramTypes,paramValues);

            // 执行这个方法并返回obj值
            Object returnValue = method.invoke(obj, parameterValues);

            System.out.println("结果："+returnValue);

        }catch (Exception e){
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }

    private static Object[] constructValues(String[] paramTypes,String[] paramValues){
        Object[] obj = new Object[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++){
            if(paramTypes[i] != null && !paramTypes[i].trim().equals("")){
                if ("Integer".equals(paramTypes[i]) || "int".equals(paramTypes[i])){
                    obj[i] = Integer.parseInt(paramValues[i]);
                }else if ("Double".equals(paramTypes[i]) || "double".equals(paramTypes[i])){
                    obj[i] = Double.parseDouble(paramValues[i]);
                }else if ("Float".equals(paramTypes[i]) || "float".equals(paramTypes[i])){
                    obj[i] = Float.parseFloat(paramValues[i]);
                }else{
                    obj[i] = paramTypes[i];
                }
            }
        }
        return obj;
    }

    @SuppressWarnings("rawtypes")
    private static Class[] constructTypes(String[] paramTypes){
        Class[] cls = new Class[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++){
            if(paramTypes[i] != null && !paramTypes[i].trim().equals("")){
                if ("Integer".equals(paramTypes[i]) || "int".equals(paramTypes[i])){
                    cls[i] = Integer.class;
                }else if ("Double".equals(paramTypes[i]) || "double".equals(paramTypes[i])){
                    cls[i] = Double.class;
                }else if ("Float".equals(paramTypes[i]) || "float".equals(paramTypes[i])){
                    cls[i] = Float.class;
                }else{
                    cls[i] = String.class;
                }
            }
        }
        return cls;
    }
}
```

#### 4.反射机制的优缺点

**优点：**
运行期类型的判断，动态类加载，动态代理使用反射。

**缺点：**
性能是一个问题，反射相当于一系列解释操作，通知jvm要做的事情，性能比直接的java代码要慢很多。

#### 5.反射的应用场景

##### 5.1框架

- 反射是框架设计的灵魂。

在我们平时的项目开发过程中，基本上很少会直接使用到反射机制，但这不能说明反射机制没有用，实际上有很多设计、开发都与反射机制有关，例如模块化的开发，通过反射去调用对应的字节码；动态代理设计模式也采用了反射机制，还有我们日常使用的 Spring／Hibernate 等框架也大量使用到了反射机制。

举例：

1. 我们在使用JDBC连接数据库时使用Class.forName()通过反射加载数据库的驱动程序

2. Spring框架也用到很多反射机制，最经典的就是xml的配置模式。Spring 通过 XML 配置模式装载 Bean 的过程
   1. 将程序内所有 XML 或 Properties 配置文件加载入内存中
   2. Java类里面解析xml或properties里面的内容，得到对应实体类的字节码字符串以及相关的属性信息
   3. 使用反射机制，根据这个字符串获得某个类的Class实例
   4. 动态配置实例的属性

##### 5.2Tomcat服务器

1. Tomcat服务器应用到的Java的三大技术
   IO技术、ServerSocket技术和反射技术。
2. Tomcat服务器大致处理用户应答的思路
   1. 对外暴露接口---->著名的Servlet (服务器脚本片段)
      1. 对外提供接口的原因：具体处理客户端应答请求的方式是不一样的。应该根据具体的请求来进行具体的处理。向上抽取形成Servlet接口并提供给客户端使用。
      2. 由开发者来实现Servlet接口中定义的具体应答请求的处理方式。
   2. 提供配置文件---->web.xml (WEB宏观部署描述文件)

每个Web应用程序都有自己的配置文件web.xml来告知Tomcat服务器（App）有哪些用户自定义的Servlet实现类。

3. Tomcat具体加载处理细节
   1. Tomcat(App)首先读取配置文件web.xml中配置好的Servlet的子类名称
   2. Tomcat根据读取到的客户端实现的Servlet子类的类名字符串去寻找对应的字节码文件。如果找到就将其加载到内存。
   3. Tomcat通过预先设置好的Java反射处理机制解析字节码文件并创建相应的实例对象。之后调用所需要的方法。

【最后】Tomcat一启动，用户自定义的Servlet的子类通过Tomcat内部的反射框架也随之运行。

##### 5.3可以用于改进设计模式 

**对工厂模式的改进**

常规工厂模式

```java
interface Fruit {
 
void eat();
 
}

class Apple implements Fruit {
    public void eat(){
        System.out.println("Eat Apple");
    }
}

class Orange implements Fruit {
    public void eat(){
        System.out.println("Eat Orange");
    }
}

//编译时就确定类型，不够灵活，不好新增，如果我要新增一个实现类，就要在工厂类中改代码，耦合度较高。

public class Factory {
 
public static Fruit getInstance(String fruitName){
 
Fruit f=null;
 
        if("Apple".equals(fruitName)){
 
        f=new Apple();
 
        }
 
if("Orange".equals(fruitName)){
 
f=new Orange();
 
        }
 
return f;
 
    }
 
}
```

改进后

```java
public class ReflexFactory {
 
public static Fruit2 getInstances(String className) {
 
Fruit2 f2 =null;
 
        try {
 
            //利用反射
 
            f2 = (Fruit2)Class.forName(className).newInstance();
 
        }catch (Exception e) {
 
e.printStackTrace();
 
        }
 
return f2;
 
    }
 
}
```

当如果要新增实现类的话，只要将传入的参数改变就好，无需更改工厂内的代码。

##### 5.4AOP和IOC都是基于反射实现的

例：拦截器，注解、XML文档、token登录拦截