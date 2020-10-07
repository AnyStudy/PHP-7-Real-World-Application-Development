# 类的扩展

开发者使用 OOP 的主要原因之一是它能够重用现有的代码，同时，还可以添加或覆盖功能。在 PHP 中，关键字 `extends` 用来建立类之间的父/子关系。

## 如何做...

1.在子类中，使用关键字 `extends` 来设置继承。在下面的例子中，`Customer` 类扩展了 `Base` 类。 `Customer` 的任何实例都将继承可见的方法和属性，在本例中为 `$id`、`getId()` 和 `setId()` ：

```php
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

2.您可以强迫任何使用你的类的开发者通过标记为 `abstract` 的方法来定义一个方法。在这个例子中，`Base` 类将 `validate()` 方法定义为 `abstract`。之所以它必须是抽象的，是因为从父类 `Base` 类的角度无法准确地确定子类如何验证：

```php
abstract class Base
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
  public function validate();
}
```

{% hint style="warning" %}
如果一个类包含一个抽象方法，那么这个类本身必须声明为抽象的。
{% endhint %}

3.PHP 仅支持单线继承。 下面的示例显示了一个类 `Member`，它继承了 `Customer` 。而 `Customer` 则继承自 `Base`：

```php
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

class Member extends Customer
{
  protected $membership;
  public function getMembership()
  {
    return $this->membership;
  }
  public function setMembership($memberId)
  {
    $this->membership = $memberId;
  }
}
```

4.为了满足类型提示，可以使用目标类的任何子类。在下面的代码片段中显示的 `test()` 函数，需要一个 `Base` 类的实例作为参数。继承线内的任何类都可以被接受为参数。传递给 `test()` 的其他任何东西都会抛出一个 `TypeError` ：

```php
function test(Base $object)
{
  return $object->getId();
}
```

## 如何运行...

在第一个要点中，定义了一个 `Base` 类和一个 `Customer` 类。为了便于演示，将这两个类的定义放在一个文件中，即 `chap_04_oop_extends.php` ，并添加以下代码：

```php
$customer = new Customer();
$customer->setId(100);
$customer->setName('Fred');
var_dump($customer);
```

请注意，`$id` 属性以及 `getId()` 和 `setId()` 方法都是从父类 `Base` 类继承到子类 `Customer` 中的：

![](../../.gitbook/assets/image%20%2850%29.png)

为了说明抽象方法的使用，想象一下，你希望给任何扩展 `Base` 的类添加某种验证能力。问题是，我们无法知道在继承的类中可能会验证什么。唯一可以确定的是，你必须有一个验证能力。

把前面解释中提到的同一个 `Base` 类，添加一个新的方法 `validate()` 。将该方法标记为抽象方法，并且不要定义任何代码。注意当子类 `Customer` 类扩展 `Base` 时，会发生什么。

![](../../.gitbook/assets/image%20%2849%29.png)

如果你再把 `Base` 类标记为抽象类，但没有在子类中定义一个 `validate()` 方法，就会产生同样的错误。最后，继续在子类 `Customer` 中实现 `validate()` 方法：

```php
class Customer extends Base
{
  protected $name;
  public function getName()
  {
    return $this->name;
  }
  public function setName($name)
  {
    $this->name = $name;
  }
  public function validate()
  {
    $valid = 0;
    $count = count(get_object_vars($this));
    if (!empty($this->id) &&is_int($this->id)) $valid++;
    if (!empty($this->name) 
    &&preg_match('/[a-z0-9 ]/i', $this->name)) $valid++;
    return ($valid == $count);
  }
}
```

然后你可以添加以下代码来测试结果：

```php
$customer = new Customer();

$customer->setId(100);
$customer->setName('Fred');
echo "Customer [id]: {$customer->getName()}" .
     . "[{$customer->getId()}]\n";
echo ($customer->validate()) ? 'VALID' : 'NOT VALID';
$customer->setId('XXX');
$customer->setName('$%£&*()');
echo "Customer [id]: {$customer->getName()}"
  . "[{$customer->getId()}]\n";
echo ($customer->validate()) ? 'VALID' : 'NOT VALID';
```

这是输出：

![](../../.gitbook/assets/image%20%2842%29.png)

为了展示单线继承，在前面步骤1所示的 `Base` 和 `Customer` 的第一个例子中添加一个新的 `Member` 类：

```php
{
  protected $membership;
  public function getMembership()
  {
    return $this->membership;
  }
  public function setMembership($memberId)
  {
    $this->membership = $memberId;
  }
}
```

创建一个 `Member` 的实例，注意，在下面的代码中，所有的属性和方法都可以从每个继承类中获得，即使不是直接继承的：

```php
$member = new Member();
$member->setId(100);
$member->setName('Fred');
$member->setMembership('A299F322');
var_dump($member);
```

这是输出：

![](../../.gitbook/assets/image%20%2838%29.png)

现在定义一个函数 `test()`，该函数将 `Base` 的实例作为参数：

```php
function test(Base $object)
{
  return $object->getId();
}
```

请注意，`Base`、`Customer`和 `Member`的实例都可以作为参数。

```php
$base = new Base();
$base->setId(100);

$customer = new Customer();
$customer->setId(101);

$member = new Member();
$member->setId(102);

// 3个类都可以在 test() 中工作
echo test($base)     . PHP_EOL;
echo test($customer) . PHP_EOL;
echo test($member)   . PHP_EOL;
```

这是输出：

![](../../.gitbook/assets/image%20%2847%29.png)

但是，如果你试图用一个不在继承线中的对象实例运行 `test()` ，就会抛出 `TypeError`：

```php
class Orphan
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
try {
    $orphan = new Orphan();
    $orphan->setId(103);
    echo test($orphan) . PHP_EOL;
} catch (TypeError $e) {
    echo 'Does not work!' . PHP_EOL;
    echo $e->getMessage();
}
```

我们可以从下图中观察到这一点：

![](../../.gitbook/assets/image%20%2840%29.png)



