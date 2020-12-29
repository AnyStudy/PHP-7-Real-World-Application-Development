# 行为型

行为型解决的是不同对象之间的通信挑战。它们描述了不同的对象和类之间如何相互发送消息以使事情发生。以下是我们归类为行为型的模式列表。

* 责任链模式 
* 命令模式 
* 解析器模式
* 迭代器模式 
* 中介者模式 
* 备忘录模式 
* 观察者模式 
* 状态模式 
* 策略模式
* 模板方法模式 
* 访问者模式

## 责任链模式

责任链模式通过使一个以上的对象以链式方式处理请求，将请求的发送方与接收方解耦。各种类型的处理对象可以动态地添加到链中。使用递归组成链，可以实现无限数量的处理对象。

下面是一个责任链模式的实现例子。

```php
abstract class SocialNotifier {
    private $notifyNext = null;

    public function notifyNext(SocialNotifier $notifier) {
        $this->notifyNext = $notifier;
        return $this->notifyNext;
    }

    final public function push($message) {
        $this->publish($message);

        if ($this->notifyNext !== null) {
            $this->notifyNext->push($message);
        }
    }

    abstract protected function publish($message);
}

class TwitterSocialNotifier extends SocialNotifier {
    public function publish($message) {
        // Implementation...
    }
}

class FacebookSocialNotifier extends SocialNotifier {
    protected function publish($message) {
        // Implementation...
    }
}

class PinterestSocialNotifier extends SocialNotifier {
    protected function publish($message) {
        // Implementation...
    }
}

// Client
$notifier = new TwitterSocialNotifier();

$notifier->notifyNext(new FacebookSocialNotifier())
    ->notifyNext(new PinterestSocialNotifier());

$notifier->push('Awesome new product available!');
```

我们首先创建了一个抽象的`SocialNotifier`类，并实现了抽象方法`publish`、`notifyNext`和`push`方法。然后我们定义了`TwitterSocialNotifier`、`FacebookSocialNotifier`和`PinterestSocialNotifier`，它们都是对抽象`SocialNotifier`的扩展。客户端首先实例化`TwitterSocialNotifier`，然后是两次`notifyNext`调用，在调用最后的`push`方法之前，向它传递另外两个`notifier`类型的实例。

## 命令模式

命令模式将执行某些操作的对象与知道如何使用它的对象解耦。它通过封装以后执行某个操作所需的所有相关信息来实现。这意味着对象、方法名和方法参数的信息。

下面是一个命令模式的实现例子。

```php
interface LightBulbCommand {
    public function execute();
}

class LightBulbControl {
    public function turnOn() {
        echo 'LightBulb turnOn';
    }

    public function turnOff() {
        echo 'LightBulb turnOff';
    }
}

class TurnOnLightBulb implements LightBulbCommand {
    private $lightBulbControl;

    public function __construct(LightBulbControl $lightBulbControl) {
        $this->lightBulbControl = $lightBulbControl;
    }

    public function execute() {
        $this->lightBulbControl->turnOn();
    }
}

class TurnOffLightBulb implements LightBulbCommand {
    private $lightBulbControl;

    public function __construct(LightBulbControl $lightBulbControl) {
        $this->lightBulbControl = $lightBulbControl;
    }

    public function execute() {
        $this->lightBulbControl->turnOff();
    }
}

// Client
$command = new TurnOffLightBulb(new LightBulbControl());
$command->execute();
```

我们首先创建一个`LightBulbCommand`接口。然后我们定义了`LightBulbControl`类，它提供了两个简单的`turnOn`/`turnOff`方法。然后，我们定义了`TurnOnLightBulb`和`TurnOffLightBulb`类，它们实现了`LightBulbCommand`接口。最后，客户端是用`LightBulbControl`的实例实例化`TurnOffLightBulb`对象，并对其调用执行方法。

## 解析器模式

解释器模式规定了如何评价语言语法或表达式。我们在定义解释器的同时，也定义了语言语法的表示方法。语言语法的表示使用复合类层次结构，其中规则被映射到类。然后，解释器使用表示法来解释语言中的表达式。

下面是一个解释器模式实现的例子。

```php
interface MathExpression
{
    public function interpret(array $values);
}

class Variable implements MathExpression {
    private $char;

    public function __construct($char) {
        $this->char = $char;
    }

    public function interpret(array $values) {
        return $values[$this->char];
    }
}

class Literal implements MathExpression {
    private $value;

    public function __construct($value) {
        $this->value = $value;
    }

    public function interpret(array $values) {
        return $this->value;
    }
}

class Sum implements MathExpression {
    private $x;
    private $y;

    public function __construct(MathExpression $x, MathExpression $y) {
        $this->x = $x;
        $this->y = $y;
    }

    public function interpret(array $values) {
        return $this->x->interpret($values) + $this->y->interpret($values);
    }
}

class Product implements MathExpression {
    private $x;
    private $y;

    public function __construct(MathExpression $x, MathExpression $y) {
        $this->x = $x;
        $this->y = $y;
    }

    public function interpret(array $values) {
        return $this->x->interpret($values) * $this->y->interpret($values);
    }
}

// Client
$expression = new Product(
    new Literal(5),
    new Sum(
        new Variable('c'),
        new Literal(2)
    )
);

echo $expression->interpret(array('c' => 3)); // 25
```

我们首先创建一个`MathExpression`接口，有一个 `interpret` 方法。然后我们添加`Variable`、`Literal`、`Sum`和`Product`类，所有这些类都实现了`MathExpression`接口。然后，客户端从`Product`类中实例化，将`Literal`和`Sum`的实例传递给它，并以一个 `interpret` 方法调用结束。

## 迭代器模式

迭代器模式用于遍历一个容器并访问其元素。换句话说，一个类可以遍历另一个类的元素。PHP对迭代器有一个本地支持，作为内置的`\Iterator`和`\IteratorAggregate`接口的一部分。

下面是一个迭代器模式实现的例子。

```php
class ProductIterator implements \Iterator {
    private $position = 0;
    private $productsCollection;

    public function __construct(ProductCollection $productsCollection) {
        $this->productsCollection = $productsCollection;
    }

    public function current() {
        return $this->productsCollection->getProduct($this->position);
    }

    public function key() {
        return $this->position;
    }

    public function next() {
        $this->position++;
    }

    public function rewind() {
        $this->position = 0;
    }

    public function valid() {
        return !is_null($this->productsCollection->getProduct($this->position));
    }
}

class ProductCollection implements \IteratorAggregate {
    private $products = array();

    public function getIterator() {
        return new ProductIterator($this);
    }

    public function addProduct($string) {
        $this->products[] = $string;
    }

    public function getProduct($key) {
        if (isset($this->products[$key])) {
            return $this->products[$key];
        }
        return null;
    }

    public function isEmpty() {
        return empty($products);
    }
}

$products = new ProductCollection();
$products->addProduct('T-Shirt Red');
$products->addProduct('T-Shirt Blue');
$products->addProduct('T-Shirt Green');
$products->addProduct('T-Shirt Yellow');

foreach ($products as $product) {
    var_dump($product);
}
```

首先创建了一个`ProductIterator`，它实现了标准的PHP `\Iterator`接口，然后我们添加了`ProductCollection`，它实现了标准的PHP `\IteratorAggregate`接口。客户端创建了一个`ProductCollection`的实例，通过`addProduct`方法调用将值堆积到其中，并循环浏览整个集合。

## 中介者模式

我们的软件中的类越多，它们的通信就越复杂。中介者模式通过将其封装成一个中介者对象来解决这种复杂性。对象不再直接通信，而是通过中介对象进行通信，因此降低了整体的耦合度。

下面是一个中介者模式实现的例子。

```php
interface MediatorInterface {
    public function fight();
    public function talk();
    public function registerA(ColleagueA $a);
    public function registerB(ColleagueB $b);
}

class ConcreteMediator implements MediatorInterface {
    protected $talk; // ColleagueA
    protected $fight; // ColleagueB

    public function registerA(ColleagueA $a) {
        $this->talk = $a;
    }

    public function registerB(ColleagueB $b) {
        $this->fight = $b;
    }

    public function fight() {
        echo 'fighting...';
    }

    public function talk() {
        echo 'talking...';
    }
}

abstract class Colleague {
    protected $mediator; // MediatorInterface
    public abstract function doSomething();
}

class ColleagueA extends Colleague {

    public function __construct(MediatorInterface $mediator) {
        $this->mediator = $mediator;
        $this->mediator->registerA($this);
    }

public function doSomething() {
        $this->mediator->talk();
}
}

class ColleagueB extends Colleague {

    public function __construct(MediatorInterface $mediator) {
        $this->mediator = $mediator;
        $this->mediator->registerB($this);
    }

    public function doSomething() {
        $this->mediator->fight();
    }
}

// Client
$mediator = new ConcreteMediator();
$talkColleague = new ColleagueA($mediator);
$fightColleague = new ColleagueB($mediator);

$talkColleague->doSomething();
$fightColleague->doSomething();
```

我们首先创建了一个带有多个方法的`MediatorInterface`，由`ConcreteMediator`类实现。然后，我们定义了抽象类`Colleague`，强制在下面的`ColleagueA`和`ColleagueB`类上实现`doSomething`方法。客户端首先实例化`ConcreteMediator`，并将其实例传递给`ColleagueA`和`ColleagueB`的实例，并根据这些实例调用`doSomething`方法。

## 备忘录模式

备忘录模式提供了对象还原功能。通过三个不同的对象来实现：originator、caretaker和memento，其中originator是保存以后还原所需的内部状态的对象。

以下是备忘录模式实现的一个例子。

```php
class Memento {
    private $state;

    public function __construct($state) {
        $this->state = $state;
    }

    public function getState() {
        return $this->state;
    }
}

class Originator {
    private $state;

    public function setState($state) {
        return $this->state = $state;
    }

    public function getState() {
        return $this->state;
    }

    public function saveToMemento() {
        return new Memento($this->state);
    }

    public function restoreFromMemento(Memento $memento) {
        $this->state = $memento->getState();
    }
}

// Client - Caretaker
$savedStates = array();

$originator = new Originator();
$originator->setState('new');
$originator->setState('pending');
$savedStates[] = $originator->saveToMemento();
$originator->setState('processing');
$savedStates[] = $originator->saveToMemento();
$originator->setState('complete');
$originator->restoreFromMemento($savedStates[1]);
echo $originator->getState(); // processing
```

我们首先创建一个`Memento`类，它将通过`getState`方法提供对象的当前状态。然后我们定义了`Originator`类，将状态推送给`Memento`。最后，客户端通过实例化`Originator`来扮演 `caretaker` 的角色，在它的几个状态之间进行杂耍，从`Memento`中保存和恢复这些状态。

## 观察者模式

观察者模式实现了对象之间一对多的依赖关系。持有依赖列表的对象被称为subject，而依赖者被称为observer。当subject对象改变状态时，所有的依赖对象都会得到通知并自动更新。

下面是一个观察者模式的实现例子。

```php
class Customer implements \SplSubject {
    protected $data = array();
    protected $observers = array();

    public function attach(\SplObserver $observer) {
        $this->observers[] = $observer;
    }

    public function detach(\SplObserver $observer) {
        $index = array_search($observer, $this->observers);

        if ($index !== false) {
            unset($this->observers[$index]);
        }
    }

    public function notify() {
        foreach ($this->observers as $observer) {
            $observer->update($this);
            echo 'observer updated';
        }
    }

    public function __set($name, $value) {
        $this->data[$name] = $value;

        // notify the observers, that user has been updated
        $this->notify();
    }
}

class CustomerObserver implements \SplObserver {
    public function update(\SplSubject $subject) {
        /* Implementation... */
    }
}

// Client
$user = new Customer();
$customerObserver = new CustomerObserver();
$user->attach($customerObserver);

$user->name = 'John Doe';
$user->email = 'john.doe@fake.mail';
```

我们首先创建一个`Customer`类，它实现了标准的PHP `\SplSubject`接口。然后我们定义了`CustomerObserver`类，它实现了标准的PHP `\SplObserver`接口。最后，客户端实例化`Customer`和`CustomerObserver`对象，并将`CustomerObserver`对象附加到`Customer`上。任何对姓名和电子邮件属性的改变都会被观察者捕获。

## 状态模式

状态模式封装了同一对象根据其内部状态而产生的不同行为，使对象看起来像是改变了它的类。

下面是一个状态模式的实现例子。

```php
interface Statelike {
    public function writeName(StateContext $context, $name);
}

class StateLowerCase implements Statelike {
    public function writeName(StateContext $context, $name) {
        echo strtolower($name);
        $context->setState(new StateMultipleUpperCase());
    }
}

class StateMultipleUpperCase implements Statelike {
    private $count = 0;

    public function writeName(StateContext $context, $name) {
        $this->count++;
        echo strtoupper($name);
        /* Change state after two invocations */
        if ($this->count > 1) {
            $context->setState(new StateLowerCase());
        }
    }
}

class StateContext {
    private $state;

    public function setState(Statelike $state) {
        $this->state = $state;
    }

    public function writeName($name) {
        $this->state->writeName($this, $name);
    }
}

// Client
$stateContext = new StateContext();
$stateContext->setState(new StateLowerCase());
$stateContext->writeName('Monday');
$stateContext->writeName('Tuesday');
$stateContext->writeName('Wednesday');
$stateContext->writeName('Thursday');
$stateContext->writeName('Friday');
$stateContext->writeName('Saturday');
$stateContext->writeName('Sunday');
```

我们首先创建了一个`Statelike`接口，然后是实现该接口的`StateLowerCase`和`StateMultipleUpperCase`。`StateMultipleUpperCase`在它的`writeName`中加入了一点计数逻辑，所以它在两次调用之后就会启动新的状态。然后我们定义了`StateContext`类，我们将用它来切换上下文。最后，客户端实例化`StateContext`，并通过`setState`方法将`StateLowerCase`的一个实例传递给它，然后是几个`writeName`方法。

## 策略模式

策略模式定义了一个算法家族，每个算法都被封装起来，并可与该家族中的其他成员互换。

下面是一个策略模式的实现例子。

```php
interface PaymentStrategy {
    public function pay($amount);
}

class StripePayment implements PaymentStrategy {
    public function pay($amount) {
        echo 'StripePayment...';
    }

}

class PayPalPayment implements PaymentStrategy {
    public function pay($amount) {
        echo 'PayPalPayment...';
    }
}

class Checkout {
    private $amount = 0;

    public function __construct($amount = 0) {
        $this->amount = $amount;
    }

    public function capturePayment() {
        if ($this->amount > 99.99) {
            $payment = new PayPalPayment();
        } else {
            $payment = new StripePayment();
        }

        $payment->pay($this->amount);
    }
}

$checkout = new Checkout(49.99);
$checkout->capturePayment(); // StripePayment...

$checkout = new Checkout(199.99);
$checkout->capturePayment(); // PayPalPayment...
```

我们首先创建了一个`PaymentStrategy`接口，然后是实现它的具体类`StripePayment`和`PayPalPayment`。然后我们定义了`Checkout`类，并在`capturePayment`方法中加入了一些决策逻辑。最后，客户端实例化`Checkout`，通过其构造函数传递一定的金额。根据金额，`Checkout`内部在调用`capturePayment`时，会触发一个或另一个付款。

## 模板方法模式

模板方法模式在方法中定义了算法的程序骨架。它让我们通过使用类重载，重新定义算法的某些步骤，而不真正改变算法的结构。

下面是一个模板方法模式实现的例子。

```php
abstract class Game {
    private $playersCount;

    abstract function initializeGame();
    abstract function makePlay($player);
    abstract function endOfGame();
    abstract function printWinner();

    public function playOneGame($playersCount)
    {
        $this->playersCount = $playersCount;
        $this->initializeGame();
        $j = 0;
        while (!$this->endOfGame()) {
            $this->makePlay($j);
            $j = ($j + 1) % $playersCount;
        }
        $this->printWinner();
    }
}

class Monopoly extends Game {
    public function initializeGame() {
        // Implementation...
    }

    public function makePlay($player) {
        // Implementation...
    }

    public function endOfGame() {
        // Implementation...
    }

    public function printWinner() {
        // Implementation...
    }
}

class Chess extends Game {
    public function  initializeGame() {
        // Implementation...
    }

    public function  makePlay($player) {
        // Implementation...
    }


    public function  endOfGame() {
        // Implementation...
    }

    public function  printWinner() {
        // Implementation...
    }
}

$game = new Chess();
$game->playOneGame(2);

$game = new Monopoly();
$game->playOneGame(4);
```

我们首先创建了一个抽象的`Game`类，该类提供了所有封装游戏玩法的实际抽象方法。然后我们定义了`Monopoly`和`Chess`类，这两个类都是由`Game`类扩展而来，为每个类实现了特定的游戏方法`game-play`。客户端只需实例化`Monopoly`和`Chess`对象，对每个对象调用`playOneGame`方法。

## 访问者模式

访问者模式是一种将算法与其操作的对象结构分离的方法。因此，我们能够在现有的对象结构上添加新的操作，而不实际修改这些结构。

下面是一个访问者模式的实现例子。

```php
interface RoleVisitorInterface {
    public function visitUser(User $role);
    public function visitGroup(Group $role);
}

class RolePrintVisitor implements RoleVisitorInterface {
    public function visitGroup(Group $role) {
        echo 'Role: ' . $role->getName();
    }

    public function visitUser(User $role) {
        echo 'Role: ' . $role->getName();
    }
}

abstract class Role {
    public function accept(RoleVisitorInterface $visitor) {
        $klass = get_called_class();
        preg_match('#([^\\\\]+)$#', $klass, $extract);
        $visitingMethod = 'visit' . $extract[1];

        if (!method_exists(__NAMESPACE__ . '\RoleVisitorInterface', $visitingMethod)) {
            throw new \InvalidArgumentException("The visitor you provide cannot visit a $klass instance");
        }

        call_user_func(array($visitor, $visitingMethod), $this);
    }
}

class User extends Role {
    protected $name;

    public function __construct($name) {
        $this->name = (string)$name;
    }

    public function getName() {
        return 'User ' . $this->name;
    }
}

class Group extends Role {
    protected $name;

    public function __construct($name) {
        $this->name = (string)$name;
    }

    public function getName() {
        return 'Group: ' . $this->name;
    }
}

$group = new Group('my group');
$user = new User('my user');

$visitor = new RolePrintVisitor;

$group->accept($visitor);
$user->accept($visitor);
```

我们首先创建一个`RoleVisitorInterface`，然后是`RolePrintVisitor`，它实现了`RoleVisitorInterface`本身。然后我们定义了一个抽象类`Role`，它有一个接受方法来接受`RoleVisitorInterface`的参数类型。我们进一步定义了具体的`User`和`Group`类，这两个类都是从`Role`扩展而来的。客户端实例化`User`、`Group`和`RolePrintVisitor`；将 `visitor` 传入`User`和`Group`实例的`accept`方法调用。

