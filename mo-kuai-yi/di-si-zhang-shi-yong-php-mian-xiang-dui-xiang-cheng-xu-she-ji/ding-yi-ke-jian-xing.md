# 定义可见性

可以理解，可见性一词与应用程序安全性无关！ 相反，它只是一种控制代码使用的机制。 它可以用来引导没有经验的开发人员远离只能在类定义内部调用的方法的公共使用。

## 如何做...

1.通过在任何属性或方法定义之前添加 `public`，`protected`或`private`关键字来指示可见性级别。 您可以将属性标记为受保护或私有，以强制只通过公共的`getter`和`setter`进行访问。

2.在这个例子中，定义了一个`Base`类，它有一个受保护的属性`$id`，为了访问这个属性，定义了`getId()`和`setId()`两个公共方法。受保护的方法`generateRandId()`可以在内部使用，并且在`Customer`子类中被继承。这个方法不能在类定义之外直接调用。注意使用新的PHP 7 `random_bytes()`函数来创建一个随机ID。

```php
class Base
{
  protected $id;
  private $key = 12345;
  public function getId()
  {
    return $this->id;
  }
  public function setId()
  {
    $this->id = $this->generateRandId();
  }
  protected function generateRandId()
  {
    return unpack('H*', random_bytes(8))[1];
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

{% hint style="warning" %}
**最佳实践**

将属性标记为受保护，并定义公共的 ****getNameOfProperty\(\) 和 setNameOfProperty\(\) 方法来控制对属性的访问。这种方法被称为 `getters` 和 `setters`。
{% endhint %}

3.将一个属性或方法标记为私有，以防止它被继承或在类定义之外可见。这是将类创建为**单例**的好方法。

4.下一个代码示例显示了一个类 `Registry`，它只能有一个实例。因为构造函数被标记为私有，所以只有通过静态方法 `getInstance()`才能创建实例：

```php
class Registry
{
  protected static $instance = NULL;
  protected $registry = array();
  private function __construct()
  {
    // 没有人可以创建这个类的实例
  }
  public static function getInstance()
  {
    if (!self::$instance) {
      self::$instance = new self();
    }
    return self::$instance;
  }
  public function __get($key)
  {
    return $this->registry[$key] ?? NULL;
  }
  public function __set($key, $value)
  {
    $this->registry[$key] = $value;
  }
}

```

{% hint style="warning" %}
您可以将方法标记为 `final`，以防止其被覆盖。将一个类标记为 `final`，以防止它被扩展。
{% endhint %}

5.通常情况下，类常量的可见性级别为 `public`，而在 PHP 7.1 中，可以将类常量声明为 `protected`或`private`。在下面的例子中，`TEST_WHOLE_WORLD` 类常量的行为与 PHP 5 中的完全相同。 接下来的两个常量，`TEST_INHERITED` 和 `TEST_LOCAL`，遵循与任何受保护或私有属性或方法相同的规则：

```php
class Test
{

  public const TEST_WHOLE_WORLD  = 'visible.everywhere';

  // NOTE: 只能在 PHP 7.1及以上版本中工作
  protected const TEST_INHERITED = 'visible.in.child.classes';

  // NOTE: 只能在 PHP 7.1及以上版本中工作
  private const TEST_LOCAL= 'local.to.class.Test.only';

  public static function getTestInherited()
  {
    return static::TEST_INHERITED;
  }

  public static function getTestLocal()
  {
    return static::TEST_LOCAL;
  }

}
```

## 如何运行...

创建文件`chap_04_basic_visibility.php`并定义两个类：`Base`和`Customer`。 接下来，编写代码以创建每个实例：

```php
$base     = new Base();
$customer = new Customer();
```

请注意，下面的代码工作正常，事实上被认为是最佳实践：

```php
$customer->setId();
$customer->setName('Test');
echo 'Welcome ' . $customer->getName() . PHP_EOL;
echo 'Your new ID number is: ' . $customer->getId() . PHP_EOL;
```

尽管 `$id` 是受保护的，但相应的方法 `getId()` 和 `setId()` 都是公共的，因此可以从类定义之外访问。下面是输出结果：

![](../../.gitbook/assets/image%20%2842%29.png)

然而，下面的代码行将无法工作，因为私有和受保护的属性无法从类定义之外访问：

```php
echo 'Key (does not work): ' . $base->key;
echo 'Key (does not work): ' . $customer->key;
echo 'Name (does not work): ' . $customer->name;
echo 'Random ID (does not work): ' . $customer->generateRandId();
```

下面的输出显示了预期的错误：

![](../../.gitbook/assets/image%20%2853%29.png)

## 参考

有关 `getter` 和 `setter` 的更多信息，请参阅第十章标题为《使用 `getter` 和 `setter` 》的章节。有关PHP 7.1类常量可见性设置的更多信息，请参见[https://wiki.php.net/rfc/class\_const\_visibility](https://wiki.php.net/rfc/class_const_visibility)。

