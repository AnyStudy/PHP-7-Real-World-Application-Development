# 依赖反转原则

依赖反转原则指出，实体应该依赖于抽象而不是具体实现。也就是说，一个高层次的模块不应该依赖于一个低层次的模块，而应该依赖于抽象。按照在维基百科上找到的定义：

> _依赖于抽象而不是一个实例_

这个原则很重要，因为它对我们的软件解耦起着重要作用。

下面是一个违反DIP的类的例子

```php
class Mailer {
    // Implementation...
}

class NotifySubscriber {
    public function notify($emailTo) {
        $mailer = new Mailer();
        $mailer->send('Thank you for...', $emailTo);
    }
}
```

在这里，我们可以看到`NotifySubscriber`类中的一个`notify`方法以依赖关系的方式编码给`Mailer`类。这就造成了紧耦合的代码，而这正是我们想要避免的。为了纠正这个问题，我们可以通过类构造函数，或者通过其他方法来传递依赖关系。此外，我们应该从具体的类依赖转向抽象的类依赖，如这里所示的整改后的例子所示：

```php
interface MailerInterface {
    // Implementation...
}

class Mailer implements MailerInterface {
    // Implementation...
}

class NotifySubscriber {
    private $mailer;

    public function __construct(MailerInterface $mailer) {
        $this->mailer = $mailer;
    }

    public function notify($emailTo) {
        $this->mailer->send('Thank you for...', $emailTo);
    }
}
```

这里我们看到一个依赖关系通过构造函数被注入。注入是由一个类型提示接口和实际的具体类抽象出来的。这使得我们的代码松散耦合。DIP可以在任何时候使用，一个类需要调用另一个类的方法，或者我们应该说是向它发送消息。

