---
title: 《Scala实用指南》读书笔记(一)
date: 2018-07-17T15:14:37+08:00
tags:
- scala
image: summer2.jpg
---
## 前言
好久没写博客了,在`Kafka`源码的海洋里挣扎ing,休息的时候还刷`Leetcode`玩儿,很多东西匆匆丢到`OneNote`里了.  
最近花了一周时间看了品神翻译的《Scala实用指南》, 这本书应该主要是面向刚入Scala大门的Javaer的,前面讲Scala基础,后期还讲了下Akka和Scala具体的应用.  
我虽然写scala也有一段时间了,再看这本书还是觉得受益匪浅,有很多地方以前没有注意到的.   
那么本文就把一些细节写写,仅仅是枚举一下记录一下,并不全面.  
P.S. 看书勘误找品神, 找到错误了发红包诶,我就领了一个冰可乐(等值)红包.  
![](hepin.png)  

## 第2章 体验Scala
- scala的REPL中, `Ctrl+A`调到行首,`Ctrl+E`调到行尾
- `scala <scala源文件>`命令可以在单独的JVM中运行scala代码,而实际上是自动合成Main类调用main()方法

## 第3章 从Java到Scala
- 用`val`定义所有字段,并只提供读不提供更新状态的方法,可以使一个类的实例不可变
- `1 to 3`,`1 until 3`这类表达式其实是隐式转换,`1`通过`intWrapper()`方法转换为成富封装器(rich wrapper,这里具体是`RichInt`类)再调用其`to()`,`until()`方法
- Scala将`Scala.Int`视作Java基本类型int是纯粹的编译期转换
- `val (a,b)=(1,2)`这种赋值方法叫`多重赋值(multiple assignment)`
- 元组的索引越界是在编译期报错的
- 一个函数如`max(values:Int*)`接受可变长度参数值,但不是字面上的数组类型,如果调用
 
```scala
max(Array(1,2))
```
 在编译期会报错,可以使用数组展开标记(Array explode notation)告诉编译期将数组展开成所需的形式:  
 
```scala
max(Array(1,2):_*)
```
- 重载基类方法时,要小心参数默认值,如果派生类重载方法使用的参数默认值和基类对应的不一样,会让人困惑
- 重载基类方法时,应该保持参数名字一致性,否则使用参数命名调用(如`f(a=3,b=4)`)时会优先使用基类的
- 隐式参数需要将参数标记为`implicit`且放在单独的参数列表中,如:  

```scala
f(a:int,b:Int)(implicit c:User)
```
隐式参数的值传递是可选的,没有传值时,scala会在调用的作用域中寻找一个隐式变量,因此每个作用域中每一种类型最多只能有一个隐式变量,如上面的例子中,定义一个隐式变量:  

```scala
implicit def user:User = User("Leibniz")
```
- scala能自动将String转换成`scala.runtime.RichString`,因此可以使用`capitalize()`,`lines()`,`reverse()`等方法
- 跨行字符串:将多行的字符串放在`"""..."""`中,里面的引号和斜杠等等字符甚至不需要转译,又称为`原始字符串`,创建正则的时候就很方便了.
- `原始字符串`中的缩进也会被带入结果字符串中,可以使用`stripMargin()`方法将管道符号`|`前面的空白或控制字符去掉, 如:  

```scala
val str = """Leibniz said 
          |"Scala is exciting".""".stripMargin
```
- 字符串拼接用字符串插值,如`s"...${expresion}"`,s代表s插值器,如果是简单的一个变量直接`$val`,复杂表达式才需要大括号括起来,而美元符号`$`此时需要转译:`$$`.表达式的值在插值时被捕获,变量后续的变更不会影响字符串.
- 字符串格式化可以用f插值器,如`f"${a}%2.2f"`将a变量显示小数点后2位,此外还有`%s`字符串和`%d`整数等格式
- scala的类和方法默认公开
- scala不强制要求捕获异常
- scala默认导入的`scala`和`scala.Predef`包,包含了一些默认类型,隐式转换等等
- scala没有操作符重载, 但是方法名可以是+_*/等操作符,同时方法调用的句号和括号可以省略,因此有操作符重载的假象;对这些方法,同级操作的左边优先级更高, scala使用方法的第一个字符决定其优先级,从低到高分别为:  

```scala
|
^
&
< >
= !
:
 - +
/ * %
```
- scala中赋值操作的返回值是Unit
- scala中==是基于值的对比(等效于`equals()`方法),需要比较引用的可以用`eq()`方法
- scala的protected方法只有派生类可以访问,同包的非派生类不可访问,在派生类中也不可以通过基类实例来访问,只能是通过派生类的实例方法访问
- scala中可以通过`private[类名/包名/this]`和`protected[类名/包名/this]`实现细粒度的权限控制,具体不表了...

## 第4章 处理对象
- 定义类的时候,`class A(var a:Int, val b:Int)`被称作主构造器(primary constructor),其中可变的参数a自动生成getter和setter,不可变的参数b自动生成getter方法,但这些getter setter方法不符合JavaBean标准,没有get/set前缀,可以通过在期望产生符合JavaBean规范的字段加上`@scala.reflect.BeanProperty`注解来解决这个问题
- scala中val修饰的属性编译后为`private final`
- 类中单独的代码会作为主构造器的一部分
- 还可以定义辅助构造器:`def this(...)`,辅助构造器的第一行必须调用主构造器或者其他辅助构造器
- 重载方法时强制使用关键字`override`(不是注解),如`override def f(...) = ...`
- 派生类在主构造器(与基类同名的)参数前面加上要关键字`override`, 编译器将这些属性的getter方法路由到基类对应方法
- 只有object没有对应class叫单例对象(不能传递参数给其构造器),class对应object叫伴生对象
- 伴生对象可以访问private修饰的方法
- 字节码层面上,单例中的方法会被编译为static方法
- scala创建枚举需要扩展`Enumeration`类,并用特殊的Value(表示枚举的类型)来赋值,如:  

```scala
object Currency extends Enumeration {
  type Currency = Value //高速编译器,将Currency视作一个类型而非实例
  val CNY,USD,JPY = Value
}
```
- scala支持包对象,为单例对象,与包同名,用`package object`关键字标记,当包中其他类`import 包名._`时,可以直接引用包对象里面的方法;包对象可以存储该包公用的一些方法如工具方法,举例, scala包也有包对象,包含了很多类型别名和隐式类型转换.

## 第5章 善用类型
- scala偏向于使用类型推断,但以下情况必须显式指定类型:1.定义没有初始值的类字段;2.定义函数或方法的参数;3.使用return或递归时,定义函数或方法返回类型;4.将变量定义为与推断出来类型不一样
- `Nothing`是所有类型的子类型, `Any`是所有类型的基础类型,包含`AnyRef`子类型(所有引用类型的基础类型,包含Java的`Object`类的方法)和`AnyVal`子类型(所有值类型Int,Double等的基础类型,映射到Java原始类型)
- 使用有类型参数的类但不指定泛型类型的时候,就会使用Nothing作为类型参数,如果没有定义协变,那么任何有意义的类型都不能使用
- 抛出异常的表达式的返回类型也是`Nothing`,可以替代任意类型,语义上是合法的,可以是类型推断进行下去
- `Option[T]`有两个子类,`Some[T]`(有值)和`None`(没有值),用于值可能存在或不存在的情况
- `Either`有两个子类,`Right[R]`(正确结果的值)和`Left[L]`(错误的时候的值),用于结果可能存在两种不同类型的值的情况
- 定义函数的时候,用等号将方法声明和方法主题分开(如`def f(a:Int) = {...}`)才能完成返回值类型推断,反之(如`def f(a:Int) {...}`)不行
- 一个方法只是字段或属性的访问器,不要将`()`放在定义中,调用的时候也不用`()`;但如果一个函数具有副作用,那么在声明和调用这个函数的时候都要使用`()`
- 任何返回`Unit`的函数必须产生副作用(否则,又不返回东西,又不产生副作用,那这个函数没什么用了)
- `T <: P`表示类型T派生自类型P, `T >: S`表示类型T是类型S的超类
- `[+T]`表示支持`协变`(若接受基类实例集合,则也支持子类实例集合);`[-T]`表示支持`逆变`(若接受基类实例集合,则也支持超类实例集合)
- 使用隐式转换函数时,需要导入`scala.language.implicitConversions`(主要是提醒阅读代码的人,黑魔法出没)
- 与隐式参数类似,如果一个函数标记为`implicit`,且存在于当前作用域(import或在当前文件作用域),scala都会自动使用这个函数进行类型转换(如果可以)
- scala还支持隐式类(用`implicit`标记类),但不能是独立的类,必须在单例对象/类/特质中;而使用隐式类的时候不需要导入`implicitConversions`;例如:  

```scala
object MyUtil{
  implicit class Wrapper(i:Int) {
    def conv(unit: String): String = i.toString + unit
  }
}

object MyUtilTest{
  import MyUtil._
  val s = 11 conv "cm"
  println(s) //调用这个对象的时候打印:11cm,仅作示例
}
```
- 隐式转换的时候会创建短生命周期的垃圾隐式对象,增加GC压力,而值对象可以解决这个问题(将隐式类示例上的调用自动改写成扩展方法),将隐式类继承`AnyVal`,如上面的例子改成:  

```scala
object MyUtil{
  implicit class Wrapper(i:Int) extends AnyVal{
    def conv(unit: String): String = i.toString + unit
  }
}
```
- 值类型还可以用在其他简单值/原始值已经够用,但是希望使用类型进行更好抽象的地方:最终源码中显示是类/示例,字节码级别上是基础类型
- 自定义字符串插值器: 定义一个隐式类,主构造器接受一个`StringContext`类型的参数,定义方法`name(args: Any*):StringBuilder`,那么当程序作用域包含该隐式类的时候,对`name"""..."""`的字符串,会自动创建`StringContext`对象(其`parts`方法可以获取字符串中被表达式分割的各个子字符串),传入隐式转换的的`name(Any*)`方法,参数传入字符串中的各个`${expression}`,处理完的`StringBuilder`对象返回,然后转换成字符串. 例如一个简单的例子,所有表达式前后加上井号:  

```scala
object MyInterpolator{
  implicit class Interpolator(val context:StringContext) extends AnyVal {
    def my(args: Any*):StringBuilder = {
      new StringBuilder(context.parts.zip(args).map(item => {
        val (text, expression) = item
        s"$text#${expression}#"
      }).mkString)
    }
  }
}

import MyInterpolator._
val name = "Leibniz"
println(my"""My name is ${name}""") //调用我们自定义的插值器,返回 My name is #Leibniz#
```

## 第6章 函数值与闭包
- 柯里化: 一个有分组参数的函数,如`f(a:A)(b:B):C`,使用`f _`创建一个部分应用函数(此处类型为`A => (B => C)`),可以用于创建可复用的临时便利函数;
- 多组参数的函数,如果有单独成组的函数参数,可以不使用小括号,直接用大括号,更直观,如`f(a:Int)(g:A=>B)`可以这样调用:`f(1) {a => xxx(a)}`
- 用下划线代表函数值的参数时,如果无法判断类型,scala会报错,此时可以显式指定类型
- scala自动将函数名视作函数值的引用
- 函数或代码块可能含有未绑定的变量,在调用前根据上下文绑定,形成闭包(closure);绑定的时候不会复制相应变量的值,实际上会绑定到变量本身,因此线程不安全

## 第7章 特质
- 在trait中定义并初始化的val/var变量，将会在混入了该trait的累的内部实现；定义并**未**初始化的val/var变量被认为是抽象的,混入该trait的类需要实现他们
- 类混入trait的时候,如果类没用用`extends`,则第一个`trait`用`extends`,后面的`trait`用`with`;如果类已经用`extends`了,那么所有`trait`都用`with`来混入
- 混入了trait的类可以调用trait的方法, 其实例引用也可以多态为trait实例
- trait的构造器不接收任何参数
- 选择性混入: 可以对没有混入trait的类的实例混入trait,即只有该实例混入了trait,该类其他实例没有,也是用`with`进行混入,如:  

```scala
val angle = new Cat("Angle") with Friend`
```
- 在trait中,使用super调用的方法会触发延迟绑定(late binding),此时并非对基类方法的调用,而是转发到混入该trait的类中,如果混入了多个这样的trait(有同样父类方法,即extends自同一个trait或抽象类),那么从右向左,混入下一个trait直到混入到类, 例如:  

```scala
abstract class Check {def check:String = "Abstract check..."}
trait CreditCheck extends Check {override def check:Stirng = s"Credit...${super.check}"} //并非调用父类方法,而是转发到下一个trait或类
trait MoneyCheck extends Check {override def check:Stirng = s"Money...${super.check}"} 
trait HouseCheck extends Check {override def check:Stirng = s"House...${super.check}"}
val app1 = new Check with MoneyCheck with CreditCheck
println(app1.check) //根据混入的顺序,打印 Credit...Money...Abstract check...
val app2 = new Check with CreditCheck with HouseCheck
println(app2.check) //根据混入的顺序,打印 House...Credit...Abstract check...
```
- 如果基类的方法是抽象的,那么方法绑定必须推迟到某个具体的方法已知为止;同样是从右到左的顺序连接混入trait,这样可以避免方法冲突的问题
