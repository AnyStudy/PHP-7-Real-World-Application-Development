# 定义实体以匹配数据库表

在PHP开发人员中，一个非常常见的做法是创建代表数据库表的类。这种类通常被称为实体类，并构成了领域模型软件设计模式的核心。

## 如何做...

1.首先，我们将建立一系列实体类的一些共同特征。这些可能包括共同的属性和共同的方法。我们将把这些放到一个`Application\Entity\Base`类中。然后，所有未来的实体类都将扩展`Base`。

2.为了便于说明，我们假设所有实体都具有两个共同的属性：`$mapping`（稍后讨论）和`$id`（及其对应的`getter`和`setter`）：

```php
namespace Application\Entity;

class Base
{

  protected $id = 0;
  protected $mapping = ['id' => 'id'];

  public function getId() : int
  {
    return $this->id;
  }

  public function setId($id)
  {
    $this->id = (int) $id;
  }
}
```

3.定义一个`arrayToEntity()`方法并不是一个坏主意，它可以将一个数组转换为实体类的实例，反之亦然\(`entityToArray()`\)。这些方法实现了一个通常被称为**hydration**的过程。由于这些方法应该是通用的，所以它们最好放在`Base`类中。

4.在以下方法中，`$mapping`属性用于在数据库列名称和对象属性名称之间进行转换。 `arrayToEntity()`从数组填充此对象实例的值。 如果需要在实例之外调用此方法，可以将其定义为静态方法：

```php
public static function arrayToEntity($data, Base $instance)
{
  if ($data && is_array($data)) {
    foreach ($instance->mapping as $dbColumn => $propertyName) {
      $method = 'set' . ucfirst($propertyName);
      $instance->$method($data[$dbColumn]);
    }
    return $instance;
  }
  return FALSE;
}
```

5.`EntityToArray()`根据当前实例属性值生成一个数组：

```php
public function entityToArray()
{
  $data = array();
  foreach ($this->mapping as $dbColumn => $propertyName) {
    $method = 'get' . ucfirst($propertyName);
    $data[$dbColumn] = $this->$method() ?? NULL;
  }
  return $data;
}
```

6. 为了建立具体的实体，您需要手头有计划建模的数据库表的结构。创建映射到数据库列的属性。分配的初始值应该反映数据库列的最终数据类型。

7. 在这个例子中，我们将使用客户表。下面是MySQL数据转储中的CREATE语句，它说明了它的数据结构。

```php
CREATE TABLE 'customer' (
  'id' int(11) NOT NULL AUTO_INCREMENT,
  'name' varchar(256) CHARACTER SET latin1 COLLATE latin1_general_cs NOT NULL,
  'balance' decimal(10,2) NOT NULL,
  'email' varchar(250) NOT NULL,
  'password' char(16) NOT NULL,
  'status' int(10) unsigned NOT NULL DEFAULT '0',
  'security_question' varchar(250) DEFAULT NULL,
  'confirm_code' varchar(32) DEFAULT NULL,
  'profile_id' int(11) DEFAULT NULL,
  'level' char(3) NOT NULL,
  PRIMARY KEY ('id'),
  UNIQUE KEY 'UNIQ_81398E09E7927C74' ('email')
);
```

8. 我们现在就可以把类的属性具体化了。这也是确定相应表的好地方。在这种情况下，我们将使用一个`TABLE_NAME`类常量。

```php
namespace Application\Entity;

class Customer extends Base
{
  const TABLE_NAME = 'customer';
  protected $name = '';
  protected $balance = 0.0;
  protected $email = '';
  protected $password = '';
  protected $status = '';
  protected $securityQuestion = '';
  protected $confirmCode = '';
  protected $profileId = 0;
  protected $level = '';
}
```

9. 最好的做法是将这些属性定义为受保护的。为了访问这些属性，你需要设计公共方法来获取和设置这些属性。这里是一个很好的地方，可以利用 PHP 7 对返回值进行数据类型化的能力。

10. 在下面的代码块中，我们已经定义了`$name`和`$balance`的`getter`和`setter`。你可以想象这些方法的其余部分将如何定义。

```php
  public function getName() : string
  {
    return $this->name;
  }
  public function setName($name)
  {
    $this->name = $name;
  }
  public function getBalance() : float
  {
    return $this->balance;
  }
  public function setBalance($balance)
  {
    $this->balance = (float) $balance;
  }
}
```

{% hint style="info" %}
数据类型检查设置器上的传入值不是一个好主意。 原因是RDBMS数据库查询的返回值都将是字符串数据类型。
{% endhint %}

11. 如果属性名与对应的数据库列不完全匹配，应该考虑创建一个映射属性，即键/值对的数组，其中键代表数据库列名，值代表属性名。

12. 你会注意到三个属性，`$securityQuestion`、`$confirmCode`和`$profileId`，与它们的等价列名`security_question`、`confirm_code`和`profile_id`不对应。`$mapping` 属性将确保进行适当的翻译。

```php
protected $mapping = [
  'id'                => 'id',
  'name'              => 'name',
  'balance'           => 'balance',
  'email'             => 'email',
  'password'          => 'password',
  'status'            => 'status',
  'security_question' => 'securityQuestion',
  'confirm_code'      => 'confirmCode',
  'profile_id'        => 'profileId',
  'level'             => 'level'
];
```

## 如何运行...

将步骤2、4、5中的代码复制到 `Application/Entity` 文件夹中的`Base.php`文件中。将步骤8到12中的代码复制到`Application/Entity`文件夹中的`Customer.php`文件中。然后你需要为步骤10中没有显示的其余属性创建`getter`和`setter`：`email`,`password`, `status`,`securityQuestion`, `confirmCode`, `profileId` 和 `level`。

然后你可以创建一个`chap_05_matching_entity_to_table.php`调用程序，该程序初始化了在第一章建立基础 中定义的自动加载器，使用`Application/Database/Connection`和新创建的`Application/Entity/Customer`类。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
use Application\Entity\Customer;
```

接下来，获取一个数据库连接，利用连接随机获取一个客户的关联数组数据。

```php
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
$id = rand(1,79);
$stmt = $conn->pdo->prepare(
  'SELECT * FROM customer WHERE id = :id');
$stmt->execute(['id' => $id]);
$result = $stmt->fetch(PDO::FETCH_ASSOC);
```

最后，你可以从数组中创建一个新的`Customer`实体实例，并使用`var_dump()`来查看结果。

```php
$cust = Customer::arrayToEntity($result, new Customer());
var_dump($cust);
```

下面是前面代码的输出。

![](../../.gitbook/assets/image%20%2874%29.png)

## 参考

有许多描述领域模型的好作品。 可能最有影响力的是Martin Fowler撰写的《企业应用程序架构模式》（请参阅[http://martinfowler.com/books/eaa.html）。](http://martinfowler.com/books/eaa.html）。) 还有一个不错的研究，也可以免费下载，InfoQ的标题为“领域驱动设计”（请参阅[http://www.infoq.com/minibooks/domain-driven-design-quickly）。](http://www.infoq.com/minibooks/domain-driven-design-quickly）。)

