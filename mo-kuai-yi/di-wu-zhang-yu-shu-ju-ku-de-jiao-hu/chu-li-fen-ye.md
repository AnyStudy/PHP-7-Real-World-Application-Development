# 处理分页

分页涉及提供数据库查询结果的有限子集。这通常是出于显示的目的，但也可以很容易地应用于其他情况。乍一看，`LimitIterator` 类似乎非常适合于分页的目的。在潜在的结果集可能是巨大的情况下；然而，`LimitIterator` 并不是一个理想的候选者，因为你需要提供整个结果集作为内部迭代器，这很可能会超过内存限制。`LimitIterator` 类构造函数的第二个和第三个参数是 `offset` 和 `count`。这表明了我们将采用的分页方案，这是SQL的原生方案：在给定的SQL语句中添加`LIMIT`和`OFFSET`子句。

## 如何做...

1.首先，我们创建一个名为 `Application\Database\Paginate` 的类来保存分页逻辑。我们添加属性来表示与分页相关的值，`$sql`、`$page`和`$linesPerPage`。

```php
namespace Application\Database;

class Paginate
{
    
  const DEFAULT_LIMIT  = 20;
  const DEFAULT_OFFSET = 0;
  
  protected $sql;
  protected $page;
  protected $linesPerPage;

}
```

2.接下来，我们定义了一个 `__construct()` 方法，该方法接受一个基础SQL语句、当前页数和每页的行数作为参数，然后我们需要重构SQL字符串，修改或添加 `LIMIT` 和 `OFFSET`子句。然后我们需要重构SQL字符串，修改或添加 `LIMIT` 和 `OFFSET` 子句。

3. 在构造函数中，我们需要使用当前页数和每页的行数来计算偏移量。我们还需要检查`LIMIT`和`OFFSET`是否已经存在于SQL语句中。最后，我们需要使用每页的行数作为我们的`LIMIT`和重新计算的`OFFSET`来修改语句。

```php
public function __construct($sql, $page, $linesPerPage)
{
  $offset = $page * $linesPerPage;
  switch (TRUE) {
    case (stripos($sql, 'LIMIT') && strpos($sql, 'OFFSET')) :
      // no action needed
      break;
    case (stripos($sql, 'LIMIT')) :
      $sql .= ' LIMIT ' . self::DEFAULT_LIMIT;
      break;
    case (stripos($sql, 'OFFSET')) :
      $sql .= ' OFFSET ' . self::DEFAULT_OFFSET;
      break;
    default :
      $sql .= ' LIMIT ' . self::DEFAULT_LIMIT;
      $sql .= ' OFFSET ' . self::DEFAULT_OFFSET;
      break;
  }
  $this->sql = preg_replace('/LIMIT \d+.*OFFSET \d+/Ui', 
     'LIMIT ' . $linesPerPage . ' OFFSET ' . $offset, 
     $sql);
}
```

4.现在我们准备使用第一个示例中讨论的 `Application\Database\Connection` 类来执行查询。

5.在我们的新分页类中，我们添加了一个 `paginate()` 方法，它需要一个 `Connection` 实例作为参数。我们还需要PDO的获取模式，以及可选的准备语句参数。

```php
use PDOException;
public function paginate(
  Connection $connection, 
  $fetchMode, 
  $params = array())
  {
  try {
    $stmt = $connection->pdo->prepare($this->sql);
    if (!$stmt) return FALSE;
    if ($params) {
      $stmt->execute($params);
    } else {
      $stmt->execute();
    }
    while ($result = $stmt->fetch($fetchMode)) yield $result;
  } catch (PDOException $e) {
    error_log($e->getMessage());
    return FALSE;
  } catch (Throwable $e) {
    error_log($e->getMessage());
    return FALSE;
  }
}
```

6.为前面的示例中提到的查询构建器类提供支持可能不是一个坏主意。 这将使更新 `LIMIT`和`OFFSET`更加容易。 提供对 `Application\Database\Finder`的支持，我们需要做的就是使用该类并修改`__construct()`方法，以检查传入的SQL是否是此类的实例。

```php
  if ($sql instanceof Finder) {
    $sql->limit($linesPerPage);
    $sql->offset($offset);
    $this->sql = $sql::getSql();
  } elseif (is_string($sql)) {
    switch (TRUE) {
      case (stripos($sql, 'LIMIT') 
      && strpos($sql, 'OFFSET')) :
          // 以上第3点所示的剩余代码
      }
   }
```

7.现在要做的就是增加一个 `getSql()` 方法，以防我们需要确认SQL语句的格式正确。

```php
public function getSql()
{
  return $this->sql;
}
```

## 如何运行...

将前面的代码复制到 `Application/Database`文件夹中的`Paginate.php`文件中，然后创建`chap_05_pagination.php`调用程序。然后你可以创建一个`chap_05_pagination.php`调用程序，该程序初始化了在第一章建立基础中定义的自动加载器。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
define('LINES_PER_PAGE', 10);
define('DEFAULT_BALANCE', 1000);
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
```

接下来，使用`Application\Database\Finder`、`Connection`和`Paginate`类，创建一个`Application\Database\Connection`的实例，并使用`Finder`生成SQL。

```php
use Application\Database\ { Finder, Connection, Paginate};
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
$sql = Finder::select('customer')->where('balance < :bal');
```

现在我们可以从 `$_GET` 参数中获取页码和余额，并创建 `Paginate`对象，结束PHP块。

```php
$page = (int) ($_GET['page'] ?? 0);
$bal  = (float) ($_GET['balance'] ?? DEFAULT_BALANCE);
$paginate = new Paginate($sql::getSql(), $page, LINES_PER_PAGE);
?>
```

在脚本的输出部分，我们只需使用一个简单的 `foreach()` 循环来迭代分页。

```php
<h3><?= $paginate->getSql(); ?></h3>	
<hr>
<pre>
<?php
printf('%4s | %20s | %5s | %7s' . PHP_EOL, 
  'ID', 'NAME', 'LEVEL', 'BALANCE');
printf('%4s | %20s | %5s | %7s' . PHP_EOL, 
  '----', str_repeat('-', 20), '-----', '-------');
foreach ($paginate->paginate($conn, PDO::FETCH_ASSOC, 
  ['bal' => $bal]) as $row) {
  printf('%4d | %20s | %5s | %7.2f' . PHP_EOL, 
      $row['id'],$row['name'],$row['level'],$row['balance']);
}
printf('%4s | %20s | %5s | %7s' . PHP_EOL, 
  '----', str_repeat('-', 20), '-----', '-------');
?>
<a href="?page=<?= $page - 1; ?>&balance=<?= $bal ?>">
<< Prev </a>&nbsp;&nbsp;
<a href="?page=<?= $page + 1; ?>&balance=<?= $bal ?>">
Next >></a>
</pre>
```

这里是输出的第3页，余额不足1000。

![](../../.gitbook/assets/image%20%2869%29.png)

## 参考

有关`LimitIterator`类的更多信息，请参考本文。

[http://php.net/manual/en/class.limititerator.php](http://php.net/manual/en/class.limititerator.php)

