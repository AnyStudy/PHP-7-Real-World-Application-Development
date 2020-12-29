# 结构型

结构型处理的是类和对象的组成。利用接口或抽象类和方法，它们定义了组成对象的方法，进而获得新的功能。以下是我们归类为结构型的模式列表。

* 适配器模式 
* 桥接模式
* 组合模式 
* 装饰器模式
* 门面模式
* 享元模式
* 代理模式

{% hint style="info" %}
有关结构型的更多信息，请参见https://en.wikipedia.org/wiki/Structural\_pattern。
{% endhint %}

## 适配器模式

适配器模式允许从另一个接口使用现有类的接口，基本上是通过将一个类的接口转换为另一个类所期望的接口，帮助两个不兼容的接口一起工作。

下面是一个适配器模式的实现例子。

```php
class Stripe {
    public function capturePayment($amount) {
        /* Implementation... */
    }

    public function authorizeOnlyPayment($amount) {
        /* Implementation... */
    }

    public function cancelAmount($amount) {
        /* Implementation... */
    }
}

interface PaymentService {
    public function capture($amount);
    public function authorize($amount);
    public function cancel($amount);
}

class StripePaymentServiceAdapter implements PaymentService {
    private $stripe;

    public function __construct(Stripe $stripe) {
        $this->stripe = $stripe;
    }

    public function capture($amount) {
        $this->stripe->capturePayment($amount);
    }

    public function authorize($amount) {
        $this->stripe->authorizeOnlyPayment($amount);
    }

    public function cancel($amount) {
        $this->stripe->cancelAmount($amount);
    }
}

// Client
$stripe = new StripePaymentServiceAdapter(new Stripe());
$stripe->authorize(49.99);
$stripe->capture(19.99);
$stripe->cancel(9.99);
```

我们首先创建一个具体的`Stripe`类。然后我们定义了`PaymentService`接口与一些基本的支付处理方法。`StripePaymentServiceAdapter`实现了`PaymentService`接口，提供了支付处理方法的具体实现。最后，客户端实例化`StripePaymentServiceAdapter`并执行支付处理方法。

## 桥接模式

当我们想把一个类或抽象从它的实现中解耦出来时，就会用到桥接模式，让它们都能独立地改变。当类和它的实现经常变化时，这很有用。

下面是一个桥模式实现的例子。

```php
interface MailerInterface {
    public function setSender(MessagingInterface $sender);
    public function send($body);
}

abstract class Mailer implements MailerInterface {
    protected $sender;

    public function setSender(MessagingInterface $sender) {
        $this->sender = $sender;
    }
}

class PHPMailer extends Mailer {
    public function send($body) {
        $body .= "\n\n Sent from a phpmailer.";
        return $this->sender->send($body);
    }
}

class SwiftMailer extends Mailer {
    public function send($body) {
        $body .= "\n\n Sent from a SwiftMailer.";
        return $this->sender->send($body);
    }
}

interface MessagingInterface {
    public function send($body);
}

class TextMessage implements MessagingInterface {
    public function send($body) {
        echo 'TextMessage > send > $body: ' . $body;
    }
}

class HtmlMessage implements MessagingInterface {
    public function send($body) {
        echo 'HtmlMessage > send > $body: ' . $body;
    }
}

// Client
$phpmailer = new PHPMailer();
$phpmailer->setSender(new TextMessage());
$phpmailer->send('Hi!');

$swiftMailer = new SwiftMailer();
$swiftMailer->setSender(new HtmlMessage());
$swiftMailer->send('Hello!');
```

我们首先创建一个`MailerInterface`。然后，具体的`Mailer`类实现了`MailerInterface`，为`PHPMailer`和`SwiftMailer`提供了一个基类。然后我们定义`MessagingInterface`，它由`TextMessage`和`HtmlMessage`类实现。最后，客户端实例化`PHPMailer`和`SwiftMailer`，在调用发送方法之前传递`TextMessage`和`HtmlMessage`的实例。

## 组合模式

组合模式就是将对象的层次结构作为一个对象，通过一个公共接口来处理。其中，对象组成三个结构，客户端因为只消耗公共接口，所以对底层结构的变化一无所知。

下面是一个组合模式的实现例子。

```php
interface Graphic {
    public function draw();
}

class CompositeGraphic implements Graphic {
    private $graphics = array();

    public function add($graphic) {
        $objId = spl_object_hash($graphic);
        $this->graphics[$objId] = $graphic;
    }

    public function remove($graphic) {
        $objId = spl_object_hash($graphic);
        unset($this->graphics[$objId]);
    }

    public function draw() {
        foreach ($this->graphics as $graphic) {
            $graphic->draw();
        }
    }
}

class Circle implements Graphic {
    public function draw()
    {
        echo 'draw-circle';
    }
}

class Square implements Graphic {
    public function draw() {
        echo 'draw-square';
    }
}

class Triangle implements Graphic {
    public function draw() {
        echo 'draw-triangle';
    }
}

$circle = new Circle();
$square = new Square();
$triangle = new Triangle();

$compositeObj1 = new CompositeGraphic();
$compositeObj1->add($circle);
$compositeObj1->add($triangle);
$compositeObj1->draw();

$compositeObj2 = new CompositeGraphic();
$compositeObj2->add($circle);
$compositeObj2->add($square);
$compositeObj2->add($triangle);
$compositeObj2->remove($circle);
$compositeObj2->draw();
```

我们首先创建了一个`Graphic`接口。然后我们创建了`CompositeGraphic`、`Circle`、`Square`和`Triangle`，它们都实现了`Graphic`接口。除了仅仅实现了`Graphic`接口中的`draw`方法外，`CompositeGraphic`还增加了两个方法，用于跟踪添加到它的图形的内部集合。然后客户端将这些`Graphic`类全部实例化，将它们全部添加到`CompositeGraphic`中，然后由`CompositeGraphic`调用`draw`方法。

## 装饰器模式

装饰器模式允许将行为添加到单个对象实例中，而不影响同一类的其他实例的行为。我们可以定义多个装饰器，其中每个装饰器都会增加新的功能。

下面是一个装饰器模式实现的例子。

```php
interface LoggerInterface {
    public function log($message);
}

class Logger implements LoggerInterface {
    public function log($message) {
        file_put_contents('app.log', $message, FILE_APPEND);
    }
}

abstract class LoggerDecorator implements LoggerInterface {
    protected $logger;

    public function __construct(Logger $logger) {
        $this->logger = $logger;
    }

    abstract public function log($message);
}

class ErrorLoggerDecorator extends LoggerDecorator {
    public function log($message) {
        $this->logger->log('ERROR: ' . $message);
    }

}

class WarningLoggerDecorator extends LoggerDecorator {
    public function log($message) {
        $this->logger->log('WARNING: ' . $message);
    }
}

class NoticeLoggerDecorator extends LoggerDecorator {
    public function log($message) {
        $this->logger->log('NOTICE: ' . $message);
    }
}

$logger = new Logger();
$logger->log('Resource not found.');

$logger = new Logger();
$logger = new ErrorLoggerDecorator($logger);
$logger->log('Invalid user role.');

$logger = new Logger();
$logger = new WarningLoggerDecorator($logger);
$logger->log('Missing address parameters.');

$logger = new Logger();
$logger = new NoticeLoggerDecorator($logger);
$logger->log('Incorrect type provided.');
```

我们首先创建了一个`LoggerInterface`，其中有一个简单的`log`方法，然后我们定义了`Logger`和`LoggerDecorator`，两者都实现了`LoggerInterface`。然后我们定义了`Logger`和`LoggerDecorator`，它们都是实现`LoggerInterface`的。其次是`ErrorLoggerDecorator`、`WarningLoggerDecorator`和`NoticeLoggerDecorator`，它们实现了`LoggerDecorator`。最后，客户端部分三次实例化`Logger`，传递给它不同的装饰器。

## 门面模式

当我们想通过一个更简单的接口来简化大型系统的复杂性时，就会用到门面模式。它通过为大多数常见任务提供方便的方法，通过一个客户端使用的单一封装类来实现。

下面是一个门面模式实现的例子。

```php
class Product {
    public function getQty() {
        // Implementation
    }
}

class QuickOrderFacade {
    private $product = null;
    private $orderQty = null;

    public function __construct($product, $orderQty) {
        $this->product = $product;
        $this->orderQty = $orderQty;
    }

    public function generateOrder() {
        if ($this->qtyCheck()) {
            $this->addToCart();
            $this->calculateShipping();
            $this->applyDiscount();
            $this->placeOrder();
        }
    }

    private function addToCart() {
        // Implementation...
    }

    private function qtyCheck() {
        if ($this->product->getQty() > $this->orderQty) {
            return true;
        } else {
            return true;
        }
    }

    private function calculateShipping() {
        // Implementation...
    }

    private function applyDiscount() {
        // Implementation...
    }

    private function placeOrder() {
        // Implementation...
    }
}

// Client
$order = new QuickOrderFacade(new Product(), $qty);
$order->generateOrder();
```

我们首先创建一个`Product`类，并提供一个单一的`getQty`方法。然后，我们创建了一个`QuickOrderFacade`类，通过构造函数接受产品实例和数量，并进一步提供`generateOrder`方法，聚合所有的订单生成动作。最后，客户端实例化产品，将其传递给`QuickOrderFacade`的实例，对其调用`generateOrder`。

## 享元模式

享元模式是关于性能和资源的减少，在相似的对象之间共享尽可能多的数据。这意味着在一个实现中，一个类的相同实例被共享。当预计要创建大量相同的类实例时，这种模式效果最好。

下面是一个享元模式实现的例子。

```php
interface Shape {
    public function draw();
}

class Circle implements Shape {
    private $colour;
    private $radius;

    public function __construct($colour) {
        $this->colour = $colour;
    }

    public function draw() {
        echo sprintf('Colour %s, radius %s.', $this->colour, $this->radius);
    }

    public function setRadius($radius) {
        $this->radius = $radius;
    }
}

class ShapeFactory {
    private $circleMap;

    public function getCircle($colour) {
        if (!isset($this->circleMap[$colour])) {
            $circle = new Circle($colour);
            $this->circleMap[$colour] = $circle;
        }

        return $this->circleMap[$colour];
    }
}

// Client
$shapeFactory = new ShapeFactory();
$circle = $shapeFactory->getCircle('yellow');
$circle->setRadius(10);
$circle->draw();

$shapeFactory = new ShapeFactory();
$circle = $shapeFactory->getCircle('orange');
$circle->setRadius(15);
$circle->draw();

$shapeFactory = new ShapeFactory();
$circle = $shapeFactory->getCircle('yellow');
$circle->setRadius(20);
$circle->draw();
```

我们首先创建了一个`Shape`接口，有一个单一的 `draw` 方法。然后我们定义了实现`Shape`接口的`Circle`类，接着是`ShapeFactory`类。在`ShapeFactory`中，`getCircle`方法根据颜色选项返回一个新Circle的实例。最后，客户端实例化多个`ShapeFactory`对象，向`getCircle`方法调用传递不同的颜色。

## 代理模式

代理模式的功能是作为一个原始对象的幕后接口。它可以作为一个简单的转发包装器，甚至可以围绕它所包装的对象提供额外的功能。额外附加功能的例子可能是懒加载或缓存，这可能会补偿原始对象的资源密集操作。

下面是一个代理模式实现的例子。

```php
interface ImageInterface {
    public function draw();
}

class Image implements ImageInterface {
    private $file;

    public function __construct($file) {
        $this->file = $file;
        sleep(5); // Imagine resource intensive image load
    }

    public function draw() {
        echo 'image: ' . $this->file;
    }
}

class ProxyImage implements ImageInterface {
    private $image = null;
    private $file;

    public function __construct($file) {
        $this->file = $file;
    }

    public function draw() {
        if (is_null($this->image)) {
            $this->image = new Image($this->file);
        }

        $this->image->draw();
    }
}

// Client
$image = new Image('image.png'); // 5 seconds
$image->draw();

$image = new ProxyImage('image.png'); // 0 seconds
$image->draw();
```

我们首先创建了一个`ImageInterface`，有一个单一的`draw`方法。然后我们定义了`Image`和`ProxyImage`类，这两个类都是`ImageInterface`的扩展。在`Image`类的 `__construct` 中，我们用`sleep`方法调用模拟了资源紧张的操作。最后，客户端同时实例化`Image`和`ProxyImage`，显示出两者的执行时间差异。

