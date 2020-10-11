# 使用特性

如果你曾经做过任何C语言编程，你也许会熟悉宏。宏是一个预定义的代码块，它在指定的行处展开。类似地，特性可以包含代码块，这些代码块在PHP解释器指定的行处被复制并粘贴到一个类中。

## 如何做...

1.特性用关键字 `trait` 标识，可以包含属性和方法。你可能已经注意到，在检查之前具有 `CountryList` 和 `CustomerList` 类的案例时，有重复的代码。在这个例子中，我们将重构这两个类，并将 `list()` 方法的功能移到`Trait` 中。请注意，两个类中的 `list()` 方法是一样的。

2.特性用于类与类之间代码重复使用的情况。但请注意，传统的创建抽象类并扩展它的方法可能比 `traits` 更可能有某些优势。特性不能用来确定继承的线路，而抽象父类可以用于这个目的。

3.现在，我们将 `list()` 复制到一个名为 `ListTrait` 的特性中：

```php
trait ListTrait
{
  public function list()
  {
    $list = [];
    $sql  = sprintf('SELECT %s, %s FROM %s', 
      $this->key, $this->value, $this->table);
    $stmt = $this->connection->pdo->query($sql);
    while ($item = $stmt->fetch(PDO::FETCH_ASSOC)) {
      $list[$item[$this->key]] = $item[$this->value];
    }
    return $list;
  }
}
```

4.然后，我们可以将 `ListTrait` 中的代码插入到新类 `CountryListUsingTrait` 中，如以下代码片段所示。 现在可以从此类中删除整个 `list()` 方法：

```php
class CountryListUsingTrait implements ConnectionAwareInterface
{

  use ListTrait;

  protected $connection;
  protected $key   = 'iso3';
  protected $value = 'name';
  protected $table = 'iso_country_codes';

  public function setConnection(Connection $connection)
  {
    $this->connection = $connection;
  }

}
```

{% hint style="warning" %}
每当您有重复的代码进行更改时就会出现潜在的问题。 您可能会发现自己不得不进行过多的全局搜索和替换操作，或者剪切和粘贴代码，而结果往往是灾难性的。 特性是避免这种维护噩梦的好方法。
{% endhint %}

5.特性受命名空间影响。 在步骤1所示的示例中，如果将新的 `CountryListUsingTrait` 类放置在命名空间 `Application\Generic` 中，则我们还需要将 `ListTrait` 也一起移入该命名空间：

```php
namespace Application\Generic;

use PDO;

trait ListTrait
{
  public function list()
  {
    // ...
  }
}
```

6. 特性中的方法将覆盖继承的方法。

7.在下面的例子中，您会注意到 `setId()` 方法的返回值在 `Base` 父类和 `Test` 特性之间有所不同。`Customer` 类继承自 `Base`，但也使用 `Test`。在这种情况下，特性中定义的方法将覆盖 `Base` 父类中定义的方法：

```php
trait Test
{
  public function setId($id)
  {
    $obj = new stdClass();
    $obj->id = $id;
    $this->id = $obj;
  }
}

class Base
{
  protected $id;
  public function getId()
  {
    return $this->id;
  }
  public function setId($id)
  {
    $this->id = $id;
  }
}

class Customer extends Base
{
  use Test;
  protected $name;
  public function getName()
  {
    return $this->name;
  }
  public function setName($name)
  {
    $this->name = $name;
  }
}
```

{% hint style="warning" %}
在 PHP 5 中，`traits` 也可以覆盖属性。在 PHP 7 中，如果 trait 中的属性初始化为与父类不同的值，会产生一个致命的错误。
{% endhint %}

8. 在类中定义了与特性中一样的方法，则在引入特性的时候了类中的方法将覆盖特性中的方法。

9.在这个例子中，`Test` 特性定义了一个属性 `$id` 以及 `getId()` 和`setId()`方法。该特性还定义了 `setName()`，它与 `Customer` 类中定义的相同方法有冲突。在这种情况下，`Customer`类中直接定义的 `setName()`方法将覆盖特性中定义的 `setName()` ：

```php
trait Test
{
  protected $id;
  public function getId()
  {
    return $this->id;
  }
  public function setId($id)
  {
    $this->id = $id;
  }
  public function setName($name)
  {
    $obj = new stdClass();
    $obj->name = $name;
    $this->name = $obj;
  }
}

class Customer
{
  use Test;
  protected $name;
  public function getName()
  {
    return $this->name;
  }
  public function setName($name)
  {
    $this->name = $name;
  }
}
```

10.当使用多个特性时，使用 `insteadof` 关键字来解决方法名冲突。结合使用 `as` 关键字来对方法名进行别名设置。

11.在这个例子中，有两个特性，`IdTrait` 和 `NameTrait`。这两个特性都定义了一个 `setKey()` 方法，但是使用了不同的方式来表达key。`Test` 类使用了这两个特性。注意 `insteadof` 关键字，它允许我们区分冲突的方法。因此，当从 `Test` 类调用 `setKey()` 时，源将从 `NameTrait` 中抽取。此外，来自 `IdTrait` 的 `setKey()` 仍将可用，但用的是别名`setKeyDate()` ：

```php
trait IdTrait
{
  protected $id;
  public $key;
  public function setId($id)
  {
    $this->id = $id;
  }
  public function setKey()
  {
    $this->key = date('YmdHis') 
    . sprintf('%04d', rand(0,9999));
  }
}

trait NameTrait
{
  protected $name;
  public $key;
  public function setName($name)
  {
    $this->name = $name;
  }
  public function setKey()
  {
    $this->key = unpack('H*', random_bytes(18))[1];
  }
}

class Test
{
  use IdTrait, NameTrait {
    NameTrait::setKey insteadof IdTrait;
    IdTrait::setKey as setKeyDate;
  }
}
```

## 如何运行...

从步骤1开始，您了解到特性用于出现重复代码的情况下。 您需要评估是否可以简单地定义一个基类并扩展它，或者使用特性是否更好地满足您的目的。 当在逻辑上不相关的类中看到代码重复时，特性将特别有用。

为了说明特性方法如何覆盖继承的方法，请将步骤7中提到的代码块复制到一个单独的文件 `chap_04_oop_traits_override_inherited.php` 中。 添加以下代码行：

```php
$customer = new Customer();
$customer->setId(100);
$customer->setName('Fred');
var_dump($customer);
```

从输出中可以看到（下图所示），属性 `$id` 中存储了 `stdClass()` 的实例，这就是在特性中定义的行为：

![](../../.gitbook/assets/image%20%2865%29.png)

为了说明直接定义的类方法如何覆盖特性方法，请将步骤9中提到的代码块复制到单独的文件 `chap_04_oop_trait_methods_do_not_override_class_methods.php` 中。 添加以下代码行：

```php
$customer = new Customer();
$customer->setId(100);
$customer->setName('Fred');
var_dump($customer);
```

从以下输出中可以看到，`$id` 属性存储为整数，如在 `Customer` 类中定义的，而特性将 `$id` 定义为 `stdClass` 的实例：

![](../../.gitbook/assets/image%20%2864%29.png)

在步骤10中，您学习了如何在使用多个特性时解决重复的方法名称冲突。 将步骤11中显示的代码块复制到单独的文件 `chap_04_oop_trait_multiple.php` 中。 添加以下代码：

```php
$a = new Test();
$a->setId(100);
$a->setName('Fred');
$a->setKey();
var_dump($a);

$a->setKeyDate();
var_dump($a);
```

请注意，在下面的输出中，`setKey()` 返回的数据是PHP 7新函数`random_bytes()` \(在 `NameTrait中定义`\)产生的，而 `setKeyDate()` 返回的数据是使用 `date()` 和 `rand()` 函数\(在 `IdTrait` 中定义\)产生的：

![](../../.gitbook/assets/image%20%2868%29.png)



