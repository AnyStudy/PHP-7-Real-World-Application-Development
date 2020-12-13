# 创建数组到对象的转化器

Hydrator模式是Data Transfer Object设计模式的一种变体。 它的设计原理非常简单：将数据从一个地方移到另一个地方。 在此，我们将定义类以将数据从数组移动到对象。

## 如何做...

1.首先，我们定义一个能够使用getters和setters的`Hydrator`类。在这个例子中，我们将使用`Application\Generic\Hydrator\GetSet`。

```php
namespace Application\Generic\Hydrator;
class GetSet
{
  // code
}
```

2.接下来，我们定义一个`hydrate()`方法，它同时接受一个数组和一个对象作为参数。然后它调用对象上的`setXXX()`方法来填充数组中的值。我们使用`get_class()`来确定对象的类，然后使用`get_class_methods()`来获取所有方法的列表。 `preg_match()`用于匹配方法的前缀和后缀，后缀被认为是数组的键。

```php
public static function hydrate(array $array, $object)
{
  $class = get_class($object);
  $methodList = get_class_methods($class);
  foreach ($methodList as $method) {
    preg_match('/^(set)(.*?)$/i', $method, $matches);
    $prefix = $matches[1] ?? '';
    $key    = $matches[2] ?? '';
    $key    = strtolower(substr($key, 0, 1)) . substr($key, 1);
    if ($prefix == 'set' && !empty($array[$key])) {
        $object->$method($array[$key]);
    }
  }
  return $object;
}
```

## 如何运行...

为了演示如何使用数组到hydrator对象，首先定义`Application\Generic\Hydrator\Hydrator\GetSet`类，如上述。接下来，定义一个可以用来测试概念的实体类。为了这个说明的目的，创建一个`Application\Entity\Person`类，其中包含适当的属性和方法。确保为所有属性定义getter和setter。这里并没有显示所有这样的方法。

```php
namespace Application\Entity;
class Person
{
  protected $firstName  = '';
  protected $lastName   = '';
  protected $address    = '';
  protected $city       = '';
  protected $stateProv  = '';
  protected $postalCode = '';
  protected $country    = '';

  public function getFirstName()
  {
    return $this->firstName;
  }

  public function setFirstName($firstName)
  {
    $this->firstName = $firstName;
  }

  // etc.
}
```

现在可以创建一个名为`chap_11_array_to_object.php`的调用程序，它设置了自动加载，并使用了相应的类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Entity\Person;
use Application\Generic\Hydrator\GetSet;
```

接下来，可以定义一个测试数组，其值将被添加到一个新的`Person`实例中。

```php
$a['firstName'] = 'Li\'l Abner';
$a['lastName']  = 'Yokum';
$a['address']   = '1 Dirt Street';
$a['city']      = 'Dogpatch';
$a['stateProv'] = 'Kentucky';
$a['postalCode']= '12345';
$a['country']   = 'USA';
```

现在可以以静态方式调用`hydrate()`和`extract()`。

```php
$b = GetSet::hydrate($a, new Person());
var_dump($b);
```

结果如下截图所示。

![](../../.gitbook/assets/image%20%28134%29.png)

