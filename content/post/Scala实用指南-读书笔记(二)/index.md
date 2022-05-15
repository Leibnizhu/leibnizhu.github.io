---
title: 《Scala实用指南》读书笔记(二)
date: 2018-07-18T16:54:47+08:00
tags:
- scala
image: nightstreet.jpg
---
# 第8章 集合
- Scala对元素较少的Set进行了优化,4个元素以内的Set有专门的实现类(`Set0`,`Set1`,`Set2`,`Set3`,`Set4`),大于4个元素的使用HashSet实现
- Set的方法(`filter`,`map`,`foreach`这些就不说了):  
  - `mkString`: 允许传入一个分隔符,相当于把集合所有元素用分隔符join起来返回字符串
  - `++`: 合并两个Set,或者说,并集
  - `&`: 两个Set的交集
- Map的方法:  
  - `filterKeys`: 根据key去过滤,而`filter`方法是根据(key,value)键值对去过滤
  - `get`: 根据key拿value,注意是返回`Option[T]`
  - `updated(K,V)`: 增加或更新键值对,返回新的Map,也可以用`X() = b`,等效于`X.updated(b)`
- List的`a :: list`读作`将a前插到list`,后面的list才是::方法的调用者; 而`list1 ::: list2`将list1前插到list2
- List的`forall()`方法判断是否所有元素都满足条件, `exists()`方法判断是否有任意元素满足条件. 其实分别相当于:

```scala
list.forall(f) 等效于 (true /: list) {_ && f(_)}
list.exists(f) 等效于 (false /: list) {_ || f(_)}
```
- 如果方法名以 `:` 结尾, 那么调用时的主语是操作符后面的实例, 即`a (操作符): b`等效于`b.操作符:(a)`;同时scala不允许字母作为操作符的名称,除非用下划线对操作符增加前缀,如`jump_:()`
- ` + - ! ~`作一元操作符时,也是调用的主语在操作符后面,分别映射到`unary_+(),unary_-(),unary_!(),unary_~()`等方法的调用,即`-a`调用`a.unary_-()`等
- for表达式:`for([parrten <- generator; definition*>]+;filter;) [yield] expression`,可以加过滤条件,而yield也是可选的,有yield的时候返回一个值列表,没有yield的时候返回Unit

# 第9章 模式匹配和正则表达式
- 匹配List的时候,可以只获取感兴趣的元素,剩下的用`_*`省略, 如:  

```scala
list match {
  case List(head,"test", _*) => f(head) //后面的直接忽略
  case List("test", others @ _*) => f(others) //后面的tail要引用
  case "haha"::"two"::tail => f(tail) //这样其实也行
}
```
- scala的模式匹配case子句无需break
- scala约定模式变量名以小写字母开头(scala假设他是模式变量),常量为大写字母(会在作用域范围查找变量)
- 不遵守以上规则的时候, 比如需要用模式匹配以外的变量, 可以显式指定作用域(如`this.max`),或用反引号 `` ` `` 包住变量名(如`` `max` ``)
- case类用于创建轻量级值对象,经常用于模式匹配；如果主构造器无参数，调用时又没加括号，那么传递的是case类的伴生对象（混合了Function0特质，可以视为函数）
- 可使用自定义的提取器进行模式匹配，提取器在伴生对象中有`unapply()`方法,接受我们想要匹配的值,返回Boolean(传入的值是否可以匹配),例如:  

```scala
object Symbol {
  def unapply(s:String):Boolean = s == "TEST"
}
input match {
  case Symbol() => println(s"matched, ${input}") //即匹配 TEST
  case _ => println(s"Invalid input: ${input}")
}
```
- 提取器还可以返回Boolean型以外的结果,即解析的结果,通过修改`unapply()`的返回类型实现(返回`Option[T]`,`T`即解析成功的结果类型), 例如:  

```scala
object Splitor {
  def unapply(s:String):Option[(String,String)] = if(!s.contains(":")) None else {
    val splited = s.split(":")
    Some((splited(0), splited(1)))
  }
}
input match {
  case Splitor(a,b) => println(s"matched, ${a} and ${b}") //即匹配 ***:***
  case _ => println(s"Invalid input: ${input}")
}
```
- 使用提取器的时候还可以应用其他提取器进行模式匹配,如

```scala
input match {
  case Splitor(a @ Symbol(),b) => println(s"matched, ${a} and ${b}") //即匹配 TEST:***
  case _ => println(s"Invalid input: ${input}")
}
```
- 正则表达式: `"regex".r`(可以用原始字符串`"""regex""".r`), 实际上是`String`隐式转换成`StringOps`再调用其`r`方法获取`Regex`类实例
- 正则表达式的方法:  
  - `findFirstIn(source)`: 获取正则表达式第一个匹配项
  - `findAllIn(source)`: 获取正则表达式的所有匹配项
  - `replaceFirstIn(source, replacement)`: 替换第一个匹配项
  - `replaceAllIn(source, replacement)`: 替换所有匹配项
- scala的正则表达式是提取器,返回值是匹配项(括号的分组)拼接成的元组
- 下划线的作用:  
  - 包引入的通配符
  - 元组索引的前缀
  - 函数值的隐式参数
  - 用默认值初始化变量
  - 在函数名中混合操作符和:
  - 在模式匹配中作为通配符
  - 处理异常时在catch代码块和case一起用(类似模式匹配了)
  - 作为分解操作的一部分,如`max(arg: _*)`可以接受列表或数组参数,自动拆解成离散的值传递给可变长度参数
  - 部分应用一个函数,如`val square = Math.pow(_:Int, 2)`部分应用了pow函数,返回一个新的单参数函数

# 第10章 处理异常
- Java处理多个异常时,会检查多个异常的处理顺序,子类必须在前面,否则编译会出错(`exception ***.*** has already been caught`),但scala对此不会警告,要自己注意(在catch块中使用case匹配的顺序)
- 这本书竟然没有讲到`Try`类,吐槽一下

# 第11章 递归
- 尾递归优化(Tail Recursive Optimization): 不是尾递归的时候,递归调用在字节码对应`invokespecial`指令,表明是递归调用,会产生新一层栈;如果写成尾递归, 递归调用时在字节码对应`goto`指令,表明使用了迭代而非方法调用,放弃了当前的上下文
- `@scala.annotation.tailrec`注解加载函数上,可以让scala检查是否使用了尾递归,如果非尾递归,会报错;该注解可选,主要是增加可读性,并在重构时保持尾递归性质
- 蹦床调用(trampoline call): 两个函数互相调用(f调用g,g调用f)构成递归, 对于蹦床调用即使是尾递归`@tailrec`注解也会报错(scala不能识别跨方法的递归);此时可以用`TailRec`类解决:  
  - 蹦床调用的函数返回`TailRec[T]`,其中`T`是真正的返回值类型
  - 蹦床调用的函数内部,需要结束递归时返回`done(结果: T)`,需要递归调用其他函数时返回`tailcall(其他函数())`;这两者都只是简单包装参数,以供后续调用或延迟执行
  - 外部调用这些函数时, 对返回值`TailRec[T]`调用`result`方法可以获取最终递归结果;真正发生计算是调用`result`方法的时候

# 第12章 惰性求值和并行集合
- 使用关键字`lazy`修饰变量,scala会推迟绑定变量和他的值,直到该值第一次被使用(才会去绑定)
- 如果变量绑定的计算有副作用,那么多个变量的绑定顺序机会对绑定的值有影响,此时就不能随便用惰性求值(`lazy`)了,否则不可交换计算的结果将会得不可知
- 前面介绍的集合都是`严格集合`,所有计算都是严格(立刻)执行的; 通过集合的`view()`方法可以获得一个严格集合的`惰性视图`, 惰性集合会推迟计算,当且晋档请求了非惰性/非视图的结果时(比如`head`,`last`等等)前面的操作才会进行
- 但集合的惰性视图不一定比严格集合性能好,要看具体情况,比如一个集合进行一些操作之后,获取head,那么惰性视图要进行操作的次数就少一些;如果在多次filter后要拿last,那么惰性视图就要把每个元素执行filter计算一遍, 而严格集合每次filter后要计算的结果都小一些,这样计算量反而会少一点.
- `Stream`仅按需生成值,有天然的惰性.拥有`#::`方法连接(惰性,需要的时候才会连接)现有的Stream和新的值,通过递归定义可以得到一个`Stream`, 如:  

```scala
//第一个元素是start,下一个是前一个加1
def gen(start:Int): Stream[Int] = start #:: gen(start + 1) 
println(gen(10)) //Stream(10, ?)
```
- 调用`Stream.force()`方法可以强制求值, `toList()`方法也类似;如果在无限流上调用的话会抛OutOfMemoryError
- 对于无限流,可以使用`take(Int)`方法获取前N个值组成的Stream,也可以使用`takeWhile()`方法按条件生成值(参数的函数值返回false时终止生成新值)
- Stream会记住(memoize)已经生成的值,即按需生成新的值后,会先缓存再返回. 比如执行`stream.take(3).force`计算了3个值,再执行`stream.take(4).force`只计算第4个值,前3个值从缓存读取
- 对很多顺序集合,scala都有对应的并行版本, 如`ParArray`,`ParHashMap`,`ParHashSet`等等; 可以用`par()`和`seq()`方法在顺序集合及其并行版本之间转换
- 不适合使用并行集合的情况:  
  - 创建和调度线程的开销不应该大于执行这些任务所需要的时间, 对于慢型任务而言并行集合可能有所裨益,但对于小心集合的快速任务并不适合; 
  - 此外,在集合上的操作如果修改全局状态(线程不安全的修改),那么整体计算结果不可知; 
  - 如果操作不满足结合律也不要使用并行集合,因为并行集合的执行顺序是不确定的

# 第13章 使用Actor编程
- Actor帮助我们将共享的可变性转换成隔离的可变性(isolated mutability),如果一个任务可以有意义地分成几个子任务,分而治之,那么可以使用Actor模型来解决这个任务
- `AtomicLog`之类的类,虽然原子性保证了单个值的线程安全性,但并不能保证跨多个值的原子性,这些值可能同时发生变化
- 一个Actor是一个对象,由一个消息队列支撑,任意给定的时间,一个Actor只会处理一条消息; `Akka`提供`Actor`模型, 创建一个Actor只要继承`Actor`特质并实现`receive()`方法, `receive()`方法主题是模式匹配, 匹配发生在一个隐式消息对象上
- Actor托管在ActorSystem中,管理了线程池(只要系统保持活跃,这个线程池就会一直保持活跃),消息队列,和Actor生命周期,使用`actorOf()`工厂方法创建Actor, 用`!()`方法(`tell()`)发送消息:  

```scala
val system = ActorSystem("sample") //创建ActorSystem
val depp = system.actorOf(Props[XxxActor]) //通过actorOd工厂方法创建Actor
depp ! "Hello" //往Actor发送消息
val terminateFuture = system.terminate() //退出ActorSystem线程
Await.ready(terminateFuture, Duration.Inf)
```
- Actor一些细节:  
  - Actor在不同线程中进行,而不是调用主线程
  - 每个Actor一次只处理一条消息,多个Actor并发运行处理多条消息
  - Actor是异步的,不会阻塞调用者(调用者不等待Actor回复)
  - 线程和Actor不绑定(没什么线程亲和力),每次处理消息**可能**使用线程池中不同的线程
- Actor中保存的任何字段都是自动线程安全的,可变但没有共享可变性;可以在Actor类中选择性地存储状态,比如用于存储状态的字段,等等
- 若希望从Actor得到响应,Akka提供了询问模式,但消息可能永远不会到达,因此强制使用超时时间;询问模式下, 使用`?()`方法(`ask()`)发送消息, 返回一个`Future`,需要用这个Future实例等待响应(可以用`Await.result()`方法), 例如:  

```scala
//Actor类
class MyActor extends Actor {
  def recieve:Receive = {
    case msg => sender ! s"Got message ${msg}"
  }
}

val system = ActorSystem("sample") //创建ActorSystem
val depp = system.actorOf(Props[MyActor]) //通过actorOd工厂方法创建Actor
implicit val timeout = Timeout(2.seconds) //通过隐式参数定义超时, ?方法要用到
val askFuture = depp ? "heihei" //询问模式,发出消息
val result = Await.result(askFuture, timeout.duration) //等待响应结果,这里也需要一个超时的参数
println(s"Response result : ${result}")
val terminateFuture = system.terminate() //退出ActorSystem线程
Await.ready(terminateFuture, Duration.Inf)
```
- Akka提供`RoundRobinPool`路由器,会将发送到这个路由器的所有消息均匀地路由到支撑他的多个Actor,使用方法:  

```scala
val system = ActorSystem("sample") //创建ActorSystem
val router:ActorRef = system.actorOf(RoundRobinPool(100).props(Props[MyActor]))
```
- Actor中想访问ActorSystem可以使用`context()`方法
- 使用建议:  
  - 更多地依赖无状态的而非有状态的Actor
  - 要保证recieve()方法中的处理速度非常狂,尤其是接收Actor具有状态时,改变状态的长时间运行任务将会降低并发性,要避免
  - 确保在Actor之间传递的消息是不可变对象
  - 避免使用ask()

# 第14章 和Java进行互操作
- scala使用Java类,当Java变量/方法名等于scala关键字冲突的时候,可以将受影响的变量/方法用反引号`` ` ``括起来
- 没有方法实现的trait在字节码层面是简单的接口;如果想在scala中创建接口,只能创建没有方法实现的trait
- 有方法实现的trait, 2.11及更早版本的scala会编译成一个接口(名为:`trait名`)和一个实现该接口的抽象类(名为:`trait名$class`), 2.12开始,只包含方法实现而不包含字段的trait会编译成带有默认方法的接口
- scala将单例对象和伴生对象编译成一个单例类,在字节码层面只有static方法
- 单例对象(假设名为Single)编译后产生两个类, 一个`Single$`类存放具体的静态方法,一个`Single`类负责转发方法的调用,在Java中可以直接通过`Single`调用其方法
- 伴生对象(假设类名为Buddy)编译后产生两个类,一个`Buddy$`类存放伴生对象的静态方法,一个`Buddy`类存放对应类的方法等;
- 在Java中使用类本身的方法可以直接调用;而伴生对象,包含一个`MODULE$`的属性保存着该类的静态单例对象,可以通过它去访问其方法,如`Buddy$.MODULE$.method()`
- scala没有throws子句,抛出异常时无需在方法签名中显示声明,如果在Java中扩展类覆写方法时抛出异常就会报错; 可以在scala方法签名中增加`@throw`注解解决该问题,如:  

```scala
abstract class Bird {
  @throws(classOf[NoFlyException]) def fly(): Unit
}
```

(结束)