# 里氏替换原则

里氏替换原则说的是继承。它规定了我们应该如何设计我们的类，使客户的依赖性可以被子类所取代，而客户却看不出其中的差别，这是在维基百科上找到的定义。

> _程序中的对象应该可以用它们的子类的实例来替换，而不改变程序的正确性_

虽然子类可能会添加一些特定的功能，但它必须符合与基类相同的行为。否则就违反了里氏原则。

谈到PHP和子类，我们要超越简单的具体类，要区分：具体类、抽象类和接口。三者中的每一个都可以放在基类中，而一切扩展或实现基类的东西都可以看成是派生类。

下面是一个违反LSP的例子，派生类没有对所有方法进行实现。

```php
interface User {
    public function getEmail();
    public function getName();
    public function getAge();
}

class Employee implements User {
    public function getEmail() {
        // Implementation...
    }

    public function getAge() {
        // Implementation...
    }
}
```

在这里，我们看到一个 `Employee` 类没有实现接口`getName`方法。我们可以很容易地使用一个抽象类来代替接口和抽象方法类来实现`getName`方法，效果是一样的。幸运的是，在这种情况下，PHP会抛出一个错误，警告我们还没有真正完全实现接口。

下面是一个违反里氏原则的例子，不同的派生类返回不同的类型：

```php
class UsersCollection implements \Iterator {
    // Implementation...
}

interface UserList {
    public function getUsers();
}

class Emloyees implements UserList {
    public function getUsers() {
        $users = new UsersCollection();
        //...
        return $users;
    }
}

class Directors implements UserList {
    public function getUsers() {
        $users = array();
        //...
        return $users;
    }
}
```

这里我们看到一个简单的边缘案例。在两个派生类上调用`getUsers`会返回一个我们可以循环的结果。然而，PHP开发人员更倾向于在数组结构上使用`count`方法。在当前的`Employees`实例上使用它，`getUsers`的结果将无法工作。这是因为 `Employees` 类返回的是实现 `Iterator` 的 `UsersCollection`，而不是实际的数组结构。由于`UsersCollection`没有实现`Countable`，所以我们不能对它使用`count`，这就会导致潜在的bug。

我们还可以进一步发现派生类在方法参数方面表现不那么宽容的情况下的LSP违规行为。通常可以通过使用类型操作符的实例来发现这些情况，如下例所示：

```php
interface LoggerProcessor {
    public function log(LoggerInterface $logger);
}

class XmlLogger implements LoggerInterface {
    // Implementation...
}

class JsonLogger implements LoggerInterface {
    // Implementation...
}

class FileLogger implements LoggerInterface {
    // Implementation...
}

class Processor implements LoggerProcessor {
    public function log(LoggerInterface $logger) {
        if ($logger instanceof XmlLogger) {
            throw new \Exception('This processor does not work with XmlLogger');
        } else {
            // Implementation...
        }
    }
}
```

这里，派生类`Processor`对方法参数进行了限制，而它应该接受一切符合`LoggerInterface`的参数。通过减少允许性，它改变了基类（在本例中是`LoggerInterface`）所隐含的行为。

概述的例子只是构成违反LSP的一个片段。为了满足该原则，我们需要确保派生类不会以任何方式改变基类所施加的行为。

