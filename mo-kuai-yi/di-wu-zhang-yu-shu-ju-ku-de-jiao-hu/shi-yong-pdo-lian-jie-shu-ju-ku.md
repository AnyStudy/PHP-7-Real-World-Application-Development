# 使用PDO连接数据库

**PDO**是一个高性能和积极维护的数据库扩展，与特定数据库的扩展相比，它具有独特的优势。它有一个通用的应用编程接口（**API**），可以兼容几乎十几种不同的关系型数据库管理系统（**RDBMS**）。学习如何使用这个扩展将节省您试图掌握同等的单个数据库扩展命令子集的时间。

PDO又分为四个主要类，如下表所示：

| 类 | 功能 |
| :--- | :--- |
| `PDO` | 保持与数据库的实际连接，并处理更底层功能，如事务支持。 |
| `PDOStatement` | 处理结果 |
| `PDOException` | 数据库特有的异常 |
| `PDODriver` | 与各数据库进行通信 |

## 如何做...

1.通过创建 `PDO` 实例来设置数据库连接。

2. 您需要构建一个数据源名称（**DSN**）。DSN中包含的信息根据使用的数据库驱动程序而不同。举个例子，这里是一个用于连接到**MySQL**数据库的DSN。

```php
$params = [
  'host' => 'localhost',
  'user' => 'test',
  'pwd'  => 'password',
  'db'   => 'php7cookbook'
];

try {
  $dsn  = sprintf('mysql:host=%s;dbname=%s',
  $params['host'], $params['db']);
  $pdo  = new PDO($dsn, $params['user'], $params['pwd']);
} catch (PDOException $e) {
  echo $e->getMessage();
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

3.而 **SQlite** 一个更简单的扩展，只需要以下命令。

```php
$params = [
  'db'   => __DIR__ . '/../data/db/php7cookbook.db.sqlite'
];
$dsn  = sprintf('sqlite:' . $params['db']);
```

4. 而 `PostgreSQL` 则在DSN中直接包含了用户名和密码。

```php
$params = [
  'host' => 'localhost',
  'user' => 'test',
  'pwd'  => 'password',
  'db'   => 'php7cookbook'
];
$dsn  = sprintf('pgsql:host=%s;dbname=%s;user=%s;password=%s', 
               $params['host'], 
               $params['db'],
               $params['user'],
               $params['pwd']);
```

5.DSN还可以包含特定于服务器的指令，例如 `unix_socket`，如下例所示。

```php
$params = [
  'host' => 'localhost',
  'user' => 'test',
  'pwd'  => 'password',
  'db'   => 'php7cookbook',
  'sock' => '/var/run/mysqld/mysqld.sock'
];

try {
  $dsn  = sprintf('mysql:host=%s;dbname=%s;unix_socket=%s', 
                  $params['host'], $params['db'], $params['sock']);
  $opts = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION];
  $pdo  = new PDO($dsn, $params['user'], $params['pwd'], $opts);
} catch (PDOException $e) {
  echo $e->getMessage();
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

{% hint style="info" %}
将创建PDO实例的语句封装在 `try {} catch {}` 块中。在失败的情况下，捕获 `PDOException`来获取数据库的特定信息。抓取Throwable来处理错误或其他异常。将PDO错误模式设置为 `PDO::ERRMODE_EXCEPTION以` 获得最佳效果。关于错误模式的更多细节请参见步骤8。

在PHP 5中，如果PDO对象不能被构造（例如，当使用无效参数时），实例被分配一个`NUL`L值。在 PHP 7 中，会抛出一个 `Exception`。如果把PDO对象的构造包在`try {} catch {}`块中，并且`PDO::ATTR_ERRMODE`设置为`PDO::ERRMODE_EXCEPTION`，就可以捕捉和记录这样的错误，而不需要测试NULL。
{% endhint %}

6.使用 `PDO::query()` 发送一个SQL命令。返回一个 `PDOStatement` 实例，您可以根据它来获取结果。在这个例子中，我们正在寻找按ID排序的前20位客户。

```php
$stmt = $pdo->query(
'SELECT * FROM customer ORDER BY id LIMIT 20');
```

{% hint style="info" %}
PDO还提供了一个便利的方法 `PDO::exec()`，它不返回结果迭代，只返回受影响的行数。这个方法最好用于管理操作，比如 `ALTER TABLE`，`DROP TABLE`等。
{% endhint %}

7.遍历 `PDOStatement` 实例来处理结果。设置获取模式为 `PDO::FETCH_NUM`或 `PDO::FETCH_ASSOC`，以数字或关联数组的形式返回结果。在这个例子中，我们使用一个 `while()` 循环来处理结果。当最后一个结果被获取后，结果是一个布尔值 `FALSE`，结束循环。

```php
while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
  printf('%4d | %20s | %5s' . PHP_EOL, $row['id'], 
  $row['name'], $row['level']);
}
```

{% hint style="info" %}
PDO获取操作涉及游标，这个游标定义了迭代的方向（即正向或反向）。`PDOStatement::fetch()` 的第二个参数可以是 `PDO::FETCHORI_*` 常量中的任何一个。游标方向包括 prior、first、last、absolute 和 relative。默认的游标方向是 `PDO::FETCH_ORI_NEXT`。
{% endhint %}

8.将获取模式设置为 `PDO::FETCH_OBJ`，以 `stdClass` 实例的形式返回结果。这里您会注意到，`while()`循环利用了 `PDO::FETCH_OBJ` 的获取模式。请注意，`printf()` 语句引用的是对象属性，而前面的例子引用的是数组元素。

```php
while ($row = $stmt->fetch(PDO::FETCH_OBJ)) {
  printf('%4d | %20s | %5s' . PHP_EOL, 
  $row->id, $row->name, $row->level);
}
```

9.如果你想在处理查询时创建一个特定类的实例，请将获取模式设置为 `PDO::FETCH_CLASS`。您还必须有类的定义，`PDO::query()` 应该设置类名。在下面的代码片段中可以看到，我们定义了一个名为 `Customer`的类，具有公共属性 `$id`、`$name`和`$level`。属性需要是公共的，这样取值注入才能正常工作。

```php
class Customer
{
  public $id;
  public $name;
  public $level;
}

$stmt = $pdo->query($sql, PDO::FETCH_CLASS, 'Customer');
```

10.当获取对象时，步骤5中所示的技术的更简单的替代方法是使用 `PDOStatement::fetchObject()`。

```php
while ($row = $stmt->fetchObject('Customer')) {
  printf('%4d | %20s | %5s' . PHP_EOL, 
  $row->id, $row->name, $row->level);
}
```

11.您也可以使用 `PDO::FETCH_INTO`，它和 `PDO::FETCH_CLASS` 本质上是一样的，但是您需要一个对象实例而不是一个类引用。每一次循环的迭代都会用当前的信息集重新填充同一个对象实例。这个例子假设`Customer` 类和第5步一样，使用和第1步中定义的相同的数据库参数和PDO连接。

```php
$cust = new Customer();
while ($stmt->fetch(PDO::FETCH_INTO)) {
  printf('%4d | %20s | %5s' . PHP_EOL, 
  $cust->id, $cust->name, $cust->level);
}
```

12.如果您没有指定错误模式，默认的PDO错误模式是 `PDO::ERRMODE_SILENT`。您可以使用`PDO::ATTR_ERRMODE` 作为键，然后 `PDO::ERRMODE_WARNING`或`PDO::ERRMODE_EXCEPTION` 作为值来设置错误模式。错误模式可以以关联数组的形式指定为PDO构造函数的第四个参数，也可以使用 `PDO::ERRMODE_WARNING`或`PDO::ERRMODE_EXCEPTION`。另外，你也可以在现有的实例上使用`PDO::setAttribute()`。

13. 让我们假设您有以下DSN和SQL（在开始认为这是一种新的SQL形式之前，请放心：这个SQL语句将无法使用！）。

```php
$params = [
  'host' => 'localhost',
  'user' => 'test',
  'pwd'  => 'password',
  'db'   => 'php7cookbook'
];
$dsn  = sprintf('mysql:host=%s;dbname=%s', $params['host'], $params['db']);
$sql  = 'THIS SQL STATEMENT WILL NOT WORK';
```

14.如果你使用默认的错误模式制定你的PDO连接，唯一的线索就是 `PDO::query()` 将返回一个布尔值`FALSE`，而不是生成一个`PDOStatement`实例。

```php
$pdo1  = new PDO($dsn, $params['user'], $params['pwd']);
$stmt = $pdo1->query($sql);
$row = ($stmt) ? $stmt->fetch(PDO::FETCH_ASSOC) : 'No Good';
```

15. 接下来的例子显示了使用构造函数方法将错误模式设置为`WARNING`。

```php
$pdo2 = new PDO(
  $dsn, 
  $params['user'], 
  $params['pwd'], 
  [PDO::ATTR_ERRMODE => PDO::ERRMODE_WARNING]);
```

16.如果你需要将准备阶段和执行阶段完全分离，可以使用 `PDO::prepare()`和`PDOStatement::execute()`来代替。语句会被发送到数据库服务器进行预编译。然后你可以根据需要多次执行该语句，很可能是在一个循环中执行。

17.`PDO::prepare()`的第一个参数是一个SQL语句，用占位符代替实际值。然后可以向`PDOStatement::execute()`提供一个数组。PDO自动提供数据库引用，这有助于防止SQL注入。

{% hint style="info" %}
**最佳实践**

任何将外部输入（即来自表单发布的输入）与SQL语句相结合的应用程序都会受到SQL注入攻击。所有的外部输入必须首先进行适当的过滤、验证和其他方面的**校验**。不要将外部输入直接放入SQL语句中。相反，使用占位符，并在执行阶段提供实际（经过校验）的值。
{% endhint %}

18.要以反向方式遍历结果，可以改变可滚动光标的方向。另外，可能更容易的是，只需将 `ORDER BY`从`ASC`反转到`DESC`。这行代码设置了一个`PDOStatement`对象，请求一个可滚动的游标。

```php
$dsn  = sprintf('pgsql:charset=UTF8;host=%s;dbname=%s', $params['host'], $params['db']);
$opts = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]; 
$pdo  = new PDO($dsn, $params['user'], $params['pwd'], $opts);
$sql  = 'SELECT * FROM customer '
    . 'WHERE balance > :min AND balance < :max '
    . 'ORDER BY id LIMIT 20';
$stmt = $pdo->prepare($sql, [PDO::ATTR_CURSOR  => PDO::CURSOR_SCROLL]);
```

19. 你还需要在获取操作中指定游标指令。这个例子获取结果集中的最后一行，然后向后滚动。

```php
$stmt->execute(['min' => $min, 'max' => $max]);
$row = $stmt->fetch(PDO::FETCH_ASSOC, PDO::FETCH_ORI_LAST);
do {
  printf('%4d | %20s | %5s | %8.2f' . PHP_EOL, 
       $row['id'], 
       $row['name'], 
       $row['level'], 
       $row['balance']);
} while ($row = $stmt->fetch(PDO::FETCH_ASSOC, PDO::FETCH_ORI_PRIOR));
```

20. `MySQL`和`SQLite`都不支持可滚动的光标! 为了达到同样的效果，请尝试对前面的代码进行以下修改。

```php
$dsn  = sprintf('mysql:charset=UTF8;host=%s;dbname=%s', $params['host'], $params['db']);
$opts = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]; 
$pdo  = new PDO($dsn, $params['user'], $params['pwd'], $opts);
$sql  = 'SELECT * FROM customer '
    . 'WHERE balance > :min AND balance < :max '
    . 'ORDER BY id DESC 
       . 'LIMIT 20';
$stmt = $pdo->prepare($sql);
while ($row = $stmt->fetch(PDO::FETCH_ASSOC));
printf('%4d | %20s | %5s | %8.2f' . PHP_EOL, 
       $row['id'], 
       $row['name'], 
       $row['level'], 
       $row['balance']);
} 
```

21. PDO提供了对事务的支持。借用步骤9中的代码，我们可以将 `INSERT` 系列命令包装成一个事务块。

```php
try {
    $pdo->beginTransaction();
    $sql  = "INSERT INTO customer ('" 
    . implode("','", $fields) . "') VALUES (?,?,?,?,?,?)";
    $stmt = $pdo->prepare($sql);
    foreach ($data as $row) $stmt->execute($row);
    $pdo->commit();
} catch (PDOException $e) {
    error_log($e->getMessage());
    $pdo->rollBack();
}
```

22.最后，为了保持一切的模块化和可重用性，我们可以将PDO连接包装成一个单独的类`Application\Database\Connection`。在这里，我们通过构造函数建立一个连接。另外，还有一个静态`factory()`方法，让我们生成一系列的PDO实例。

```php
namespace Application\Database;
use Exception;
use PDO;
class Connection
{
    const ERROR_UNABLE = 'ERROR: no database connection';
    public $pdo;
    public function __construct(array $config)
    {
        if (!isset($config['driver'])) {
            $message = __METHOD__ . ' : ' 
            . self::ERROR_UNABLE . PHP_EOL;
            throw new Exception($message);
        }
        $dsn = $this->makeDsn($config);        
        try {
            $this->pdo = new PDO(
                $dsn, 
                $config['user'], 
                $config['password'], 
                [PDO::ATTR_ERRMODE => $config['errmode']]);
            return TRUE;
        } catch (PDOException $e) {
            error_log($e->getMessage());
            return FALSE;
        }
    }

    public static function factory(
      $driver, $dbname, $host, $user, 
      $pwd, array $options = array())
    {
        $dsn = $this->makeDsn($config);
        
        try {
            return new PDO($dsn, $user, $pwd, $options);
        } catch (PDOException $e) {
            error_log($e->getMessage);
        }
    }
```

23.这个 `Connection` 类的一个重要组成部分是一个可以用来构造DSN的通用方法。我们需要做的就是建立`PDODriver`作为前缀，后面是"`:`"。之后，我们只需从配置数组中附加键/值对。每个键/值对都用分号隔开。我们还需要把后面的分号去掉，为此使用带有负限制的`substr()`。

```php
  public function makeDsn($config)
  {
    $dsn = $config['driver'] . ':';
    unset($config['driver']);
    foreach ($config as $key => $value) {
      $dsn .= $key . '=' . $value . ';';
    }
    return substr($dsn, 0, -1);
  }
}
```

## 如何运行...

首先，你可以将第一步的初始连接代码复制到 `chap_05_pdo_connect_mysql.php` 文件中。在这个例子中，我们假设你已经创建了一个名为 `php7cookbook` 的MySQL数据库，用户名为cook，密码为book。接下来，我们使用 `PDO::query()` 方法向数据库发送一条简单的SQL语句。最后，我们使用产生的语句对象以关联数组的形式获取结果。不要忘记用 `try {} catch {}` 块来包装你的代码。

```php
<?php
$params = [
  'host' => 'localhost',
  'user' => 'test',
  'pwd'  => 'password',
  'db'   => 'php7cookbook'
];
try {
  $dsn  = sprintf('mysql:charset=UTF8;host=%s;dbname=%s',
    $params['host'], $params['db']);
  $pdo  = new PDO($dsn, $params['user'], $params['pwd']);
  $stmt = $pdo->query(
    'SELECT * FROM customer ORDER BY id LIMIT 20');
  printf('%4s | %20s | %5s | %7s' . PHP_EOL, 
    'ID', 'NAME', 'LEVEL', 'BALANCE');
  printf('%4s | %20s | %5s | %7s' . PHP_EOL, 
    '----', str_repeat('-', 20), '-----', '-------');
  while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    printf('%4d | %20s | %5s | %7.2f' . PHP_EOL, 
    $row['id'], $row['name'], $row['level'], $row['balance']);
  }
} catch (PDOException $e) {
  error_log($e->getMessage());
} catch (Throwable $e) {
  error_log($e->getMessage());
}
```

下面是结果输出。

![](../../.gitbook/assets/image%20%2871%29.png)

在PDO构造函数中添加选项，将错误模式设置为 `EXCEPTION`。现在修改SQL语句，观察产生的错误信息。

```php
$opts = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION];
$pdo  = new PDO($dsn, $params['user'], $params['pwd'], $opts);
$stmt = $pdo->query('THIS SQL STATEMENT WILL NOT WORK');
```

你会看到如下内容。

![](../../.gitbook/assets/image%20%2869%29.png)

占位符可以是命名的，也可以是定位的。已命名的占位符在准备好的SQL语句中以冒号\(`:`\)开头，是提供给 `execute()` 的关联数组中作为键的引用。位置占位符在准备好的SQL语句中用问号\(`?`\)表示。

在下面的例子中，命名的占位符被用来表示`WHERE`子句中的值。

```php
try {
  $dsn  = sprintf('mysql:host=%s;dbname=%s', 
                  $params['host'], $params['db']);
  $pdo  = new PDO($dsn, 
                  $params['user'], 
                  $params['pwd'], 
                  [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]);
  $sql  = 'SELECT * FROM customer '
      . 'WHERE balance < :val AND level = :level '
      . 'ORDER BY id LIMIT 20'; echo $sql . PHP_EOL;
  $stmt = $pdo->prepare($sql);
  $stmt->execute(['val' => 100, 'level' => 'BEG']);
  while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    printf('%4d | %20s | %5s | %5.2f' . PHP_EOL, 
      	$row['id'], $row['name'], $row['level'], $row['balance']);
  }
} catch (PDOException $e) {
  echo $e->getMessage();
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

这个例子展示了在 `INSERT` 操作中使用位置占位符。请注意，作为第四个客户要插入的数据包括一个潜在的SQL注入攻击。你还会注意到，需要对正在使用的数据库的SQL语法有一定的认识。在这种情况下，MySQL的列名是用背标（`'`）来引用的。

```php
$fields = ['name', 'balance', 'email', 
           'password', 'status', 'level'];
$data = [
  ['Saleen',0,'saleen@test.com', 'password',0,'BEG'],
  ['Lada',55.55,'lada@test.com',   'password',0,'INT'],
  ['Tonsoi',999.99,'tongsoi@test.com','password',1,'ADV'],
  ['SQL Injection',0.00,'bad','bad',1,
   'BEG\';DELETE FROM customer;--'],
];

try {
  $dsn  = sprintf('mysql:host=%s;dbname=%s', 
    $params['host'], $params['db']);
  $pdo  = new PDO($dsn, 
                  $params['user'], 
                  $params['pwd'], 
                  [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]);
  $sql  = "INSERT INTO customer ('" 
   . implode("','", $fields) 
   . "') VALUES (?,?,?,?,?,?)";
  $stmt = $pdo->prepare($sql);
  foreach ($data as $row) $stmt->execute($row);
} catch (PDOException $e) {
  echo $e->getMessage();
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

要测试使用带有命名参数的预备语句，请修改SQL语句，添加一个 `WHERE` 子句，检查余额小于一定金额的客户，以及级别等于 `BEG`、`INT`或`ADV`（即初级、中级或高级）。不使用`PDO::query()`，而使用`PDO::prepare()`。在获取结果之前，你必须执行`PDOStatement::execute()`，提供余额和级别的值。

```php
$sql  = 'SELECT * FROM customer '
     . 'WHERE balance < :val AND level = :level '
     . 'ORDER BY id LIMIT 20';
$stmt = $pdo->prepare($sql);
$stmt->execute(['val' => 100, 'level' => 'BEG']);
```

下面是结果输出。

![](../../.gitbook/assets/image%20%2870%29.png)

在调用 `PDOStatement::execute()`时，你可以不提供参数，而是绑定参数。这允许您将变量分配给占位符。在执行时，使用变量的当前值。

在这个例子中，我们将变量`$min`、`$max`和`$level`绑定到准备好的语句中。

```php
$min   = 0;
$max   = 0;
$level = '';

try {
  $dsn  = sprintf('mysql:host=%s;dbname=%s', $params['host'], $params['db']);
  $opts = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION];
  $pdo  = new PDO($dsn, $params['user'], $params['pwd'], $opts);
  $sql  = 'SELECT * FROM customer '
      . 'WHERE balance > :min '
      . 'AND balance < :max AND level = :level '
      . 'ORDER BY id LIMIT 20';
  $stmt = $pdo->prepare($sql);
  $stmt->bindParam('min',   $min);
  $stmt->bindParam('max',   $max);
  $stmt->bindParam('level', $level);
  
  $min   =  5000;
  $max   = 10000;
  $level = 'ADV';
  $stmt->execute();
  showResults($stmt, $min, $max, $level);
  
  $min   = 0;
  $max   = 100;
  $level = 'BEG';
  $stmt->execute();
  showResults($stmt, $min, $max, $level);
  
} catch (PDOException $e) {
  echo $e->getMessage();
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

当这些变量的值发生变化时，下一次执行将反映修改后的标准。

{% hint style="info" %}
**最佳实践**

使用`PDO::query()`来执行一次性的数据库命令。当你需要多次处理同一条语句但使用不同的值时，使用`PDO::prepare()`和`PDOStatement::execute()`。
{% endhint %}

## 参考

关于不同数据库特定PDO驱动的语法和独特行为的信息，请看这篇文章。

[http://php.net/manual/en/pdo.drivers.php](http://php.net/manual/en/pdo.drivers.php) 

关于PDO预定义常量的总结，包括获取模式、光标方向和属性，请看下面的文章。

[http://php.net/manual/en/pdo.constants.php](http://php.net/manual/en/pdo.constants.php)

