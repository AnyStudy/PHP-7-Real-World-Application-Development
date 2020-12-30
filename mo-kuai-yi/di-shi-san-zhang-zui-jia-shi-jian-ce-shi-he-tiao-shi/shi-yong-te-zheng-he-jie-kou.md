# 使用特征和接口

利用接口作为建立一组类的分类，并保证某些方法的存在，被认为是一种最好的做法。Traits和Interface经常一起工作，是实现的一个重要方面。只要你有一个经常使用的Interface，定义了一个代码不会改变的方法（比如setter或getter），那么也定义一个包含实际代码实现的Trait是很有用的。

## 如何做...

1.在这个例子中，我们将使用`ConnectionAwareInterface`，首先在第4章，使用PHP面向对象编程中介绍。这个接口定义了一个`setConnection()`方法来设置`$connection`属性。`Application\Generic`命名空间中的两个类，`CountryList`和`CustomerList`，包含了多余的代码，它们与接口中定义的方法相匹配。

2. 以下是更改前的CountryList的样子。

```php
class CountryList
{
  protected $connection;
  protected $key   = 'iso3';
  protected $value = 'name';
  protected $table = 'iso_country_codes';
    
  public function setConnection(Connection $connection)
  {
    $this->connection = $connection;
  }
  public function list()
  {
    $list = [];
    $sql  = sprintf('SELECT %s,%s FROM %s', $this->key, 
                    $this->value, $this->table);
    $stmt = $this->connection->pdo->query($sql);
    while ($item = $stmt->fetch(PDO::FETCH_ASSOC)) {
      $list[$item[$this->key]] =  $item[$this->value];
    }
    return $list;
  }

}
```

3. 现在我们将`list()`移到一个名为`ListTrait`的特质中去。

```php
trait ListTrait
{
  public function list()
  {
    $list = [];
    $sql  = sprintf('SELECT %s,%s FROM %s', 
                    $this->key, $this->value, $this->table);
    $stmt = $this->connection->pdo->query($sql);
    while ($item = $stmt->fetch(PDO::FETCH_ASSOC)) {
           $list[$item[$this->key]] = $item[$this->value];
    }
    return $list;
  }
}
```

4. 然后我们可以将`ListTrait`中的代码插入到一个新的类--`CountryListUsingTrait`中，如下图所示。

```php
class CountryListUsingTrait
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

5. 接下来，我们观察到很多类需要设置一个连接实例。同样，这需要一个trait。然而这次，我们将trait放在`Application\Database`命名空间中。这就是新的trait。

```php
namespace Application\Database;
trait ConnectionTrait
{
  protected $connection;
  public function setConnection(Connection $connection)
  {
    $this->connection = $connection;
  }
}
```

6. 特质经常被用来避免代码的重复。通常情况下，你还需要确定使用该特征的类。一个好的方法是开发一个与trait匹配的接口。在这个例子中，我们将定义`Application\Database\ConnectionAwareInterface`。

```php
namespace Application\Database;
use Application\Database\Connection;
interface ConnectionAwareInterface
{
  public function setConnection(Connection $connection);
}
```

7. 而这里是修改后的`CountryListUsingTrait`类。请注意，由于新的trait受其在命名空间中的位置影响，我们需要在类的顶部添加一个使用声明。你还会注意到，我们实现了`ConnectionAwareInterface`来标识这个类需要trait中定义的方法的事实。请注意，我们正在利用新的 PHP 7 组使用语法。

```php
namespace Application\Generic;
use PDO;
use Application\Database\ { 
Connection, ConnectionTrait, ConnectionAwareInterface 
};
class CountryListUsingTrait implements ConnectionAwareInterface
{
  use ListTrait;
  use ConnectionTrait;
    
  protected $key   = 'iso3';
  protected $value = 'name';
  protected $table = 'iso_country_codes';
    
}
```

## 如何运行...

首先，确保在第4章 "使用PHP面向对象编程 "中开发的类已经被创建。这些类包括`Application\Generic\CountryList`和`Application\Generic\CustomerList`类，这些类在第4章 "使用接口 "中讨论过。将每个类以`CountryListUsingTrait.php`和`CustomerListUsingTrait.php`的形式保存在`Application\Generic`文件夹下的新文件中。一定要把类名改成与新文件名一致的名字!

如第3步所述，从`CountryListUsingTrait.php`和`CustomerListUsingTrait.php`中删除`list()`方法。添加`use ListTrait`;来代替删除的方法。将删除的代码放到一个单独的文件中，在同一个文件夹中，名为`ListTrait.php`。

你还会注意到两个list类之间的代码进一步重复，在这种情况下，`setConnection()`方法。这将调用另一个特质!

将`CountryListUsingTrait.php`和`CustomerListUsingTrait.php`列表类中的`setConnection()`方法剪掉，并将删除的代码放到一个单独的文件中，名为`ConnectionTrait.php`。由于这个trait在逻辑上与`ConnectionAwareInterface`和`Connection`有关，所以将该文件放在`Application\Database`文件夹下是合理的，并相应地指定其命名空间。

最后，定义`Application\Database\ConnectionAwareInterface`，如步骤6中讨论的那样。下面是所有改动后的最终`Application\Generic\CustomerListUsingTrait`类。

```php
<?php
namespace Application\Generic;
use PDO;
use Application\Database\Connection;
use Application\Database\ConnectionTrait;
use Application\Database\ConnectionAwareInterface;
class CustomerListUsingTrait implements ConnectionAwareInterface
{
   
  use ListTrait;
  use ConnectionTrait;
    
  protected $key   = 'id';
  protected $value = 'name';
  protected $table = 'customer';
}
```

现在你可以将第4章，PHP面向对象编程中提到的`chap_04_oop_simple_interfaces_example.php`文件复制到一个名为`chap_13_trait_and_interface.php`的新文件中。将`CountryList`的引用改为`CountryListUsingTrait`。同样地，将`CustomerList`的引用改为`CustomerListUsingTrait`。否则，代码可以保持不变。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
$params = include __DIR__ . DB_CONFIG_FILE;
try {
    $list = Application\Generic\ListFactory::factory(
      new Application\Generic\CountryListUsingTrait(), $params);
    echo 'Country List' . PHP_EOL;
    foreach ($list->list() as $item) echo $item . ' ';
    $list = Application\Generic\ListFactory::factory(
      new Application\Generic\CustomerListUsingTrait(), 
      $params);
    echo 'Customer List' . PHP_EOL;
    foreach ($list->list() as $item) echo $item . ' ';
    
} catch (Throwable $e) {
    echo $e->getMessage();
}
```

输出将与第4章 "使用面向对象的编程" 中的使用接口示例中的描述一模一样。你可以在下面的截图中看到输出的国家列表部分。

![](../../.gitbook/assets/image%20%28193%29.png)

下一张图片显示输出的客户列表部分。

![](../../.gitbook/assets/image%20%28188%29.png)

