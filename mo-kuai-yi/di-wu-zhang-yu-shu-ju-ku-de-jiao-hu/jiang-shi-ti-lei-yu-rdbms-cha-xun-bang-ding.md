# 将实体类与RDBMS查询绑定

大多数商业上可行的RDBMS系统都是在程序化编程处于领先地位的时候发展起来的。想象一下，RDBMS世界是二维的、方形的、面向程序的。相反，实体可以被认为是圆形的、三维的、面向对象的。这让你了解到我们想通过将RDBMS查询的结果绑定到实体实例的迭代中来实现什么。

{% hint style="info" %}
现代RDBMS系统所基于的关系模型是由数学家Edgar F. Codd在1969年首次描述的。第一套商业化的系统是在70年代中后期发展起来的。所以，换句话说，RDBMS技术已经有40多年的历史了!
{% endhint %}

## 如何做...

1.首先，我们需要设计一个类来存放我们的查询逻辑。如果你遵循的是领域模型，这个类可能被称为仓库。另外，为了保持简单和通用，我们可以简单地调用新类`Application\Database\CustomerService`。该类将接受一个`Application\Database\Connection`实例作为参数。

```php
namespace Application\Database;

use Application\Entity\Customer;

class CustomerService
{
    
    protected $connection;
    
    public function __construct(Connection $connection)
    {
      $this->connection = $connection;
    }

}
```

2.现在，我们将定义一个`fetchById()`方法，它以客户ID作为参数，并返回一个单一的`Application\Entity\Customer`实例或失败时返回boolean `FALSE`。乍一看，简单地使用`PDOStatement::fetchObject()`并指定实体类作为参数似乎是不费吹灰之力。

```php
public function fetchById($id)
{
  $stmt = $this->connection->pdo
               ->prepare(Finder::select('customer')
               ->where('id = :id')::getSql());
  $stmt->execute(['id' => (int) $id]);
  return $stmt->fetchObject('Application\Entity\Customer');
}
```

{% hint style="info" %}
然而，这里的危险是，`fetchObject()`实际上在调用构造函数之前就已经填充了属性（即使它们是受保护的）！相应地，构造函数有可能意外地覆盖值。相应地，构造函数有可能意外地覆盖值。如果你没有定义一个构造函数，或者你可以忍受这种危险，我们就可以了。否则，要正确地实现RDBMS查询和OOP结果之间的联系就开始变得艰难了。
{% endhint %}

3.`fetchById()`方法的另一种方法是先创建对象实例，从而运行其构造函数，并将获取模式设置为`PDO::FETCH_INTO`，如下例所示。

```php
public function fetchById($id)
{
  $stmt = $this->connection->pdo
               ->prepare(Finder::select('customer')
               ->where('id = :id')::getSql());
  $stmt->execute(['id' => (int) $id]);
  $stmt->setFetchMode(PDO::FETCH_INTO, new Customer());
  return $stmt->fetch();
}
```

4.然而，在这里我们又遇到了一个问题：`fetch()`与`fetchObject()`不同，不能覆盖受保护的属性；如果尝试的话，会产生以下错误信息。这意味着我们要么将所有属性定义为`public`，要么考虑另一种方法。

![](../../.gitbook/assets/image%20%2871%29.png)

5. 我们将考虑的最后一种方法是以数组的形式获取结果，并手动给实体注入。尽管这种方法在性能上成本略高，但它允许任何潜在的实体构造函数正常运行，并将属性安全地定义为私有或保护。

```php
public function fetchById($id)
{
  $stmt = $this->connection->pdo
               ->prepare(Finder::select('customer')
               ->where('id = :id')::getSql());
  $stmt->execute(['id' => (int) $id]);
  return Customer::arrayToEntity(
    $stmt->fetch(PDO::FETCH_ASSOC));
}
```

6. 为了处理一个产生多个结果的查询，我们需要做的就是产生一个填充实体对象的迭代。在这个例子中，我们实现了一个`fetchByLevel()`方法，它以`Application\Entity\Customer`实例的形式，返回给定级别的所有客户。

```php
public function fetchByLevel($level)
{
  $stmt = $this->connection->pdo->prepare(
            Finder::select('customer')
            ->where('level = :level')::getSql());
  $stmt->execute(['level' => $level]);
  while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    yield Customer::arrayToEntity($row, new Customer());
  }
}
```

7. 我们希望实现的下一个方法是`save()`。然而，在我们继续之前，必须考虑到如果发生`INSERT`，将返回什么值。

8. 通常情况下，我们会在INSERT之后返回新完成的实体类，但有一个方便的`PDO::lastInsertId()`方法。有一个方便的`PDO::lastInsertId()`方法，乍一看，似乎可以做到这一点。然而，进一步阅读文档后发现，并不是所有的数据库扩展都支持这个功能，而且支持的扩展在实现上也不一致。因此，除了`$id`之外，最好是有一个唯一的列，可以用来唯一地识别新客户。

9. 在这个例子中，我们选择了电子邮件列，因此需要实现一个`fetchByEmail()`服务方法。

```php
public function fetchByEmail($email)
{
  $stmt = $this->connection->pdo->prepare(
    Finder::select('customer')
    ->where('email = :email')::getSql());
  $stmt->execute(['email' => $email]);
  return Customer::arrayToEntity(
    $stmt->fetch(PDO::FETCH_ASSOC), new Customer());
}
```

10.现在我们准备定义`save()`方法。我们将不区分`INSERT`和`UPDATE`，而是在`ID`已经存在的情况下，将该方法架构为更新，否则就进行插入。

11.首先，我们定义了一个基本的`save()`方法，它接受一个`Customer`实体作为参数，并使用`fetchById()`来确定这个条目是否已经存在。如果存在，我们调用`doUpdate()`更新方法；否则，我们调用`doInsert()`插入方法。

```php
public function save(Customer $cust)
{
  // 检查客户ID> 0是否存在
  if ($cust->getId() && $this->fetchById($cust->getId())) {
    return $this->doUpdate($cust);
  } else {
    return $this->doInsert($cust);
  }
}
```

12.接下来，我们定义`doUpdate()`，它将`Customer`实体对象的属性拉到一个数组中，建立一个初始SQL语句，并调用`flush()`方法，将数据推送到数据库中。我们不希望ID字段被更新，因为它是主键。同时我们还需要指定更新哪条记录，也就是附加一个`WHERE`子句。

```php
protected function doUpdate($cust)
{
  // 以数组形式获取属性
  $values = $cust->entityToArray();
  // 建立SQL语句
  $update = 'UPDATE ' . $cust::TABLE_NAME;
  $where = ' WHERE id = ' . $cust->getId();
  // 未设置ID，因为我们不想更新它
  unset($values['id']);
  return $this->flush($update, $values, $where);
}
```

13.`doInsert()`方法也是类似的，只是初始SQL需要以`INSERT INTO...`开头，并且`id`数组元素需要取消设置。后者的原因是，我们希望这个属性是由数据库自动生成的。如果成功的话，我们使用我们新定义的`fetchByEmail()`方法来查找新客户，并返回一个完成的实例。

```php
protected function doInsert($cust)
{
  $values = $cust->entityToArray();
  $email  = $cust->getEmail();
  unset($values['id']);
  $insert = 'INSERT INTO ' . $cust::TABLE_NAME . ' ';
  if ($this->flush($insert, $values)) {
    return $this->fetchByEmail($email);
  } else {
    return FALSE;
  }
}
```

14.最后，我们可以定义`flush()`，它完成实际的准备和执行。

```php
protected function flush($sql, $values, $where = '')
{
  $sql .=  ' SET ';
  foreach ($values as $column => $value) {
    $sql .= $column . ' = :' . $column . ',';
  }
  // 去掉尾部的','
  $sql     = substr($sql, 0, -1) . $where;
  $success = FALSE;
  try {
    $stmt = $this->connection->pdo->prepare($sql);
    $stmt->execute($values);
    $success = TRUE;
  } catch (PDOException $e) {
    error_log(__METHOD__ . ':' . __LINE__ . ':' 
    . $e->getMessage());
    $success = FALSE;
  } catch (Throwable $e) {
    error_log(__METHOD__ . ':' . __LINE__ . ':' 
    . $e->getMessage());
    $success = FALSE;
  }
  return $success;
}
```

15.为了结束讨论，我们需要定义一个`remove()`方法，从数据库中删除一个客户。同样，与之前定义的`save()`方法一样，我们使用`fetchById()`来确保操作成功。

## 如何运行...

将步骤1至步骤5中描述的代码复制到`Application/Database`文件夹中的`CustomerService.php`文件中，并在其中定义一个`chap_05_entity_to_query.php`调用程序。让调用程序使用相应的类来初始化自动加载器。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
use Application\Database\CustomerService;
```

现在你可以创建一个服务的实例，并随机获取一个客户。然后，该服务将返回一个客户实体作为结果。

```php
// 获取服务实例
$service = new CustomerService(new Connection(include __DIR__ . DB_CONFIG_FILE));

echo "\nSingle Result\n";
var_dump($service->fetchById(rand(1,79)));
```

这是输出。

![](../../.gitbook/assets/image%20%2876%29.png)

现在将步骤6至15中的代码复制到服务类中。将要插入的数据添加到`chap_05_entity_to_query.php`调用程序中。然后我们使用这些数据生成一个`Customer`实体实例。

```php
// 样本数据
$data = [
  'name'              => 'Doug Bierer',
  'balance'           => 326.33,
  'email'             => 'doug' . rand(0,999) . '@test.com',
  'password'          => 'password',
  'status'            => 1,
  'security_question' => 'Who\'s on first?',
  'confirm_code'      => 12345,
  'level'             => 'ADV'
];

// 建立新 Customer
$cust = Customer::arrayToEntity($data, new Customer());
```

然后我们可以检查调用`save()`前后的`ID`。

```php
echo "\nCustomer ID BEFORE Insert: {$cust->getId()}\n";
$cust = $service->save($cust);
echo "Customer ID AFTER Insert: {$cust->getId()}\n";
```

最后，我们修改余额，再次调用`save()`，查看结果。

```php
echo "Customer Balance BEFORE Update: {$cust->getBalance()}\n";
$cust->setBalance(999.99);
$service->save($cust);
echo "Customer Balance AFTER Update: {$cust->getBalance()}\n";
var_dump($cust);
```

下面是调用程序的输出。

![](../../.gitbook/assets/image%20%2873%29.png)

## 更多...

有关关系模型的更多信息，请参考[https://en.wikipedia.org/wiki/Relational\_model](https://en.wikipedia.org/wiki/Relational_model)。关于RDBMS的更多信息，请参考[https://en.wikipedia.org/wiki/Relational\_database\_management\_system](https://en.wikipedia.org/wiki/Relational_database_management_system)。关于`PDOStatement::fetchObject()`如何在构造函数之前插入属性值的信息，请看 "rasmus at mindplay dot dk "在php.net文档参考中关于`fetchObject()`的评论\([http://php.net/manual/en/pdostatement.fetchobject.php](http://php.net/manual/en/pdostatement.fetchobject.php)\)。

