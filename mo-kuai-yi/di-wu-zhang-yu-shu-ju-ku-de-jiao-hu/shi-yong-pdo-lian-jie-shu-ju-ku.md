# 使用PDO连接数据库

**PDO**是一个高性能和积极维护的数据库扩展，与特定数据库的扩展相比，它具有独特的优势。它有一个通用的应用编程接口（**API**），可以兼容几乎十几种不同的关系型数据库管理系统（**RDBMS**）。学习如何使用这个扩展将节省你试图掌握同等的单个数据库扩展命令子集的时间。

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

6.使用 `PDO::query()` 发送一个SQL命令。返回一个 `PDOStatement` 实例，你可以根据它来获取结果。在这个例子中，我们正在寻找按ID排序的前20位客户。

```php
$stmt = $pdo->query(
'SELECT * FROM customer ORDER BY id LIMIT 20');
```

{% hint style="info" %}
PDO还提供了一个便利的方法 `PDO::exec()`，它不返回结果迭代，只返回受影响的行数。这个方法最好用于管理操作，比如 `ALTER TABLE`，`DROP TABLE`等。
{% endhint %}

7.

