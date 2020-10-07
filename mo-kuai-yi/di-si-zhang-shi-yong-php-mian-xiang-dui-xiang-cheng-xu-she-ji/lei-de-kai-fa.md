# 类的开发

传统的开发方法是将类放入自己的文件中。 通常，类包含实现单一目的的逻辑。 将类进一步细分为自包含的函数，这些函数称为方法。 在类内部定义的变量称为属性。 建议同时开发一个测试类，第13章，最佳实践，测试和调试中将更详细地讨论该主题。

## 如何做...

1.创建一个文件以包含类定义。 为了自动加载，建议文件名与类名匹配。 在文件顶部的 `class` 关键字之前，添加一个 **DocBlock**。 然后，您就可以定义属性和方法了。 在此示例中，我们定义了一个 `Test` 类。 它具有属性 `$test` 和方法 `getTest()`：

```php
<?php
declare(strict_types=1);
/**
 * 这是一个示例类
 *
 * 这个类的目的是获取和设置一个受保护的属性 $test
 *
 */
class Test
{

  protected $test = 'TEST';

  /**
   * 此方法返回 $test 的当前值
   *
   * @return string $test
   */
  public function getTest() : string
  {
    return $this->test;
  }

  /**
   * 此方法设置 $test 的值
   *
   * @param string $test
   * @return Test $this
   */
  public function setTest(string $test)
  {
    $this->test = $test;
    return $this;
  }
}
```

> **小贴士**
>
> **最佳实践**
>
> 最好的做法是以类的名字命名文件。尽管PHP中的类名不区分大小写，但最好的做法是首字母大写。不应该在类定义文件中放入可执行代码。
>
> 每个类都应该在 `class` ****关键字之前包含一个**DocBlock**。在**DocBlock**中，您应该简短描述一下类目的。跳过一行，然后包含更详细的描述。你也可以包含 `@` 标签，如 `@author`、`@license`等。每个方法前面同样应该有一个**DocBlock**，标识该方法的目的，以及它的传入参数和返回值。

2.每个文件可以定义多个类，但这不是最好的做法。在这个例子中，我们创建了一个文件，`NameAddress.php`，其中定义了两个类，`Name`和`Address`：

```php
<?php
declare(strict_types=1);
class Name
{

  protected $name = '';

  public function getName() : string
  {
    return $this->name;
  }

  public function setName(string $name)
  {
    $this->name = $name;

    return $this;
  }
}

class Address
{

  protected $address = '';

  public function getAddress() : string
  {
    return $this->address;
  }

  public function setAddress(string $address)
  {
    $this->address = $address;
    return $this;
  }
}
```

> **小贴士**
>
> 虽然您可以在一个文件中定义多个类，如前面的代码片段所示，但这并不是最好的做法。这不仅否定了文件的逻辑纯度，而且使自动加载更加困难。

3. 类名不区分大小写。重复将被标记为错误。在这个例子中，在 `TwoClass.php` 文件中，我们定义了两个类，`TwoClass` 和 `twoclass`：

```php
<?php
class TwoClass
{
  public function showOne()
  {
    return 'ONE';
  }
}

// 解析第二个类定义时将发生致命错误
class twoclass
{
  public function showTwo()
  {
    return 'TWO';
  }
}
```

4.PHP 7.1 解决了关键字 `$this` 的使用不一致的问题。尽管在 PHP 7.0 和 PHP 5.x 中是允许的，但从 PHP 7.1 开始，以下任何 `$this` 的使用都会产生错误：

* 参数 
* 静态变量 
* 全局变量 
* 用于 `try...catch` 块的变量 
* `foreach()` 中使用的变量 
* 作为 `unset()` 的参数
* 作为变量（即 `$a ='this'; echo $$a`）
* 通过间接引用

5.如果您需要创建一个对象实例，但又不想定义一个独立的类，您可以使用 PHP 中内置的通用`stdClass`，`stdClass`允许你在运行中定义属性，而不必定义一个扩展`stdClass`的独立类：

```php
$obj = new stdClass();
```

6.这个功能在 PHP 中的很多地方都有使用。例如，当你使用**PHP数据对象**\(**PDO**\)进行数据库查询时，其中一个获取模式是`PDO::FETCH_OBJ`。这种模式返回的是 `stdClass` 的实例，其中的属性代表数据库表列：

```php
$stmt = $connection->pdo->query($sql);
$row  = $stmt->fetch(PDO::FETCH_OBJ);
```

## 如何运行...

以前面代码片断中的 `Test` 类为例，将代码放在一个名为 `Test.php`的文件中，然后创建另一个名为 `chap_04_oop_defining_class_test.php` 的文件。再创建一个名为 `chap_04_oop_defining_class_test.php` 的文件。添加以下代码：

```php
require __DIR__ . '/Test.php';

$test = new Test();
echo $test->getTest();
echo PHP_EOL;

$test->setTest('ABC');
echo $test->getTest();
echo PHP_EOL;
```

输出将显示 `$test` 属性的初始值，之后是通过调用 `setTest()` 修改的新值：

![](../../.gitbook/assets/image%20%2839%29.png)

下一个例子在 `NameAddress.php` 中定义两个类，`Name`和`Address`。您可以用下面的代码来调用和使用这两个类：

```php
require __DIR__ . '/NameAddress.php';

$name = new Name();
$name->setName('TEST');
$addr = new Address();
$addr->setAddress('123 Main Street');

echo $name->getName() . ' lives at ' . $addr->getAddress();

```

{% hint style="warning" %}
虽然PHP解释器不会产生任何错误，但通过定义多个类，文件的逻辑纯度受到影响。另外，文件名与类名不匹配，会影响自动加载的能力。
{% endhint %}

接下来是这个例子的输出：

![](../../.gitbook/assets/image%20%2847%29.png)

步骤3也展示了在一个文件中定义两个类，但在本例中，目的是证明 PHP 中的类名是不区分大小写的。将代码放入文件 `TwoClass.php` 中。 当您尝试包含文件时，将生成错误：

![](../../.gitbook/assets/image%20%2844%29.png)

为了演示直接使用 `stdClass`创建一个实例，给一个属性赋值，然后使用 `var_dump()` 来显示结果。要了解 `stdClass` 是如何在内部使用的，使用 `var_dump()` 来显示一个 `PDO` 查询的结果，其中 fetch 模式被设置为 `FETCH_OBJ`。

输入以下代码：

```php
$obj = new stdClass();
$obj->test = 'TEST';
echo $obj->test;
echo PHP_EOL;

include (__DIR__ . '/../Application/Database/Connection.php');
$connection = new Application\Database\Connection(
  include __DIR__ . DB_CONFIG_FILE);

$sql  = 'SELECT * FROM iso_country_codes';
$stmt = $connection->pdo->query($sql);
$row  = $stmt->fetch(PDO::FETCH_OBJ);
```

这是输出：

![](../../.gitbook/assets/image%20%2846%29.png)

## 参考

关于 PHP 7.1 中关键字 `$this` 的更多信息，请参见 [https://wiki.php.net/rfc/this\_var](https://wiki.php.net/rfc/this_var)。



