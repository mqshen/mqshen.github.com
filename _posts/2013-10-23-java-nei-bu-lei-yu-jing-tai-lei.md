---
layout: post
title: "Java内部类与静态类"
description: "Java中的嵌套类（Nested Class）分为两种：静态内部类（也叫静态嵌套类，Static Nested Class）和内部类（Inner Class）。"
category: ""
tags: [Java]
---
{% include JB/setup %}
Java中的嵌套类（Nested Class）分为两种：静态内部类（也叫静态嵌套类，Static Nested Class）和内部类（Inner Class）。内部类我们介绍过很多了，现在来看看静态内部类。什么是静态内部类呢？是内部类，并且是静态（static修饰）的即为静态内部类。只有在是静态内部类的情况下才能把static修复符放在类前，其他任何时候static都是不能修饰类的。  
静态内部类的形式很好理解，但是为什么需要静态内部类呢？那是因为静态内部类有两个优点：加强了类的封装性和提高了代码的可读性，我们通过一段代码来解释这两个优点，如下所示：

    public class Person{  
         //姓名  
         private String name;  
         //家庭  
         private Home home;  
         //构造函数设置属性值  
         public Person(String _name){  
              name = _name;  
         }  
         /* home、name的getter/setter方法省略 */  
     
         public static class Home{  
              //家庭地址  
              private String address;  
              //家庭电话  
              private String tel;  
     
              public Home(String _address,String _tel){  
                address = _address;  
                tel = _tel;  
              }  
              /* address、tel的getter/setter方法省略 */  
         }  
    }

其中，Person类中定义了一个静态内部类Home，它表示的意思是“人的家庭信息”，由于Home类封装了家庭信息，不用在Person类中再定义homeAddre、homeTel等属性，这就使封装性提高了。同时我们仅仅通过代码就可以分析出Person和Home之间的强关联关系，也就是说语义增强了，可读性提高了。所以在使用时就会非常清楚它要表达的含义：

    public static void main(String[] args) {  
         //定义张三这个人  
         Person p = new Person("张三");  
         //设置张三的家庭信息  
         p.setHome(new Person.Home("上海","021"));  
    } 

定义张三这个人，然后通过Person.Home类设置张三的家庭信息，这是不是就和我们真实世界的情形相同了？先登记人的主要信息，然后登记人员的分类信息。可能你又要问了，这和我们一般定义的类有什么区别呢？又有什么吸引人的地方呢？如下所示：  
提高封装性。从代码位置上来讲，静态内部类放置在外部类内，其代码层意义就是：静态内部类是外部类的子行为或子属性，两者直接保持着一定的关系，比如在我们的例子中，看到Home类就知道它是Person的Home信息。  
提高代码的可读性。相关联的代码放在一起，可读性当然提高了。  
形似内部，神似外部。静态内部类虽然存在于外部类内，而且编译后的类文件名也包含外部类（格式是：外部类+$+内部类），但是它可以脱离外部类存在，也就是说我们仍然可以通过new Home()声明一个Home对象，只是需要导入“Person.Home”而已。  
解释了这么多，读者可能会觉得外部类和静态内部类之间是组合关系（Composition）了，这是错误的，外部类和静态内部类之间有强关联关系，这仅仅表现在“字面”上，而深层次的抽象意义则依赖于类的设计。  
那静态内部类与普通内部类有什么区别呢？问得好，区别如下：
#### 内部类
1. 内部类拥有普通类的所有特性，也拥有类成员变量的特性
2. 内部类可以访问其外部类的成员变量，属性，方法，其它内部类
#### 静态类
1. 只有内部类才能声明为static，也可以说是静态内部类
2. 只有静态内部类才能拥有静态成员，普通内部类只能定义普通成员
3. 静态类跟静态方法一样，只能访问其外部类的静态成员
4. 如果在外部类的静态方法中访问内部类，这时候只能访问静态内部类

#### 内部类的实例化
1. 访问内部类，必须使用：外部类.内部类，OutClass.InnerClass
2. 普通内部类必须绑定在其外部类的实例上
3. 静态内部类可以直接 new

