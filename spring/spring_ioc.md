



# 1  加载容器

```
加载spring容器
        ApplicationContext spring的顶层核心接口
        ClassPathXmlApplicationContext 根据项目路径的xml配置来实例化spring容器
        FileSystemXmlApplicationContext 根据磁盘路径的xml配置来实例化spring容器
        AnnotationConfigApplicationContext 根据javaconfig 来配置实例化spring容器
        在容器实例化的时候 就会加载所有的bean
```



# 2  获取bean

```
1.通过类来获取bean    getBean(User.class)

2.通过bean的名字或者id来获取Bean   (User) ioc.getBean("user");

3.通过名字+类型
```

## 2.2  bean 命名

```xml
<!--设置Bean的别名-->
    <alias name="user" alias="tulingxueyuan"></alias>

<!--使用name也可以设置别名
    使用 空格或者，或者； 设置多个别名
    -->
    <bean class="cn.tulingxueyuan.beans.User" id="user" name="user2 user3,user4;user5">
        <description>用来描述一个Bean是干嘛的</description>
    </bean>
```

## 2.3  实例化bean

### 2.3.1  基于setter方法的依赖注入

```xml
<!--基于setter方法的依赖注入
     注意： name属性对应的set方法的名字
            比如 setId    ->  name="id"    setXxx   -> name="xxx"
     -->
    <bean class="cn.tulingxueyuan.beans.User" id="user6">
        <property name="idxx" value="1"></property>
        <property name="username" value="徐庶"></property>
        <property name="realname" value="吴彦祖"></property>
    </bean>
```

```java
 public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getRealName() {
        return realName;
    }

    public void setRealName(String realName) {
        this.realName = realName;
    }
```



### 2.3.2  基于构造方法的依赖注入

```xml
 <!--基于构造函数的依赖注入
        1. 基于name属性设置构造函数参数
        2. 可以只有value属性
        3. 如果省略name属性 一定注意参数顺序
        4. 如果参数顺序错乱
             可以使用name,
             还可以使用index:设置参数的下标  从0开始
             还可以使用type: 在错乱的参数类型一致的情况下不能使用
    -->
    <bean class="cn.tulingxueyuan.beans.User" id="user7">
        <constructor-arg name="id" value="1"></constructor-arg>
        <constructor-arg name="username"  value="徐庶"></constructor-arg>
        <constructor-arg name="realname"  value="刘德华"></constructor-arg>
    </bean>

```

```java
 public User(int id, String name, String realName) {
        this.id = id;
        this.name = name;
        this.realName = realName;
    }
```

### 2.3.3  依赖和配置的细节 

#### 2.3.3.1  直接值(基本类型，String等)

#### 2.3.3.2 对其他bean的引用(装配)

#### 2.3.3.3 内部bean

#### 2.3.3.4 集合

#### 2.3.3.5 null和空的字符串值

#### 2.3.3.6 使用p命名空间简化基于setter属性注入XML配置

#### 2.3.3.7 使用c命名空间简化基于构造函数的XML

```xml
 <!--使用p命名空间简化基于setter属性注入XML配置
        p:按Alt+Enter 自动加上命名空间
        设置基本数据类型   或者p:wife-ref 引用外部bean
        如果有集合类型 就不支持， 需要额外配置 <property>
    -->
    <bean class="cn.tulingxueyuan.beans.Wife" id="wife" p:age="18" p:name="迪丽热巴">
    </bean>
    <bean class="cn.tulingxueyuan.beans.Person"  id="person2" p:wife-ref="wife2" >
        <property name="hobbies">
            <list>
                <value>唱歌</value>
                <value>跳舞</value>
            </list>
        </property>
    </bean>
 <!--使用c命名空间简化基于构造函数的XML-->
    <bean class="cn.tulingxueyuan.beans.Wife" id="wife2" c:age="20" c:name="林青霞">
        <!--<constructor-arg></constructor-arg>-->
    </bean>
```

### 2.3.4 使用depend-on 

```xml
<!‐‐使用depends‐on可以设置先加载的Bean 也就是控制bean的加载顺序‐‐>
 <bean class="cn.tulingxueyuan.beans.Person" id="person" depends‐
on="wife"></bean>
 <bean class="cn.tulingxueyuan.beans.Wife" id="wife"></bean>
```



### 2.3.5 懒加载bean

```xml
<!‐‐使用lazy‐init设置懒加载
 默认为false: 在spring容器创建的时候加载（实例化）
 true: 在使用的时候(getBean)才会去加载（实例化）‐‐>
 <bean class="cn.tulingxueyuan.beans.Person" id="person2" lazy‐
init="true">
 <property name="id" value="1"></property>
 <property name="realName" value="吴彦祖"></property>
 <property name="name" value="徐庶"></property>
 </bean>
```



### 2.3.6 自动注入

byType：按照类型进行装配,以属性的类型作为查找依据去容器中找到这个组件，如果有多个类型相同的bean对象，那么会报异常，如果找不到则装配null

constructor：按照构造器进行装配，先按照有参构造器参数的类型进行装配，没有就直接装配null；如果按照类型找到了多个，那么就使用参数名作为id继续匹
配，找到就装配，找不到就装配null



## 2.4 bean的作用域

### 2.4.1 Singleton(单例)的作用域

### 2.4.2 prototype(原型)的作用域

```xml
singleton 默认:单例 只会在Ioc容器种创建一次
prototype 多例（原型bean) 每次获取都会new一次新的bean‐‐>

 <bean class="cn.tulingxueyuan.beans.Person" id="person3"
scope="prototype">
     <property name="id" value="1"></property>7 <property name="realName" value="吴彦祖"></property>
 <property name="name" value="徐庶"></property>
 </bean>
```





### 2.3.7  使用静态工厂方法实例化

```xml
<bean class="com.tuling.service.impl.UserServiceImpl" id="userService2"
 factory‐method="createUserServiceInstance" >
 </bean>
```

```java
public static UserServiceImpl createUserServiceInstance(){
 return new UserServiceImpl();
 }
```

### 2.3.8 使用实例工厂方法实例化

```xml
<bean class="com.tuling.service.impl.UserServiceImpl" id="userService"factory‐bean="UserServiceFactory" factory‐method="createUserFactory" >
 </bean>

```

```java
public class UserServiceFactory{

 public UserServiceImpl createUserFactory(){
 return new UserServiceImpl();
 }
 }
```





##  2.4  bean生命周期

```java

/**
 * 生命周期回调
 * 1. 使用接口实现的方式来实现生命周期的回调：
 * 1.1 初始化方法： 实现接口： InitializingBean 重写afterPropertiesSet方法
初始化会自动调用的方法
 * 1.1 销毁的方法： 实现接口： DisposableBean 重写destroy 方法 销毁的时候自动
调用方法
 * 什么时候销毁：在spring容器关闭的时候 close()
 * 或者 使用ConfigurableApplicationContext.registerShutdownHook方法优雅的关
闭
 *
 * 2. 使用指定具体方法的方式实现生命周期的回调：
 * 在对应的bean里面创建对应的两个方法
 * init‐method="init" destroy‐method="destroy"
 */
```

## 2.5 创建第三方bean对象

在Spring中，很多对象都是单实例的，在日常的开发中，我们经常需
要使用某些外部的单实例对象，例如数据库连接池，下面我们来讲解下如何在
spring中创建第三方bean实例









 