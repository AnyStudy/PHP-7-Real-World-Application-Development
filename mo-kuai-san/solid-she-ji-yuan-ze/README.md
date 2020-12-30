# SOLID 设计原则

构建模块化软件需要很强的类设计知识。有很多指南，涉及到我们对类的命名方式，类应该有多少个变量，方法的大小应该是多少等等。PHP生态系统设法将这些打包成了官方的PSR标准，更准确地说，是**PSR-1：基本编码标准**和**PSR-2：编码风格指南**。这些都是一般的编程指南，让我们的代码可读、可理解、可维护。

除了编程指南外，在类设计过程中，我们还可以应用更具体的设计原则。关于解决低耦合、高内聚和强封装的概念。我们把它们称为SOLID设计原则，这个术语是Robert Cecil Martin在2000年初创造的。

SOLID是以下五个原则的缩写：

* S：单一职责原则\(SRP\) 
* O：开闭原则\(OCP\) 
* L：里氏替换原则\(LSP\)
* I： 接口隔离原则\(ISP\) 
* D：依赖反转原则\(DIP\)

SOLID原则的概念已经有十多年的历史了，但它远远没有过时，因为它们是好的类设计的核心。在本章中，我们将研究这些原则中的每一条，通过观察一些明显违反原则的行为来了解它们。

在本章中，我们将涉及以下主题：

* 单一职责原则
* 开闭原则
* 里氏替换原则 
* 接口隔离原则
* 依赖反转原则

## 单一职责原则

单一职责原则处理的是试图做得太多的类。这里的职责是指改变的原因。按照罗伯特-C-马丁的定义。

> _一个类仅有一个引起它变化的原因_

下面是一个违反SRP的类的例子：

```php
class Ticket {
    const SEVERITY_LOW = 'low';
    const SEVERITY_HIGH = 'high';
    // ...
    protected $title;
    protected $severity;
    protected $status;
    protected $conn;

    public function __construct(\PDO $conn) {
        $this->conn = $conn;
    }

    public function setTitle($title) {
        $this->title = $title;
    }

    public function setSeverity($severity) {
        $this->severity = $severity;
    }

    public function setStatus($status) {
        $this->status = $status;
    }

    private function validate() {
        // Implementation...
    }

    public function save() {
        if ($this->validate()) {
            // Implementation...
        }
    }

}

// Client
$conn = new PDO(/* ... */);
$ticket = new Ticket($conn);
$ticket->setTitle('Checkout not working!');
$ticket->setStatus(Ticket::STATUS_OPEN);
$ticket->setSeverity(Ticket::SEVERITY_HIGH);
$ticket->save();
```

`Ticket`类处理的是 `ticket` 实体的验证和保存到数据库。这两个职责是它改变的两个原因。每当关于票据验证，或者关于票据保存的需求发生变化时，就必须修改`Ticket` 类。为了解决这里的SRP违规，我们可以使用辅助类和接口来分割职责。

下面是一个重构后的实现实例，符合SRP的要求。

```php
interface KeyValuePersistentMembers {
    public function toArray();
}

class Ticket implements KeyValuePersistentMembers {
    const STATUS_OPEN = 'open';
    const SEVERITY_HIGH = 'high';
    //...
    protected $title;
    protected $severity;
    protected $status;

    public function setTitle($title) {
        $this->title = $title;
    }

    public function setSeverity($severity) {
        $this->severity = $severity;
    }

    public function setStatus($status) {
        $this->status = $status;
    }

    public function toArray() {
        // Implementation...
    }
}

class EntityManager {
    protected $conn;

    public function __construct(\PDO $conn) {
        $this->conn = $conn;
    }

    public function save(KeyValuePersistentMembers $entity)
    {
        // Implementation...
    }
}

class Validator {
    public function validate(KeyValuePersistentMembers $entity) {
        // Implementation...
    }
}

// Client
$conn = new PDO(/* ... */);

$ticket = new Ticket();
$ticket->setTitle('Payment not working!');
$ticket->setStatus(Ticket::STATUS_OPEN);
$ticket->setSeverity(Ticket::SEVERITY_HIGH);

$validator = new Validator();

if ($validator->validate($ticket)) {
    $entityManager = new EntityManager($conn);
    $entityManager->save($ticket);
}
```

在这里，我们引入了一个简单的`KeyValuePersistentMembers`接口，它只有一个`toArray`方法，然后与`EntityManager`和`Validator`类一起使用，现在这两个类都只承担一个职责。`Ticket`类变成了一个简单的数据持有模型，而客户端现在控制实例化、验证和保存为三个不同的步骤。虽然这肯定不是如何分离职责的通用公式，但它确实提供了一个简单而清晰的例子。

以单一职责原则为前提进行设计，可以产生更小的类，具有更高的可读性和更容易测试代码。

