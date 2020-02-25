---
title: java基础1
date: 2020-02-12 15:06:04
categories: java
tags: java
---

1. 面向对象的特点：
>* 封装：将代码及其处理的数据绑定在一起的一种编程机制；
>* 继承：从已有类继承信息创建新的类；
>* 抽象(N): 将一类对象的共同特征抽取出来构造类的过程；
>* 多态：同一种类型的不同对象在调用相同方法时表现出不同的行为；原因是java中的变量拥有两种类型：编译时类型和运行时类型，具体表现在子类
重写父类方法，或者相同方法名不同参数(重载)；重载是编译时的多态性，重写是运行时的多态性；

2. java中变量、代码块、构造器之间的执行顺序：
> 原则是先执行静态、父类、变量、代码块、构造器；因此执行顺序如下：
>1. 父类中的静态变量；
>2. 父类中的静态代码块；
>3. 子类中的静态变量；
>4. 子类中的静态代码块；
>5. 父类中的普通成员变量；
>6. 父类中的动态代码块；
>7. 父类中的构造器；
>8. 子类中的普通成员变量；
>9. 子类中的动态代码块；
>10. 子类中的构造器；

3. 面向对象设计原则：
>* 开闭原则；
>* 单一职责原则；
>* 里氏代换原则；
>* 依赖倒置原则；
>* 接口隔离原则；
>* 迪米特原则；
>* 合成/聚合复用；(Extend)

4. String、StringBuffer、StringBuilder区别；
>* String是不可变对象，任何对string的改变都会生成新的对象；StringBuffer和StringBuilder是可变对象；
>* StringBuffer是线程安全的，StringBuilder是线程不安全的，效率上StringBuilder较高；

5. hashCode()和equal().
>hashCode()和equal()用于判断两个对象是否相等；hashCode()用于快速判断对象是否相等；如果hashCode()相同，equal()不一定true；
>因为一些集合类存储元素时要判断元素是否相同，因此hashCode()和equal()都需要重写，防止集合由于误判添加重复元素；

6. java中的内部类.
>java中的内部类分为静态内部类和非静态内部类；

7.反射.
>java反射是在运行时；对于任意一个已加载的类，都能知道它的所有属性和方法，对于任意的对象，都能调用它的任意方法；这种动态的获取信息和
>动态的调用对象方法被称为java的反射；

8.元注解.
>* @Document.是否被javadoc编入文档
>* @Target. 标识注解的范围，赋值类型是ElementType.
>* @Retention. 定义该注解的生命周期.RetentionPolicy.SOURCE、CLASS、RUNTIME.
>* @Inherited. 是否被继承.

9.泛型.
>泛型即参数化类型；

10.Integer缓冲池
>new Integer(112)和Integer.valueOf(112)的区别是：new Integer(112)每次都会产生新的对象，Integer.valueOf(112)
会使用缓冲池中的对象，多次调用会返回同一对象；这是因为Integer内部默认有个IntegerCache缓冲池，会缓冲-128~127的对象;

11.异常.
相关关键字：try..catch、finally、throw、throws；
>finally：必定执行，当try和catch中有return时，finally仍会执行，且finally比return先执行；finally是在return
>后面的表达式运算后执行的；

>finally不执行的情况：程序提前终止如调用System.exit，病毒或断电等；

>* 异常基类是Throwable，下面分为Error和Exception.
>* Error是指程序无法处理的错误，无法恢复也无法catch，Exception是程序可以处理的异常；
>* Exception又分为受检查异常和运行时异常；区别在于编译时是否需要处理，受检查异常编译时必须处理,例如IOException；
>运行时异常可以处理也可以不处理；常见的运行时异常有：NullPointException、ClassCastException、IndexOutOfBoundsException、
>IllegalArgumentException、NoSuchMethodException.

12.局部变量为什么一定要初始化.
>局部变量是指方法内的变量，必须初始化；因为局部变量运行时被分配在栈中，量大且生命周期短，如果虚拟机初始化开销大，但是
>变量不初始化默认值使用是不安全的，因此出于速度和安全性考虑，局部变量需要初始化；

13.内部类访问局部变量时，为什么变量一定要用final修饰.
>因为生命周期不同。局部变量在方法结束后销毁，内部类对象不一定，就会导致内部类引用了一个不存在的变量，所以编译器会在
>内部类生成一个局部变量的拷贝，但如果其中一个变量修改，就会导致两个变量可能是不同的值，因此编译器要求局部变量加final，
>保证两个变量值相同；JDK8后不需要局部变量加final，是因为编译器检查时默认加上了final.

14.如何打破ClassLoader的双亲委托机制.
>重写loadClass().

