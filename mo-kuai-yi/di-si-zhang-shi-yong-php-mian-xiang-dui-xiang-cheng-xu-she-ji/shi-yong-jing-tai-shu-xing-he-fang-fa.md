# 使用静态属性和方法

PHP 让你无需创建一个类的实例就可以访问属性或方法。为此使用的关键字是 `static`。

## 如何做...

1.最简单的是，在声明一个普通属性或方法时，只需在声明可见性级别后添加 `static` 关键字。使用 `self`关键字在内部引用该属性：

```php
class Test
{
  public static $test = 'TEST';
  public static function getTest()
  {
    return self::$test;
  }
}
```

2. `self` 关键字会提前绑定，在访问子类的静态信息时会出现问题。如果一定需要访问子类中的信息，请使用 `static` 关键字来代替 `self` 。这个过程被称为**后期静态绑定**（**Late Static Binding**）。

3.在下面的例子中，如果运行 `Child::getEarlyTest()` ，输出将是 **TEST**。另一方面，如果运行 `Child::getLateTest()` ，输出将是 **CHILD**。原因是 PHP 在使用 `self` 时，会绑定最早的定义，而静态关键字则使用最新的绑定：

```php
class Test2
{
  public static $test = 'TEST2';
  public static function getEarlyTest()
  {
    return self::$test;
  }
  public static function getLateTest()
  {
    return static::$test;
  }
}

class Child extends Test2
{
  public static $test = 'CHILD';
}
```

4.在许多情况下，工厂设计模式与静态方法一起使用，以产生给定不同参数的对象实例。在这个例子中，定义了一个静态方法 `factory()`，它返回一个 PDO 连接：

```php
  public static function factory(
  $driver,$dbname,$host,$user,$pwd,array $options = [])
  {
    $dsn = sprintf('%s:dbname=%s;host=%s', 
    $driver, $dbname, $host);
    try {
        return new PDO($dsn, $user, $pwd, $options);
    } catch (PDOException $e) {
        error_log($e->getMessage);
    }
  }
```

## 如何运行...

你可以使用类解析运算符 **“::“** 来引用静态属性和方法。给定前面所示的 `Test` 类，如果运行这段代码：

```php
echo Test::$test;
echo PHP_EOL;
echo Test::getTest();
echo PHP_EOL;
```

你会看到这样的输出：

![](../../.gitbook/assets/image%20%2849%29.png)

为了说明后期静态绑定，基于前面显示的类 `Test2` 和 `Child` ，请尝试以下代码：

```php
echo Test2::$test;
echo Child::$test;
echo Child::getEarlyTest();
echo Child::getLateTest();
```

输出说明了 `self` 和 `static` 的区别：

![](../../.gitbook/assets/image%20%2854%29.png)

最后，要测试前面显示的 `factory()` 方法，请将代码保存到 `Application\Database` 文件夹中`Connection.php` 文件的 `Application\Database\Connection` 类中。 然后，您可以尝试以下操作：

```php
include __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
$connection = Connection::factory(
'mysql', 'php7cookbook', 'localhost', 'test', 'password');
$stmt = $connection->query('SELECT name FROM iso_country_codes');
while ($country = $stmt->fetch(PDO::FETCH_COLUMN)) 
echo $country . '';
```

您将看到从示例数据库中提取的国家/地区列表：

![](../../.gitbook/assets/image%20%2845%29.png)

## 参考...

关于后期静态绑定的更多信息，请参见PHP文档中的解释。

[http://php.net/manual/en/language.oop5.late-static-bindings.php](http://php.net/manual/en/language.oop5.late-static-bindings.php)

