# 实施策略模式

通常情况下，运行时条件会迫使开发人员定义做同一件事的几种方法。传统上，这涉及到大量的if/elseif/else命令块。然后，你将不得不在if语句中定义大量的逻辑块，或者创建一系列函数或方法来实现不同的方法。策略模式试图将这一过程正式化，让主类封装了一系列子类，这些子类代表了解决同一问题的不同方法。

## 如何做...

1.在这个例子中，我们将使用前面定义的`GetSet` hydrator类作为策略。我们将定义一个主`Application\Generic\Hydrator\Any`类，然后它将消耗`Application\Generic\Hydrator\Strategy`命名空间中的策略类，包括`GetSet`、`PublicProps`和`Extending`。

2.我们首先定义类常量，以反映可用的内置策略。

```php
namespace Application\Generic\Hydrator;
use InvalidArgumentException;
use Application\Generic\Hydrator\Strategy\ { 
GetSet, PublicProps, Extending };
class Any
{
  const STRATEGY_PUBLIC  = 'PublicProps';
  const STRATEGY_GET_SET = 'GetSet';
  const STRATEGY_EXTEND  = 'Extending';
  protected $strategies;
  public $chosen;
```

3.然后我们定义一个构造函数，将所有内置策略添加到`$strategies`属性中。

```php
public function __construct()
{
  $this->strategies[self::STRATEGY_GET_SET] = new GetSet();
  $this->strategies[self::STRATEGY_PUBLIC] = new PublicProps();
  $this->strategies[self::STRATEGY_EXTEND] = new Extending();
}
```

4.我们还添加了一个`addStrategy()`方法，使我们可以覆盖或添加新的策略，而无需重新编写类。

```php
public function addStrategy($key, HydratorInterface $strategy)
{
  $this->strategies[$key] = $strategy;
}
```

5.`hydrate()`和`extract()`方法只是简单地调用了所选策略的方法。

```php
public function hydrate(array $array, $object)
{
  $strategy = $this->chooseStrategy($object);
  $this->chosen = get_class($strategy);
  return $strategy::hydrate($array, $object);
}

public function extract($object)
{
  $strategy = $this->chooseStrategy($object);
  $this->chosen = get_class($strategy);
  return $strategy::extract($object);
}
```

6.棘手的是要弄清楚选择哪种策略。为此我们定义了 `chooseStrategy()`，它以一个对象作为参数。我们首先通过获取一个类方法的列表来执行一些检测工作。然后我们扫描列表，看看是否有`getXXX()`或`setXXX()`方法。如果有，我们就选择`GetSet` hydrator作为我们选择的策略。

```php
public function chooseStrategy($object)
{
  $strategy = NULL;
  $methodList = get_class_methods(get_class($object));
  if (!empty($methodList) && is_array($methodList)) {
      $getSet = FALSE;
      foreach ($methodList as $method) {
        if (preg_match('/^get|set.*$/i', $method)) {
            $strategy = $this->strategies[self::STRATEGY_GET_SET];
      break;
    }
  }
}
```

7.还是在我们的 `chooseStrategy()`方法中，如果没有getter或setter，我们接下来使用`get_class_vars()`来确定是否有任何可用的属性。如果有的话，我们选择`PublicProps`作为我们的转化器。

```php
if (!$strategy) {
    $vars = get_class_vars(get_class($object));
    if (!empty($vars) && count($vars)) {
        $strategy = $this->strategies[self::STRATEGY_PUBLIC];
    }
}
```

8.如果所有其他方法都失败了，我们又回到了`Extending` hydrator，它返回一个新的类，简单地扩展对象类，从而使任何公共或保护的属性可用。

```php
if (!$strategy) {
    $strategy = $this->strategies[self::STRATEGY_EXTEND];
}
return $strategy;
}
}
```

9.现在我们将注意力转向策略本身。首先，我们定义一个新的`Application\Generic\Hydrator\Strategy`命名空间。

10.在新的命名空间中，我们定义了一个接口，允许我们识别任何可以被`Application\Generic\Hydrator\Any`消耗的策略。

```php
namespace Application\Generic\Hydrator\Strategy;
interface HydratorInterface
{
  public static function hydrate(array $array, $object);
  public static function extract($object);
}
```

11. `GetSet`转化器与前面两个配方中定义的完全一样，唯一增加的是它将实现新的接口。

```php
namespace Application\Generic\Hydrator\Strategy;
class GetSet implements HydratorInterface
{

  public static function hydrate(array $array, $object)
  {
    // defined in the recipe:
    // "Creating an Array to Object Hydrator"
  }

  public static function extract($object)
  {
    // defined in the recipe:
    // "Building an Object to Array Hydrator"
  }
}
```

12. 下一个hydrator只是简单地读写公共属性。

```php
namespace Application\Generic\Hydrator\Strategy;
class PublicProps implements HydratorInterface
{
  public static function hydrate(array $array, $object)
  {
    $propertyList= array_keys(
      get_class_vars(get_class($object)));
    foreach ($propertyList as $property) {
      $object->$property = $array[$property] ?? NULL;
    }
    return $object;
  }

  public static function extract($object)
  {
    $array = array();
    $propertyList = array_keys(
      get_class_vars(get_class($object)));
    foreach ($propertyList as $property) {
      $array[$property] = $object->$property;
    }
    return $array;
  }
}
```

13. 最后，水兵的瑞士军刀--Extending，扩展了对象类，从而提供了对属性的直接访问。我们进一步定义了神奇的getter和setter来提供对属性的访问。

14.hydrate\(\)方法是最困难的，因为我们假设没有定义getter或setter，也没有定义可见性级别为public的属性。相应地，我们需要定义一个类，该类扩展了要被转化的对象的类。我们的做法是首先定义一个字符串，这个字符串将被用作构建新类的模板。

```php
namespace Application\Generic\Hydrator\Strategy;
class Extending implements HydratorInterface
{
  const UNDEFINED_PREFIX = 'undefined';
  const TEMP_PREFIX = 'TEMP_';
  const ERROR_EVAL = 'ERROR: unable to evaluate object';
  public static function hydrate(array $array, $object)
  {
    $className = get_class($object);
    $components = explode('\\', $className);
    $realClass  = array_pop($components);
    $nameSpace  = implode('\\', $components);
    $tempClass = $realClass . self::TEMP_SUFFIX;
    $template = 'namespace ' 
      . $nameSpace . '{'
      . 'class ' . $tempClass 
      . ' extends ' . $realClass . ' '
```

15.继续使用hydrate\(\)方法，我们定义了一个$values属性和一个构造函数，该构造函数将要转化的数组作为参数分配到对象中。我们在值的数组中循环，将值分配给属性。我们还定义了一个有用的 `getArrayCopy()` 方法，在需要时返回这些值，以及一个神奇的 `__get()` 方法来模拟直接属性访问。

```php
. '{ '
. '  protected $values; '
. '  public function __construct($array) '
. '  { $this->values = $array; '
. '    foreach ($array as $key => $value) '
. '       $this->$key = $value; '
. '  } '
. '  public function getArrayCopy() '
. '  { return $this->values; } '
```

16.为了方便起见，我们定义了一个神奇的 `__get()` 方法，它可以模拟直接访问变量，就像它们是公共的一样。

```php
. '  public function __get($key) '
. '  { return $this->values[$key] ?? NULL; } '
```

17.还是在新类的模板中，我们还定义了一个神奇的`__call()`方法，它可以模拟getters和setters。

```php
. '  public function __call($method, $params) '
. '  { '
. '    preg_match("/^(get|set)(.*?)$/i", '
. '        $method, $matches); '
. '    $prefix = $matches[1] ?? ""; '
. '    $key    = $matches[2] ?? ""; '
. '    $key    = strtolower(substr($key, 0, 1)) ' 
. '              substr($key, 1); '
. '    if ($prefix == "get") { '
. '        return $this->values[$key] ?? NULL; '
. '     } else { '
. '        $this->values[$key] = $params[0]; '
. '     } '
. '  } '
. '} '
. '} // ends namespace ' . PHP_EOL
```

18. 最后，还是在新类的模板中，我们在全局命名空间中添加一个函数，用于构建和返回类实例。

```php
. 'namespace { '
. 'function build($array) '
. '{ return new ' . $nameSpace . '\\' 
.    $tempClass . '($array); } '
. '} // ends global namespace '
. PHP_EOL;
```

19. 还是在`hydrate()`方法中，我们使用`eval()`执行完成的模板。然后我们运行刚刚在模板最后定义的`build()`方法。注意，由于我们不确定要填充的类的命名空间，我们从全局命名空间定义并调用`build()`。

```php
try {
    eval($template);
} catch (ParseError $e) {
    error_log(__METHOD__ . ':' . $e->getMessage());
    throw new Exception(self::ERROR_EVAL);
}
return \build($array);
}
```

20. `extract()`方法更容易定义，因为我们的选择极其有限。使用神奇的方法扩展一个类并从一个数组中填充它是很容易实现的。反之则不然。如果我们要扩展这个类，我们将失去所有的属性值，因为我们是在扩展这个类，而不是对象实例。相应地，我们唯一的选择是使用获取器和公共属性的组合。

```php
public static function extract($object)
{
  $array = array();
  $class = get_class($object);
  $methodList = get_class_methods($class);
  foreach ($methodList as $method) {
    preg_match('/^(get)(.*?)$/i', $method, $matches);
    $prefix = $matches[1] ?? '';
    $key    = $matches[2] ?? '';
    $key    = strtolower(substr($key, 0, 1)) 
    . substr($key, 1);
    if ($prefix == 'get') {
        $array[$key] = $object->$method();
    }
  }
  $propertyList= array_keys(get_class_vars($class));
  foreach ($propertyList as $property) {
    $array[$property] = $object->$property;
  }
  return $array;
  }
}

```

## 如何运行...

你可以先定义三个具有相同属性的测试类：`firstName`，`lastName`，等等。第一个类，`Person`，应该有受保护的属性以及getters和setters。第二个，`PublicPerson`，将有公共属性。第三个，`ProtectedPerson`，有受保护的属性，但没有`getters和setters`。

```php
<?php
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

  // be sure to define remaining getters and setters

}

<?php
namespace Application\Entity;
class PublicPerson
{
  private $id = NULL;
  public $firstName  = '';
  public $lastName   = '';
  public $address    = '';
  public $city       = '';
  public $stateProv  = '';
  public $postalCode = '';
  public $country    = '';
}

<?php
namespace Application\Entity;

class ProtectedPerson
{
  private $id = NULL;
  protected $firstName  = '';
  protected $lastName   = '';
  protected $address    = '';
  protected $city       = '';
  protected $stateProv  = '';
  protected $postalCode = '';
  protected $country    = '';
}
```

现在你可以定义一个名为`chap_11_strategy_pattern.php`的调用程序，它可以设置自动加载并使用相应的类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Entity\ { Person, PublicPerson, ProtectedPerson };
use Application\Generic\Hydrator\Any;
use Application\Generic\Hydrator\Strategy\ { GetSet, Extending, PublicProps };
```

接下来，创建一个`Person`的实例，并运行setters来定义属性的值。

```php
$obj = new Person();
$obj->setFirstName('Li\'lAbner');
$obj->setLastName('Yokum');
$obj->setAddress('1 Dirt Street');
$obj->setCity('Dogpatch');
$obj->setStateProv('Kentucky');
$obj->setPostalCode('12345');
$obj->setCountry('USA');
```

接下来，创建一个`Any` hydrator的实例，调用`extract()`，并使用`var_dump()`来查看结果。

```php
$hydrator = new Any();
$b = $hydrator->extract($obj);
echo "\nChosen Strategy: " . $hydrator->chosen . "\n";
var_dump($b);
```

从下面的输出中观察，选择了`GetSet`策略。

![](../../.gitbook/assets/image%20%28135%29.png)

{% hint style="info" %}
请注意，由于其可见性级别是私有的，所以没有设置id属性。
{% endhint %}

接下来，你可以定义一个具有相同值的数组。在`Any`实例上调用`hydrate()`，并提供一个新的`PublicPerson`实例作为参数。

```php
$a = [
  'firstName'  => 'Li\'lAbner',
  'lastName'   => 'Yokum',
  'address'    => '1 Dirt Street',
  'city'       => 'Dogpatch',
  'stateProv'  => 'Kentucky',
  'postalCode' => '12345',
  'country'    => 'USA'
];

$p = $hydrator->hydrate($a, new PublicPerson());
echo "\nChosen Strategy: " . $hydrator->chosen . "\n";
var_dump($p);
```

这就是结果。请注意，在这种情况下选择了`PublicProps`策略。

![](../../.gitbook/assets/image%20%28138%29.png)

最后，再次调用`hydrate()`，但这次提供一个`ProtectedPerson`的实例作为对象参数。然后我们调用`getFirstName()`和`getLastName()`来测试神奇的获取器。我们还将名字和姓氏作为直接变量访问。

```php
$q = $hydrator->hydrate($a, new ProtectedPerson());
echo "\nChosen Strategy: " . $hydrator->chosen . "\n";
echo "Name: {$q->getFirstName()} {$q->getLastName()}\n";
echo "Name: {$q->firstName} {$q->lastName}\n";
var_dump($q);
```

这是最后的输出，显示选择了`Extending`策略。你还会注意到，这个实例是一个新的`ProtectedPerson_TEMP`类，而且受保护的属性已经被完全填充。

![](../../.gitbook/assets/image%20%28137%29.png)

