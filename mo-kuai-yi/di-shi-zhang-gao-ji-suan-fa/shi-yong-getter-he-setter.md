# 使用 getter 和 setter

乍一看，定义具有公共属性的类似乎是有意义的，这些属性可以直接被读取或写入。然而，我们认为最好的做法是让属性受保护，然后为每个属性定义一个getter和setter。顾名思义，getter用于检索属性的值。setter用于设置值。

{% hint style="info" %}
**最佳实践**

将属性定义为受保护的，以防止意外的外部访问。使用公共的 get\* 和 __set\* 方法来提供对这些属性的访问。通过这种方式，不仅可以更精确地控制访问，还可以在获取和设置属性的同时对其进行格式和数据类型的更改。
{% endhint %}

## 如何做...

1.当获取或设置值时，Getters和setters提供了额外的灵活性。如果需要的话，你可以添加一个额外的逻辑层，如果你直接读取或写入一个公共属性，这是不可能的。你需要做的就是创建一个前缀为 `get` 或 `set` 的公共方法。属性的名称就成了后缀。惯例是让变量的第一个字母大写。因此，如果属性是 `$testValue` ，那么getter将是 `getTestValue()` 。

2. 在这个例子中，我们定义了一个带有保护属性 `$date` 的类。请注意，`get` 和 `set` 方法允许作为 `DateTime` 对象或字符串来处理。在任何情况下，该值实际上都是作为一个 `DateTime` 实例存储的。

```php
$a = new class() {
  protected $date;
  public function setDate($date)
  {
    if (is_string($date)) {
        $this->date = new DateTime($date);
    } else {
        $this->date = $date;
    }
  }
  public function getDate($asString = FALSE)
  {
    if ($asString) {
        return $this->date->format('Y-m-d H:i:s');
    } else {
        return $this->date;
    }
  }
};
```

3. getters和setters允许你过滤进入或流出的数据。在下面的例子中，有两个属性，`$intVal`和`$arrVal`，它们被设置为默认的初始值 `NULL`。注意，不仅获取者的返回值是数据类型的，而且还提供了默认值。设置器还强制执行传入的数据类型，或者将传入的值类型化为某种数据类型。

```php
<?php
class GetSet
{
  protected $intVal = NULL;
  protected $arrVal = NULL;
  // note the use of the null coalesce operator to return a default value
  public function getIntVal() : int
  {
    return $this->intVal ?? 0;
  }
  public function getArrVal() : array
  {
    return $this->arrVal ?? array();
  }
  public function setIntVal($val)
  {
    $this->intVal = (int) $val ?? 0;
  }
  public function setArrVal(array $val)
  {
    $this->arrVal = $val ?? array();
  }
}
```

4.  如果有一个有很多很多属性的类，为每个属性定义一个不同的getter和setter可能会变得很乏味。在这种情况下，你可以使用神奇的 `__call()` 方法定义一种回调。下面这个类定义了九个不同的属性。我们不必定义九个getter和九个setter，而是定义了一个方法 `__call()`，它可以确定使用方式是 `get` 还是 `set`。如果是 `get`，它从内部数组中检索键。如果是set，它将值存储在内部数组中。

{% hint style="info" %}
`__call()` 方法是一个神奇的方法，如果应用程序对一个不存在的方法进行调用，它就会被执行。
{% endhint %}

```php
<?php
class LotsProps
{
  protected $firstName  = NULL;
  protected $lastName   = NULL;
  protected $addr1      = NULL;
  protected $addr2      = NULL;
  protected $city       = NULL;
  protected $state      = NULL;
  protected $province   = NULL;
  protected $postalCode = NULL;
  protected $country    = NULL;
  protected $values     = array();
    
  public function __call($method, $params)
  {
    preg_match('/^(get|set)(.*?)$/i', $method, $matches);
    $prefix = $matches[1] ?? '';
    $key    = $matches[2] ?? '';
    $key    = strtolower($key);
    if ($prefix == 'get') {
        return $this->values[$key] ?? '---';
    } else {
        $this->values[$key] = $params[0];
    }
  }
}
```

## 如何运行...

将步骤1中提到的代码复制到一个新的文件中，`chap_10_oop_using_getters_and_setters.php`。为了测试该类，添加以下内容。

```php
// set date using a string
$a->setDate('2015-01-01');
var_dump($a->getDate());

// retrieves the DateTime instance
var_dump($a->getDate(TRUE));

// set date using a DateTime instance
$a->setDate(new DateTime('now'));
var_dump($a->getDate());

// retrieves the DateTime instance
var_dump($a->getDate(TRUE));
```

在输出中（如下图所示），可以看到 `$date` 属性可以使用字符串或实际的 `DateTime` 实例来设置。当执行 `getDate()` 时，你可以根据 `$asString` 标志的值返回一个字符串或一个 `DateTime` 实例。

![](../../.gitbook/assets/image%20%28121%29.png)

接下来，看一下第2步中定义的代码。将这段代码复制到文件`chap_10_oop_using_getters_and_setters_defaults.php` 中，并添加以下内容。

```php
// create the instance
$a = new GetSet();

// set a "proper" value
$a->setIntVal(1234);
echo $a->getIntVal();
echo PHP_EOL;

// set a bogus value
$a->setIntVal('some bogus value');
echo $a->getIntVal();
echo PHP_EOL;

// NOTE: boolean TRUE == 1
$a->setIntVal(TRUE);
echo $a->getIntVal();
echo PHP_EOL;

// returns array() even though no value was set
var_dump($a->getArrVal());
echo PHP_EOL;

// sets a "proper" value
$a->setArrVal(['A','B','C']);
var_dump($a->getArrVal());
echo PHP_EOL;

try {
    $a->setArrVal('this is not an array');
    var_dump($a->getArrVal());
    echo PHP_EOL;
} catch (TypeError $e) {
    echo $e->getMessage();
}

echo PHP_EOL;
```

正如你可以从下面的输出中看到的，设置一个合适的整数值和预期的一样，非数字值默认为`0`。有趣的是，如果你提供一个布尔值 `true` 作为参数给`settingVal()`，它就会被插值为`1`。

如果你调用`getArrVal()`而不设置一个值，默认是一个空数组。设置一个数组值的工作原理和预期一样。然而，如果你提供一个非数组值作为参数，数组的类型提示会导致抛出`TypeError`，这个错误可以如这里所示。

![](../../.gitbook/assets/image%20%28119%29.png)

最后，把步骤3中定义的`LotsProps`类放到一个单独的文件中，`chap_10_oop_using_getters_and_setters_magic_call.php`。现在添加代码来设置值。当然，会发生的是调用了神奇的方法`__call()`。在运行`preg_match()`后，不存在的属性的剩余部分，在字母设置后，将成为内部数组`$values`中的一个键。

```php
$a = new LotsProps();
$a->setFirstName('Li\'l Abner');
$a->setLastName('Yokum');
$a->setAddr1('1 Dirt Street');
$a->setCity('Dogpatch');
$a->setState('Kentucky');
$a->setPostalCode('12345');
$a->setCountry('USA');
?>
```

然后，你可以定义HTML，使用相应的 `get` 方法显示这些值。这些方法又会从内部数组中返回键。

```php
<div class="container">
<div class="left blue1">Name</div>
<div class="right yellow1">
<?= $a->getFirstName() . ' ' . $a->getLastName() ?></div>   
</div>
<div class="left blue2">Address</div>
<div class="right yellow2">
    <?= $a->getAddr1() ?>
    <br><?= $a->getAddr2() ?>
    <br><?= $a->getCity() ?>
    <br><?= $a->getState() ?>
    <br><?= $a->getProvince() ?>
    <br><?= $a->getPostalCode() ?>
    <br><?= $a->getCountry() ?>
</div>   
</div>
```

这是最后的输出。

![](../../.gitbook/assets/image%20%28120%29.png)

