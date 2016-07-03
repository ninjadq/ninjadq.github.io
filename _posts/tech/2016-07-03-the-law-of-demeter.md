---
layout: post
title: 【翻译】 Demeter 法则 - 编写害羞的代码
category: tech 
tags: [translation, demeter, refactoring]
---

# 【翻译】 Demeter 法则 - 编写害羞的代码

在我所有的编写服务端代码的生涯中，我已经发现项目能够长期成功的关键，并不是一个快速的算法和炫酷的框架，而是代码的复杂程度。**难以管理的复杂度，对大型项目的可维护性有深远的影响**。难以理解的应用是很难重构的。引入**新的特性是一个缓慢痛苦的过程，增加了到市场发布的时间**。我曾经见过一个系统，开发者即使想做一些小的修改都会被吓尿，因为这可能会无意间的弄坏应用的其他的部分。或者，这些扭曲的代码只有一个或者一小撮开发者可以理解，当他们离职后，这个项目将会变成噩梦。

有许多的因素会影响代码的复杂度。**一个很重要的因素就是[耦合](https://en.wikipedia.org/wiki/Coupling_(computer_programming))的总数，或者说是应用的模块之间的依赖性**。让我们来看下下面这段代码。假设这儿有一台服务器允许用户连接。当用户连接并且认证后，他们会被封装到`User`类中：

```java
// 用来表示一个用户
class User {
  public final String username;
  public final int id;
  public final Socket socket;

  public void disconnect() {
    socket.close();
  }

  // 其他的才做socket的方法。
}
```

如果你需要发送消息给用户，我们将会这样做：

```java
// Another class
class Message {
  // Send "Hello." string to a User
  public void sayHello(User user) {
    Socket s = user.socket; // Get the user's socket
    OutputStream outputStream = s.getOutputStream();
    PrintWriter out = new PrintWriter(outputStream);
    out.println("Hello."); // send the message to the socket.
  }
}
```

`User`类有个明显的缺陷：它没有封装`socket`对象，将它暴露给了所有人。如果我们想改变这个实现（比如使用异步的socket库），我们必须改变代码的许多地方。

然而，`Message`类的`sayHello(...)` 方法也同样有问题。**它与`socket`对象直接交互**。它**获取了不相干的“第三方”对象（socket）并且直接使用它**。这个例子可能有点随意，但是我在真实的应用代码中见到了太多的这样的代码。好消息是，这个能够**非常容易被一项技术检测出来，即 [Demeter法则](http://www.ccs.neu.edu/research/demeter/papers/law-of-demeter/oopsla88-law-of-demeter.pdf)**。这个法则（其实叫一项技术或者一个guideline更加合适）总结如下：

>  * 每一个单元仅仅**对其他的单元有非常有限的了解**：即联系非常紧密的单元
>  * 每一个单元只与他的朋友交互；**不要与陌生人说话**
>  * **只与亲密的朋友交互**。
>  基本原则为：**一个给定的对象，要尽可能少的了解它的结构、属性等等（包括他的子组件），与“infomation hiding”原则一致。

简单的说，罗辑不相干的模块的紧耦合违背了Demeter 法则。尽管那是缺少封装导致的副作用，这个法则并不鼓励访问和使用第三方的对象，下面是一个例子来解释Demeter法则：

```java
public class Foo {
    /**
     * 这个例子将会导致两个违反原则的地方
     */
    public void example(Bar b) {
        // 这个方法调用是ok的, 因为 "example" 的一个参数
        C c = b.getC();

        // 这个方法调用有问题, 因为我们使用的 c, 是从 B获取的。
        // 我们应该从b直接访问, e.g. "b.doItOnC();"
        c.doIt();

        // 这同样有问题, 这只是函数链来表示相同的罗辑，只是少了临时变量.
        b.getC().doIt();

        //构造函数，而不是方法调用
        D d = new D();
        // 这样也是ok的，因为我们创建了一个新的d的实例
        d.doSomethingElse();
    }
}
```

现在回到 `sayHello(...)`方法，它违背了Demeter法则里面的与陌生人（socket）交流，尽管**这个方法他自己病没啥卵用，但是它帮助我们检测紧耦合**。问题最开始是因为`User`类没有隐藏好它的内部细节，现在我们修复它:

```java
class User {
  public final String username;
  public final int id;
  // 把类变成私有的，以防止暴露给其他人.
  private final Socket socket;

  // 封装消息功能的简单的实现
  void sendMessage(String message) {
    OutputStream outputStream = socket.getOutputStream();
    PrintWriter out = new PrintWriter(outputStream);
    out.println(message);
  }
}
```

现在我们能够修改`sayHello(...)`方法，让其不再依赖`socket`对象。

```java
class Message {
  // Doesn't violate the Law of Demeter anymore.
  public void sayHello(User user) {
    user.sendMessage("Hello.");
  }
```

耦合问题就解决了。在书 [The Pragmatic Programmer](https://www.amazon.com/Pragmatic-Programmer-Journeyman-Master/dp/020161622X), [Andrew](https://twitter.com/pragmaticandy) 建议编写 **不会与太多对象交互的，“害羞”的代码**

我平时在写代码和review代码的时候，都会关注Demeter法则。但是这个是可以自动化的，使用[源码分析工具](http://pmd.github.io/)。虽然我没用用过他。


原文链接 [http://codeahoy.com/2016/06/17/the-law-of-demeter-writing-shy-code/](http://codeahoy.com/2016/06/17/the-law-of-demeter-writing-shy-code/)
