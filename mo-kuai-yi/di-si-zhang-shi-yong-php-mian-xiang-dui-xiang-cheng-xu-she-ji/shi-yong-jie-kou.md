# 使用接口

接口是系统架构师的实用工具，通常用于应用编程接口（API）的原型设计。接口不包含实际的代码，但可以包含方法的名称以及方法的签名。

{% hint style="warning" %}
接口中标识的所有方法的可见性级别均为 `public`。
{% endhint %}

## 如何做...

1.接口标识的方法不能包含实际的代码实现。 但是，您可以指定方法参数的数据类型。

2.在这个例子中，`ConnectionAwareInterface` 标识了一个方法 `setConnection()` ，它需要一个 `Connection` 的实例作为参数：

```php
interface ConnectionAwareInterface
{
  public function setConnection(Connection $connection);
}
```

3.要使用该接口，请在定义该类的空白行之后添加关键字 `Implements`。 我们定义了两个类，`CountryList` 和 `CustomerList` ，这两个类都需要通过 `setConnection()` 方法访问 `Connection` 类。 为了识别这种依赖性，两个类都实现 `ConnectionAwareInterface` ：

```php
class CountryList implements ConnectionAwareInterface
{
  protected $connection;
  public function setConnection(Connection $connection)
  {
    $this->connection = $connection;
  }
  public function list()
  {
    $list = [];
    $stmt = $this->connection->pdo->query(
      'SELECT iso3, name FROM iso_country_codes');
    while ($country = $stmt->fetch(PDO::FETCH_ASSOC)) {
      $list[$country['iso3']] =  $country['name'];
    }
    return $list;
  }

}
class CustomerList implements ConnectionAwareInterface
{
  protected $connection;
  public function setConnection(Connection $connection)
  {
    $this->connection = $connection;
  }
  public function list()
  {
    $list = [];
    $stmt = $this->connection->pdo->query(
      'SELECT id, name FROM customer');
    while ($customer = $stmt->fetch(PDO::FETCH_ASSOC)) {
      $list[$customer['id']] =  $customer['name'];
    }
    return $list;
  }

}
```

4.接口可用于满足类型提示。 下面的类 `ListFactory` 包含一个 `factory()` 方法，该方法将初始化任何实现 `ConnectionAwareInterface` 的类。 该接口可确保定义了 `setConnection()` 方法。 将类型提示设置为接口，而不是特定的类实例，使工厂方法更通用：

```php
namespace Application\Generic;

use PDO;
use Exception;
use Application\Database\Connection;
use Application\Database\ConnectionAwareInterface;

class ListFactory
{
  const ERROR_AWARE = 'Class must be Connection Aware';
  public static function factory(
    ConnectionAwareInterface $class, $dbParams)
  {
    if ($class instanceof ConnectionAwareInterface) {
        $class->setConnection(new Connection($dbParams));
        return $class;
    } else {
        throw new Exception(self::ERROR_AWARE);
    }
    return FALSE;
  }
}
```

5.如果一个类实现了多个接口，如果方法签名不匹配，就会发生命名冲突。在这个例子中，有两个接口，`DateAware` 和 `TimeAware`。除了定义 `setDate()` 和 `setTime()` 方法外，它们都定义了 `setBoth()`。拥有重复的方法名并不是问题，尽管它不被认为是最佳实践。问题在于方法的签名不同：

```php
interface DateAware
{
  public function setDate($date);
  public function setBoth(DateTime $dateTime);
}

interface TimeAware
{
  public function setTime($time);
  public function setBoth($date, $time);
}

class DateTimeHandler implements DateAware, TimeAware
{
  protected $date;
  protected $time;
  public function setDate($date)
  {
    $this->date = $date;
  }
  public function setTime($time)
  {
    $this->time = $time;
  }
  public function setBoth(DateTime $dateTime)
  {
    $this->date = $date;
  }
}
```

6.由于该代码块的存在，将产生一个致命的错误（不能被捕获！）。为了解决这个问题，首选的方法是从任意一个接口中删除setBoth\(\)的定义。或者，你也可以调整方法签名来匹配。

{% hint style="warning" %}
**最佳实践**

不要定义有重复或重叠方法定义的接口。
{% endhint %}

## **如何运行...**

在 `Application/Database` 文件夹中，创建一个文件，`ConnectionAwareInterface.php`。插入前面第2步中讨论的代码。

接下来，在 `Application/Generic`文件夹中，创建两个文件，`CountryList.php` 和 `CustomerList.php`。插入步骤3中讨论的代码。

接下来，在与 `Application` 目录平行的目录下，创建一个源代码文件，`chap_04_oop_simple_interfaces_example.php`，该文件初始化自动加载器并包含数据库参数：

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
$params = include __DIR__ . DB_CONFIG_FILE;
```

在这个例子中，数据库参数被假定在 `DB_CONFIG_FILE` 常量所指示的数据库配置文件中。

现在您可以使用 `ListFactory::factory()` 来生成 `CountryList` 和 `CustomerList` 对象。请注意，如果这些类没有实现 `ConnectionAwareInterface`，就会抛出一个错误：

```php
  $list = Application\Generic\ListFactory::factory(
    new Application\Generic\CountryList(), $params);
  foreach ($list->list() as $item) echo $item . '';
```

以下是国家列表的输出：

![](../../.gitbook/assets/image%20%2860%29.png)

你也可以使用工厂方法来生成一个 `CustomerList` 对象并使用它：

```php
 $list = Application\Generic\ListFactory::factory(
    new Application\Generic\CustomerList(), $params);
  foreach ($list->list() as $item) echo $item . '';
```

下面是 `CustomerList` 的输出：

如果您想研究当实现了多个接口，但方法签名不同时，会发生什么情况，将前面第4步所示的代码输入一个文件，`chap_04_oop_interfaces_collisions.php`。当您尝试运行这个文件时，会产生一个错误，如图所示：

![](../../.gitbook/assets/image%20%2859%29.png)

如果您在 `TimeAware` 接口中进行以下调整，就不会产生错误：

```php
interface TimeAware
{
  public function setTime($time);
  // 这会引起麻烦
  public function setBoth(DateTime $dateTime);
}
```



