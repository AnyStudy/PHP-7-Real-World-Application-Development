# 实施发布/订阅设计模式

发布/订阅\(Pub/Sub\)设计模式通常构成软件事件驱动编程的基础。这种方法允许在不同的软件应用程序之间，或在单个应用程序中的不同软件模块之间进行异步通信。该模式的目的是允许一个方法或函数在一个重要的动作发生时发布一个信号。然后，如果某个信号已经发布，一个或多个类就会订阅并采取行动。

这种操作的例子是当数据库被修改时，或者当用户已经登录时。这种设计模式的另一个常见用途是当一个应用程序提供新闻源时。如果一个紧急的新闻项目已经发布，应用程序将发布这个事实，允许客户端订阅者刷新他们的新闻列表。

## 如何做...

1.首先，我们定义我们的发布者类，`Application\PubSub\Publisher`。你会注意到我们使用了两个有用的**标准PHP库（SPL）**接口，`SplSubject`和`SplObserver`。

```php
namespace Application\PubSub;
use SplSubject;
use SplObserver;
class Publisher implements SplSubject
{
  // code
}
```

2. 接下来，我们添加属性来表示发布者名称、要传递给订阅者的数据以及订阅者数组（也称为听众）。你还会注意到，我们将使用一个链表（第10章，高级算法）来允许优先级。

```php
protected $name;
protected $data;
protected $linked;
protected $subscribers;
```

3.构造函数初始化了这些属性。我们还抛出了`__toString()`，以备我们需要快速访问这个发布者的名称。

```php
public function __construct($name)
{
  $this->name = $name;
  $this->data = array();
  $this->subscribers = array();
  $this->linked = array();
}

public function __toString()
{
  return $this->name;
}
```

4.为了将一个订阅者与这个发布者关联起来，我们定义了 `attach()`，它在 `SplSubject` 接口中被指定。我们接受一个`SplObserver`实例作为参数。请注意，我们需要向 `$subscribers` 和 `$linked` 属性添加条目。然后，使用`arsort()`对`$linked`进行按值排序，用优先级来表示，它的排序方式是反向的，并保持键。

```php
public function attach(SplObserver $subscriber)
{
  $this->subscribers[$subscriber->getKey()] = $subscriber;
  $this->linked[$subscriber->getKey()] = 
    $subscriber->getPriority();
  arsort($this->linked);
}
```

5.该接口还要求我们定义`detach()`，将订阅者从列表中删除。

```php
public function detach(SplObserver $subscriber)
{
  unset($this->subscribers[$subscriber->getKey()]);
  unset($this->linked[$subscriber->getKey()]);
}
```

6.同样是接口所要求的，我们定义了`notify()`，它调用所有订阅者的`update()`。注意，我们在链表中循环，以确保订阅者按优先级顺序被调用。

```php
public function notify()
{
  foreach ($this->linked as $key => $value)
  {
    $this->subscribers[$key]->update($this);
  }
}
```

7. 接下来，我们定义相应的getter和setter。为了节省篇幅，我们不在这里一一展示。

```php
public function getName()
{
  return $this->name;
}

public function setName($name)
{
  $this->name = $name;
}

```

8. 最后，我们需要提供一种通过键设置数据项的方法，然后在调用`notify()`时，订阅者可以使用该方法。

```php
public function setDataByKey($key, $value)
{
  $this->data[$key] = $value;
}
```

9. 现在我们可以看看`Application\PubSub\Subscriber`。通常情况下，我们会为每个发布者定义多个订阅者。在这种情况下，我们实现了`SplObserver`接口。

```php
namespace Application\PubSub;
use SplSubject;
use SplObserver;
class Subscriber implements SplObserver
{
  // code
}
```

10. 每个订阅者都需要一个唯一的标识符。在这种情况下，我们使用`md5()`和日期/时间信息，结合一个随机数来创建密钥。构造函数初始化属性如下。订阅者执行的实际逻辑功能是以回调的形式进行的。

```php
protected $key;
protected $name;
protected $priority;
protected $callback;
public function __construct(
  string $name, callable $callback, $priority = 0)
{
  $this->key = md5(date('YmdHis') . rand(0,9999));
  $this->name = $name;
  $this->callback = $callback;
  $this->priority = $priority;
}
```

11. 当调用发布者上的`notifiy()`时，就会调用`update()`函数。我们传递一个发布者实例作为参数，并调用为这个订阅者定义的回调。

```php
public function update(SplSubject $publisher)
{
  call_user_func($this->callback, $publisher);
}
```

12. 为了方便，我们还需要定义getter和setter。这里没有全部显示出来。

```php
public function getKey()
{
  return $this->key;
}

public function setKey($key)
{
  $this->key = $key;
}

// other getters and setters not shown
```

## 如何运行...

在这个例子中，定义一个名为`chap_11_pub_sub_simple_example.php`的调用程序，该程序设置自动加载并使用相应的类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\PubSub\ { Publisher, Subscriber };
```

接下来，创建一个发布者实例并分配数据。

```php
$pub = new Publisher('test');
$pub->setDataByKey('1', 'AAA');
$pub->setDataByKey('2', 'BBB');
$pub->setDataByKey('3', 'CCC');
$pub->setDataByKey('4', 'DDD');
```

现在你可以创建测试订阅者，从发布者那里读取数据并呼应结果。第一个参数是名称，第二个是回调，最后一个是优先级。

```php
$sub1 = new Subscriber(
  '1',
  function ($pub) {
    echo '1:' . $pub->getData()[1] . PHP_EOL;
  },
  10
);
$sub2 = new Subscriber(
  '2',
  function ($pub) {
    echo '2:' . $pub->getData()[2] . PHP_EOL;
  },
  20
);
$sub3 = new Subscriber(
  '3',
  function ($pub) {
    echo '3:' . $pub->getData()[3] . PHP_EOL;
  },
  99
);
```

为了测试的目的，不按顺序附加订阅者，并调用`notify()`两次。

```php
$pub->attach($sub2);
$pub->attach($sub1);
$pub->attach($sub3);
$pub->notify();
$pub->notify();
```

接下来，定义并附加另一个订阅者，该订阅者查看订阅者1的数据，如果不是空的就退出。

```php
$sub4 = new Subscriber(
  '4',
  function ($pub) {
    echo '4:' . $pub->getData()[4] . PHP_EOL;
    if (!empty($pub->getData()[1]))
      die('1 is set ... halting execution');
  },
  25
);
$pub->attach($sub4);
$pub->notify();
```

这里是输出。请注意，输出的顺序是按优先级排列的（优先级高的先输出），第二块输出被打断。

![](../../.gitbook/assets/image%20%28144%29.png)

## 更多...

与之密切相关的软件设计模式是Observer。其机制类似，但普遍认为的区别是，**Observer**以同步方式运行，当收到信号（通常也称为消息或事件）时，所有观察者方法都会被调用。而Pub/Sub模式则是异步操作，通常使用消息队列。另一个区别是，在Pub/Sub模式中，发布者不需要知道订阅者。

## 参见...

关于Observer和Pub/Sub模式之间的区别，请参考文章[http://stackoverflow.com/questions/15594905/difference-between-observer-pub-sub-and-data-binding](http://stackoverflow.com/questions/15594905/difference-between-observer-pub-sub-and-data-binding)。

