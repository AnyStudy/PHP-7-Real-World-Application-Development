# GoF设计模式

有很多事情可以使一个优秀的软件开发人员成为可能。设计模式的知识和使用就是其中之一。设计模式使开发人员能够使用各种软件交互的知名名称进行交流。无论某人是PHP、Python、C\#、Ruby或任何其他语言的开发者，设计模式都能为经常出现的软件问题提供语言不可知的解决方案。

设计模式的概念出现在1994年，作为《可重用的面向对象的软件要素》一书的一部分。该书详细介绍了23种不同的设计模式，由四位作者Erich Gamma、Richard Helm、Ralph Johnson和John Vlissides撰写。作者们通常被称为四人帮（GoF），所介绍的设计模式有时也被称为GoF设计模式。在二十多年后的今天，如果不将设计模式作为实现的一部分，设计出可扩展、可重用、可维护和可适应的软件几乎是不可能的。

有三种类型的设计模式，我们将在本章介绍。

* 创建型
* 结构型
* 行为型

## 创建型

创建型，顾名思义就是为我们创建对象，所以我们不必直接实例化它们。实现创建型使我们的应用程序具有一定的灵活性，应用程序本身可以决定在给定时间内实例化哪些对象。以下是我们归类为创建型的模式列表。

* 抽象工厂模式 
* 生成器模式 
* 工厂方法模式 
* 原型模式 
* 单例模式

{% hint style="info" %}
关于创建模式的更多信息，请参见https://en.wikipedia.org/wiki/Creational\_pattern。
{% endhint %}

### 抽象工厂模式

构建可移植的应用程序需要高水平的依赖性封装。抽象工厂通过抽象化相关或依赖对象家族的创建来实现这一点。客户端从来没有直接创建这些平台对象，工厂为他们做了这些工作，使得在不改变使用它们的代码的情况下，甚至在运行时也可以互换具体实现。

下面是一个抽象工厂模式实现的例子。

```php
interface Button {
    public function render();
}

interface GUIFactory {
    public function createButton();
}

class SubmitButton implements Button {
    public function render() {
        echo 'Render Submit Button';
    }
}

class ResetButton implements Button {
    public function render() {
        echo 'Render Reset Button';
    }
}

class SubmitFactory implements GUIFactory {
    public function createButton() {
        return new SubmitButton();
    }
}

class ResetFactory implements GUIFactory {
    public function createButton() {
        return new ResetButton();
    }
}

// Client
$submitFactory = new SubmitFactory();
$button = $submitFactory->createButton();
$button->render();

$resetFactory = new ResetFactory();
$button = $resetFactory->createButton();
$button->render();
```

我们首先创建一个接口`Button`，之后由我们的`SubmitButton`和`ResetButton`具体类实现。`GUIFactory`和`ResetFactory`实现了`GUIFactory`接口，它指定了`createButton`方法。然后客户端只需实例化工厂并调用`createButton`，就会返回一个合适的按钮实例，我们调用渲染方法。

### 生成器模式

生成器模式将复杂对象的构建与表示分离，使得同一个构建过程可以创建不同的表示。有些生成器模式是在一次调用中构建一个产品，而生成器模式则是在 director 的控制下逐步完成。

下面是一个生成器模式的实现实例。

```php
class Car {
    public function getWheels() {
        /* implementation... */
    }

    public function setWheels($wheels) {
        /* implementation... */
    }

    public function getColour($colour) {
        /* implementation... */
    }

    public function setColour() {
        /* implementation... */
    }
}

interface CarBuilderInterface {
    public function setColour($colour);
    public function setWheels($wheels);
    public function getResult();
}

class CarBuilder implements CarBuilderInterface {
    private $car;

    public function __construct() {
        $this->car = new Car();
    }

    public function setColour($colour) {
        $this->car->setColour($colour);
        return $this;
    }

    public function setWheels($wheels) {
        $this->car->setWheels($wheels);
        return $this;
    }

    public function getResult() {
        return $this->car;
    }
}

class CarBuildDirector {
    private $builder;

    public function __construct(CarBuilder $builder) {
        $this->builder = $builder;
    }

    public function build() {
        $this->builder->setColour('Red');
        $this->builder->setWheels(4);

        return $this;
    }

    public function getCar() {
        return $this->builder->getResult();
    }
}

// Client
$carBuilder = new CarBuilder();
$carBuildDirector = new CarBuildDirector($carBuilder);
$car = $carBuildDirector->build()->getCar();
```

我们首先创建了一个具体的`Car`类，其中有几个方法定义了汽车的一些基本特性。然后我们创建了一个`CarBuilderInterface`，它将控制其中的一些特性，并得到最终的结果（汽车）。然后，具体的`CarBuilder`类实现了`CarBuilderInterface`，接着是具体的`CarBuildDirector`类，它定义了`build`和`getCar`方法。然后，客户端简单地实例化一个新的`CarBuilder`实例，将其作为构造参数传递给一个新的`CarBuildDirector`实例。最后，我们调用`CarBuildDirector`的`build和getCar`方法来获取实际的汽车`Car`实例。

### 工厂方法模式

工厂方法模式处理了创建对象的问题，而不必指定将要创建的对象的确切类。

下面是一个工厂方法模式的实现例子。

```php
interface Product {
    public function getType();
}

interface ProductFactory {
    public function makeProduct();
}

class SimpleProduct implements Product {
    public function getType() {
        return 'SimpleProduct';
    }
}

class SimpleProductFactory implements ProductFactory {
    public function makeProduct() {
        return new SimpleProduct();
    }
}

/* Client */
$factory = new SimpleProductFactory();
$product = $factory->makeProduct();
echo $product->getType(); //outputs: SimpleProduct
```

我们首先创建一个`ProductFactory`和`Product`接口。`SimpleProductFactory`实现了`ProductFactory`，并通过其`makeProduct`方法返回新的产品实例。`SimpleProduct`类实现`Product`，并返回产品类型。最后，客户端创建`SimpleProductFactory`的实例，对其调用`makeProduct`方法。`makeProduct`返回`Product`的实例，其`getType`方法返回`SimpleProduct`字符串。

### 原型模式

原型模式通过使用克隆来复制其他对象。这意味着我们不是使用`new`关键字来实例化新的对象。PHP提供了一个`clone`关键字，它可以对一个对象进行浅层复制，从而提供了非常直接的原型模式实现。浅层复制不复制引用，只复制新对象的值。我们可以进一步在我们的类上利用神奇的`__clone`方法，以实现更强大的克隆行为。

下面是一个原型模式实现的例子。

```php
class User {
    public $name;
    public $email;
}

class Employee extends User {
    public function __construct() {
        $this->name = 'Johhn Doe';
        $this->email = 'john.doe@fake.mail';
    }

    public function info() {
        return sprintf('%s, %s', $this->name, $this->email);
    }

    public function __clone() {
        /* additional changes for (after)clone behavior? */
    }
}

$employee = new Employee();
echo $employee->info();

$director = clone $employee;
$director->name = 'Jane Doe';
$director->email = 'jane.doe@fake.mail';
echo $director->info(); //outputs: Jane Doe, jane.doe@fake.mail
```

我们先创建一个简单的`User`类。然后，`Employee`扩展了`User`，同时在其构造函数中设置了名称和电子邮件。然后客户端通过`new`关键字实例化`Employee`，并将其克隆到`director` 变量中。现在`$director`变量是一个新的实例，这个实例不是通过new关键字制作的，而是通过克隆，使用`clone`关键字制作的。改变`$director`上的名字和邮箱，不会影响`$employee`。

### 单例模式

单例模式的目的是将类的实例化限制在一个对象上。它是通过在类中创建一个方法来实现的，如果一个类的实例不存在，该方法就会创建一个新的实例。如果一个对象实例已经存在，该方法只是返回一个现有对象的引用。

下面是一个单例模式实现的例子。

```php
class Logger {
    private static $instance;

    public static function getInstance() {
        if (!isset(self::$instance)) {
            self::$instance = new self;
        }

        return self::$instance;
    }

    public function logNotice($msg) {
        return 'logNotice: ' . $msg;
    }

    public function logWarning($msg) {
        return 'logWarning: ' . $msg;
    }

    public function logError($msg) {
        return 'logError: ' . $msg;
    }
}

// Client
echo Logger::getInstance()->logNotice('test-notice');
echo Logger::getInstance()->logWarning('test-warning');
echo Logger::getInstance()->logError('test-error');
// Outputs:
// logNotice: test-notice
// logWarning: test-warning
// logError: test-error
```

我们首先创建了一个具有静态`$instance`成员的`Logger`类，以及总是返回该类的单个实例的`getInstance`方法。然后我们添加了一些示例方法来演示客户端在单个实例上执行各种方法。

