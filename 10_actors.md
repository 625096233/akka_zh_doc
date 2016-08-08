# 10.actors
Actor模型为编写并发和分布式系统提供了一种更高的抽象级别。它将开发人员从显式地处理锁和线程管理的工作中解脱出来，使编写并发和并行系统更加容易。Actor模型是在1973年Carl Hewitt的论文中提的，但是被Erlang语言采用后才变得流行起来，一个成功案例是爱立信使用Erlang非常成功地创建了高并发的可靠的电信系统。
Akka Actor的API与Scala Actor类似，并且从Erlang中借用了一些语法。

## 10.1 创建Actor
> 由于Akka采用强制性的父子监管，每一个actor都被监管着，并且（会）监管它的子actor们；我们建议你熟悉一下Actor系统 和 监管与监控，阅读 总结:actorOf vs. actorFor 也有帮助。

### 10.1.1 定义一个Actor类
actor在Java中是通过扩展UntypedActor类和实现onReceive方法来实现的。该方法以消息作为参数。
下面看一个实例:
```java
import akka.actor.UntypedActor;
import akka.event.Logging;
import akka.event.LoggingAdapter;
 
public class MyUntypedActor extends UntypedActor {
  LoggingAdapter log = Logging.getLogger(getContext().system(), this);
 
  public void onReceive(Object message) throws Exception {
    if (message instanceof String) {
      log.info("Received String message: {}", message);
      getSender().tell(message, getSelf());
    } else
      unhandled(message);
  }
}
```

### 10.1.2 Props
Props是一个用来在创建actor时指定选项的配置类。 以下是使用如何创建Props实例的示例。 首先需要导入akka.actor.Props和akka.japi.Creator.
```java
import akka.actor.Props;
import akka.japi.Creator;
```
创建Props对象的方法有两种,一种是通过构造参数创建一个actor.如:props1和props2通过匹配actor的构造函数来创建Props的对象,如果没有发现或匹配了多个构造函数抛出一个IllegalArgumentException;
第二种是通过akka.japi.Creator实现类来创建,如props3通过使用akka.japi.Creator接口实现类来创建Props对象.
```java
static class MyActorC implements Creator<MyActor> {
  @Override public MyActor create() {
    return new MyActor("...");
  }
}
 
  Props props1 = Props.create(MyUntypedActor.class);
  Props props2 = Props.create(MyActor.class, "...");
  Props props3 = Props.create(new MyActorC());
```

下面Creator的使用.creator类也许是一个静态类,这个类会在Props创建的时候校验.泛型是用来确定生产actor类,如果不使用泛型,需要填写Actor。实例:
```java
static class ParametricCreator<T extends MyActor> implements Creator<T> {
  @Override public T create() {
    // ... fabricate actor here
  }
}
```


#### 10.1.2.1 建议实践
提供在UntypedActor中提供静态工厂方法来创建一套Props尽可能使actor一致是一个好主意.
```java
public class DemoActor extends UntypedActor {
  
  /**
   * Create Props for an actor of this type.
   * @param magicNumber The magic number to be passed to this actor’s constructor.
   * @return a Props for creating this actor, which can then be further configured
   *         (e.g. calling `.withDispatcher()` on it)
   */
  public static Props props(final int magicNumber) {
    return Props.create(new Creator<DemoActor>() {
      private static final long serialVersionUID = 1L;
 
      @Override
      public DemoActor create() throws Exception {
        return new DemoActor(magicNumber);
      }
    });
  }
  
  final int magicNumber;
 
  public DemoActor(int magicNumber) {
    this.magicNumber = magicNumber;
  }
  
  @Override
  public void onReceive(Object msg) {
    // some behavior here
  }
  
}
 
  system.actorOf(DemoActor.props(42), "demo");
```
另一个好的实践是通过声明actor的消息类(例如:在Actor内部声明静态消息类,或者使用其他合适的类),让接受方能更容易知道接受的是什么信息.
```java
public class DemoMessagesActor extends UntypedActor {
 
  static public class Greeting {
    private final String from;
 
    public Greeting(String from) {
      this.from = from;
    }
 
    public String getGreeter() {
      return from;
    }
  }
 
  public void onReceive(Object message) throws Exception {
    if (message instanceof Greeting) {
      getSender().tell("Hello " + ((Greeting) message).getGreeter(), getSelf());
    } else
      unhandled(message);
  }
}
```

###10.1.3 使用Props创建Actor
Actor可以通过将 Props 实例传入 ActorSystem和ActorContext的actorOf 工厂方法来创建。
```java
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
```
```java
// ActorSystem is a heavy object: create only one per application
final ActorSystem system = ActorSystem.create("MySystem");
final ActorRef myActor = system.actorOf(Props.create(MyUntypedActor.class),"myactor");
```
使用ActorSystem将创建顶级的actor,由actor系统提供监管actor来监控,使用actor的上下文来创建actor。
```java
class A extends UntypedActor {
  final ActorRef child =
      getContext().actorOf(Props.create(MyUntypedActor.class), "myChild");
  // plus some behavior ...
}
```

推荐创建一个包含子、孙子等等的层次结构，使之符合应用的逻辑错误处理结构，见[Actor系统](./02_actor_system.md) 。

对actorOf的调用返回一个ActorRef实例。ActorRef是actor实例的句柄，并且是与actor实例进行交互的唯一方法。ActorRef是不可变的并且与actor是一一对应的关系。ActorRef也可序列化，并能通过网络传输。这意味着你可以将其序列化，通过网线发送，并在远程主机上使用它，而且虽然跨网络，它将仍然代表在原始节点上相同的actor。

actor的名称参数是可选的，不过你应该给actor起个好名字，因为名称将被用于日志打印和标识actor。名称不能为空，且不能以$开头，不过它可以包含URL编码字符（如空格用%20表示）。如果给定的名称已经被同一个父亲下的另一个子actor使用，则会抛出一个InvalidActorNameException。

当actor创建的时候,它会自动异步启动。

##10.1.4 依赖注入
如果你的UntypedActor有带参数的构造函数，则这些参数也需要成为Props的一部分，如上文所述。
但有些情况下必须使用工厂方法，例如，当实际构造函数的参数由依赖注入框架决定。
```java
import akka.actor.Actor;
import akka.actor.IndirectActorProducer;
```
```java
class DependencyInjector implements IndirectActorProducer {
  final Object applicationContext;
  final String beanName;
  
  public DependencyInjector(Object applicationContext, String beanName) {
    this.applicationContext = applicationContext;
    this.beanName = beanName;
  }
  
  @Override
  public Class<? extends Actor> actorClass() {
    return MyActor.class;
  }
  
  @Override
  public MyActor produce() {
    MyActor result;
    // obtain fresh Actor instance from DI framework ...
    return result;
  }
}
  
  final ActorRef myActor = getContext().actorOf(
    Props.create(DependencyInjector.class, applicationContext, "MyActor"),
      "myactor3");
```
>你有时可能倾向于提供一个IndirectActorProducer始终返回相同的实例，例如通过使用静态属性。
这是不受支持的，因为它违背了actor重启的意义。当使用依赖注入框架时，actor bean必须不是单例作用域。


关于依赖注入及其Akka集成的更多内容情参见“在Akka中使用依赖注入”指南和Typesafe Activator的“Akka Java Spring”教程。


###10.1.5 收件箱
当编写代码在actor之外让actor进行通信,询问匹配是一个解决方案,但是有两件事情不能做到,一是接受多个回复(例如:通过描述一个ActorRef
到一个通知服务),二是监控其他actor的生命周期.Inbox类可以满足需求:
```java
final Inbox inbox = Inbox.create(system);
inbox.send(target, "hello");
try {
  assert inbox.receive(Duration.create(1, TimeUnit.SECONDS)).equals("world");
} catch (java.util.concurrent.TimeoutException e) {
  // timeout
}
```
send方法包装了一个普通tell方法,提供内部的到sender发送方的actor引用.这允许在最后一行接收回复.监控一个actor就是如此简单:
```java
final Inbox inbox = Inbox.create(system);
inbox.watch(target);
target.tell(PoisonPill.getInstance(), ActorRef.noSender());
try {
  assert inbox.receive(Duration.create(1, TimeUnit.SECONDS)) instanceof Terminated;
} catch (java.util.concurrent.TimeoutException e) {
  // timeout
}
```

##10.2 UntypedActor API
UntypedActor只定义了一个抽象方法，就是上面提到的receive, 用来实现actor的行为。

如果当前actor的行为与收到的消息不匹配，建议调用unhandled方法, 它的缺省实现是向actor系统的事件流中发布一条`new akka.actor.UnhandledMessage(message, sender, recipient)`,设置配置项akka.actor.debug.unhandled打开,将它们转换成实际的调试消息。

另外，它还包括:

- self代表本actor的ActorRef
- sender代表最近收到的消息的发送actor，通常用于下面将讲到的回应消息中
- supervisorStrategy用户可重写它来定义对子actor的监管策略
- context暴露actor和当前消息的上下文信息，如：
    - 用于创建子actor的工厂方法 (actorOf)
    - actor所属的系统
    - 父监管者
    - 所监管的子actor
    - 生命周期监控
    - hotswap行为栈，见 Become/Unbecome

剩余的可见方法user-overridable生命周期钩子在以下描述:
```java
public void preStart() {
}
 
public void preRestart(Throwable reason, scala.Option<Object> message) {
  for (ActorRef each : getContext().getChildren()) {
    getContext().unwatch(each);
    getContext().stop(each);
  }
  postStop();
}
 
public void postRestart(Throwable reason) {
  preStart();
}
 
public void postStop() {
}
```

###10.2.1 actor生命周期

###10.2.2 使用DeathWatch进行生命周期监控
为了在其它actor结束时 (i.e. 永久终止, 而不是临时的失败和重启)收到通知, actor可以将自己注册为其它actor在终止时所发布的 Terminated 消息 的接收者 (见 停止 Actor). 这个服务是由actor系统的DeathWatch组件提供的。

注册一个监控器很简单：

