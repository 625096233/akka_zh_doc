# actors
Actor模型为编写并发和分布式系统提供了一种更高的抽象级别。它将开发人员从显式地处理锁和线程管理的工作中解脱出来，使编写并发和并行系统更加容易。Actor模型是在1973年Carl Hewitt的论文中提的，但是被Erlang语言采用后才变得流行起来，一个成功案例是爱立信使用Erlang非常成功地创建了高并发的可靠的电信系统。
Akka Actor的API与Scala Actor类似，并且从Erlang中借用了一些语法。

## 创建Actor
> 由于Akka采用强制性的父子监管，每一个actor都被监管着，并且（会）监管它的子actor们；我们建议你熟悉一下Actor系统 和 监管与监控，阅读 总结:actorOf vs. actorFor 也有帮助。

### 定义一个Actor类
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

### Props
Props是一个用来在创建actor时指定选项的配置类。 以下是使用如何创建Props实例的示例。 
```
import akka.actor.Props;
import akka.japi.Creator;
```

```
static class MyActorC implements Creator<MyActor> {
  @Override public MyActor create() {
    return new MyActor("...");
  }
}
 
  Props props1 = Props.create(MyUntypedActor.class);
  Props props2 = Props.create(MyActor.class, "...");
  Props props3 = Props.create(new MyActorC());
```
第二行展示如何通过构造参数创建一个actor.施工中存在一个匹配的构造函数验证Props的对象,如果没有发现或多了匹配的构造函数抛出一个IllegalArgumentException。

第三行演示了Creator的使用.creator类也许是一个静态类,这个类会在Props创建的时候校验.泛型是用来确定生产actor类,如果不使用泛型,需要填写Actor。实例:
```
static class ParametricCreator<T extends MyActor> implements Creator<T> {
  @Override public T create() {
    // ... fabricate actor here
  }
}
```


#### 建议实践
提供在UntypedActor中提供静态工厂方法来创建一套Props尽可能使actor一致是一个好主意.
```
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
```
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

###使用Props创建Actor
Actor可以通过将 Props 实例传入 ActorSystem和ActorContext的actorOf 工厂方法来创建。
```
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
```
```
// ActorSystem is a heavy object: create only one per application
final ActorSystem system = ActorSystem.create("MySystem");
final ActorRef myActor = system.actorOf(Props.create(MyUntypedActor.class),"myactor");
```
使用ActorSystem将创建顶级的actor,由actor系统提供监管actor来监控,使用actor的上下文来创建actor。
```
class A extends UntypedActor {
  final ActorRef child =
      getContext().actorOf(Props.create(MyUntypedActor.class), "myChild");
  // plus some behavior ...
}
```

推荐创建一个包含子actor、孙子等等的层次结构，使之符合应用的逻辑错误处理结构，见Actor系统。

对actorOf的调用返回一个ActorRef实例。它是actor实例的句柄，并且是与之进行交互的唯一方法。ActorRef是不可变的，与其代表的actor有一一对应关系。ActorRef也可序列化，并能通过网络传输。这意味着你可以将其序列化，通过网线发送，并在远程主机上使用它，而且虽然跨网络，它将仍然代表在原始节点上相同的actor。

名称参数是可选的，不过你应该良好命名你的actor，因为名称将被用于日志打印和标识actor。名称不能为空，且不能以$开头，不过它可以包含URL编码字符（如空格用%20表示）。如果给定的名称已经被同一个父亲下的另一个子actor使用，则会抛出一个InvalidActorNameException。

actor在创建时，会自动异步启动。

##依赖注入
如果你的UntypedActor有带参数的构造函数，则这些参数也需要成为Props的一部分，如上文所述。
但有些情况下必须使用工厂方法，例如，当实际构造函数的参数由依赖注入框架决定。
```
import akka.actor.Actor;
import akka.actor.IndirectActorProducer;
```
```
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


##收件箱
当编写代码在actor之外让actor进行通信,询问匹配是一个解决方案,但是有2件事情不能做到,接受多个回复(例如:通过描述一个ActorRef
到一个通知服务),监控其他actor的生命周期.Inbox类可以满足需求:
```
final Inbox inbox = Inbox.create(system);
inbox.send(target, "hello");
try {
  assert inbox.receive(Duration.create(1, TimeUnit.SECONDS)).equals("world");
} catch (java.util.concurrent.TimeoutException e) {
  // timeout
}
```
send方法包装了一个普通tell方法,提供内部的到sender发送方的actor引用.这允许在最后一行接收
回复.监控一个actor就是如此简单:
```
final Inbox inbox = Inbox.create(system);
inbox.watch(target);
target.tell(PoisonPill.getInstance(), ActorRef.noSender());
try {
  assert inbox.receive(Duration.create(1, TimeUnit.SECONDS)) instanceof Terminated;
} catch (java.util.concurrent.TimeoutException e) {
  // timeout
}
```







