> 文档版本:2.4.9 RC1

#1.环境
akka需要java8以上版本.

#2.入门指导和模板项目
学习akka最好的途径是下载 [Lightbend Activator](http://www.lightbend.com/activator/download) 并实验一个akka模板项目.

#3.模块
akka非常的模块化,是由包含不同特性的jar文件组成.

- akka-actor – Classic Actors, Typed Actors, IO Actor etc.
- akka-agent – Agents, integrated with Scala STM
- akka-camel – Apache Camel integration
- akka-cluster – Cluster membership management, elastic routers.
- akka-osgi – utilities for using Akka in OSGi containers
- akka-osgi-aries – Aries blueprint for provisioning actor systems
- akka-remote – Remote Actors
- akka-slf4j – SLF4J Logger (event bus listener)
- akka-testkit – Toolkit for testing Actor systems

#4.使用akka(maven)
通过maven构建akka项目最简单的方法就是下载 [Lightbend Activator](http://www.lightbend.com/activator/download)的叫做 [Akka Main in Java](http://www.lightbend.com/activator/template/akka-sample-main-java?_ga=1.45605284.1226551537.1468204219) 的教程.

从2.1-M2版本开始akka被发布到maven中央仓库,下面是akka-actor依赖实例:
```
<dependency>
  <groupId>com.typesafe.akka</groupId>
  <artifactId>akka-actor_2.11</artifactId>
  <version>2.4.9-RC1</version>
</dependency>
```
如果需要使用snapshot版本,还需要添加snapshot库:
```
<repositories>
  <repository>
    <id>akka-snapshots</id>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    <url>http://repo.akka.io/snapshots/</url>
  </repository>
</repositories>
```

#5.hello world
下载 [Lightbend Activator](http://www.lightbend.com/activator/download)的叫做 [Akka Main in Java](http://www.lightbend.com/activator/template/akka-sample-main-java?_ga=1.45605284.1226551537.1468204219) 的教程.
helloworld项目pom.xml文件:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <modelVersion>4.0.0</modelVersion>
  <artifactId>akka-sample-main-java</artifactId>
  <groupId>com.typesafe.akka.samples</groupId>
  <name>Akka Main in Java</name>
  <version>1.0</version>
  
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  
  <dependencies>
    <dependency>
      <groupId>com.typesafe.akka</groupId>
      <artifactId>akka-actor_2.11</artifactId>
      <version>2.4.4</version>
    </dependency>
  </dependencies>

</project>

```

Helloworld.java
```java
package sample.hello;

import akka.actor.Props;
import akka.actor.UntypedActor;
import akka.actor.ActorRef;

public class HelloWorld extends UntypedActor {

  @Override
  public void preStart() {
    // create the greeter actor
    final ActorRef greeter = getContext().actorOf(Props.create(Greeter.class), "greeter");
    // tell it to perform the greeting
    greeter.tell(Greeter.Msg.GREET, getSelf());
  }

  @Override
  public void onReceive(Object msg) {
    if (msg == Greeter.Msg.DONE) {
      // when the greeter is done, stop this actor and with it the application
      getContext().stop(getSelf());
    } else
      unhandled(msg);
  }
}

```
Greeter.java
```java
package sample.hello;

import akka.actor.UntypedActor;

public class Greeter extends UntypedActor {

  public static enum Msg {
    GREET, DONE;
  }

  @Override
  public void onReceive(Object msg) {
    if (msg == Msg.GREET) {
      System.out.println("Hello World!");
      getSender().tell(Msg.DONE, getSelf());
    } else
      unhandled(msg);
  }

}
```

HelloWorld actor是应用的main class,当它终结时程序被关闭.主业务逻辑发生在preStart方法,Greeter actor被创建,并且等待HelloWorld发起一个问候(GREET).当HelloWorld一个问候(GREET)通过发送后端消息发送给Greeter,Greeter会通过onReceive方法接收,打印"Hello World"后,再返回一个消息(DONE)告诉HelloWorld问候一句收到,HelloWorld通过onReceive接收到这消息(DONE)后关闭HelloWorld actor.


Main.java
```java
package sample.hello;

public class Main {

  public static void main(String[] args) {
    akka.Main.main(new String[] { HelloWorld.class.getName() });
  }
}

```
Main.java实际上只是一个小包装的通用类akka.Main,它只需要一个参数:应用程序主actor的类名这主方法将创建运行actor所需的基础设施,启动给定主要actor和安排整个应用程序关闭。因此你可以使用下面的命令运行应用程序:
`java -classpath  akka.Main sample.hello.HelloWorld`

如果你需要比akka.Main更多的控制启动代码,你可以参考Main2.java编写自己的主类.
Main2.java
```java
package sample.hello;

import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;
import akka.actor.Terminated;
import akka.actor.UntypedActor;
import akka.event.Logging;
import akka.event.LoggingAdapter;

public class Main2 {

  public static void main(String[] args) {
    ActorSystem system = ActorSystem.create("Hello");
    ActorRef a = system.actorOf(Props.create(HelloWorld.class), "helloWorld");
    system.actorOf(Props.create(Terminator.class, a), "terminator");
  }

  public static class Terminator extends UntypedActor {

    private final LoggingAdapter log = Logging.getLogger(getContext().system(), this);
    private final ActorRef ref;

    public Terminator(ActorRef ref) {
      this.ref = ref;
      getContext().watch(ref);
    }

    @Override
    public void onReceive(Object msg) {
      if (msg instanceof Terminated) {
        log.info("{} has terminated, shutting down system", ref.path());
        getContext().system().terminate();
      } else {
        unhandled(msg);
      }
    }

  }
}

```













