# 使用高速缓存提高性能

缓存软件设计模式是你存储一个需要长时间才能生成的结果。这可能是一个冗长的视图脚本或复杂的数据库查询的形式。当然，如果你希望改善网站访问者的用户体验，那么存储目标需要具有很高的性能。由于不同的安装会有不同的潜在存储目标，缓存机制也适合于适配器模式。潜在的存储目标的例子包括内存、数据库和文件系统。

## 如何做...

1.与本章的其他几个示例一样，由于有共享的常量，我们定义了一个谨慎的`Application\Cache\Constants`类。

```php
<?php
namespace Application\Cache;

class Constants
{
  const DEFAULT_GROUP  = 'default';
  const DEFAULT_PREFIX = 'CACHE_';
  const DEFAULT_SUFFIX = '.cache';
  const ERROR_GET      = 'ERROR: unable to retrieve from cache';
  // not all constants are shown to conserve space
}
```

2. 由于我们遵循的是适配器设计模式，所以接下来我们定义一个接口。

```php
namespace Application\Cache;
interface  CacheAdapterInterface
{
  public function hasKey($key);
  public function getFromCache($key, $group);
  public function saveToCache($key, $data, $group);
  public function removeByKey($key);
  public function removeByGroup($group);
}
```

3. 现在我们准备好定义我们的第一个缓存适配器，在这个例子中，通过使用MySQL数据库。我们需要定义一些属性，这些属性将保存列名以及准备好的语句。

```php
namespace Application\Cache;
use PDO;
use Application\Database\Connection;
class Database implements CacheAdapterInterface
{
  protected $sql;
  protected $connection;
  protected $table;
  protected $dataColumnName;
  protected $keyColumnName;
  protected $groupColumnName;
  protected $statementHasKey       = NULL;
  protected $statementGetFromCache = NULL;
  protected $statementSaveToCache  = NULL;
  protected $statementRemoveByKey  = NULL;
  protected $statementRemoveByGroup= NULL;
```

4. 构造函数允许我们提供关键列名以及 `Application\Database\Connection` 实例和用于缓存的表的名称。

```php
public function __construct(Connection $connection,
  $table,
  $idColumnName,
  $keyColumnName,
  $dataColumnName,
  $groupColumnName = Constants::DEFAULT_GROUP)
  {
    $this->connection  = $connection;
    $this->setTable($table);
    $this->setIdColumnName($idColumnName);
    $this->setDataColumnName($dataColumnName);
    $this->setKeyColumnName($keyColumnName);
    $this->setGroupColumnName($groupColumnName);
  }
```

5. 接下来的几个方法是准备语句，并在我们访问数据库时被调用。我们没有展示所有的方法，但足够给你一个概念。

```php
public function prepareHasKey()
{
  $sql = 'SELECT `' . $this->idColumnName . '` '
  . 'FROM `'   . $this->table . '` '
  . 'WHERE `'  . $this->keyColumnName . '` = :key ';
  $this->sql[__METHOD__] = $sql;
  $this->statementHasKey = 
  $this->connection->pdo->prepare($sql);
}
public function prepareGetFromCache()
{
  $sql = 'SELECT `' . $this->dataColumnName . '` '
  . 'FROM `'   . $this->table . '` '
  . 'WHERE `'  . $this->keyColumnName . '` = :key '
  . 'AND `'    . $this->groupColumnName . '` = :group';
  $this->sql[__METHOD__] = $sql;
  $this->statementGetFromCache = 
  $this->connection->pdo->prepare($sql);
}
```

6. 现在，我们定义一个方法来确定给定键的数据是否存在。

```php
public function hasKey($key)
{
  $result = 0;
  try {
      if (!$this->statementHasKey) $this->prepareHasKey();
          $this->statementHasKey->execute(['key' => $key]);
  } catch (Throwable $e) {
      error_log(__METHOD__ . ':' . $e->getMessage());
      throw new Exception(Constants::ERROR_REMOVE_KEY);
  }
  return (int) $this->statementHasKey
  ->fetch(PDO::FETCH_ASSOC)[$this->idColumnName];
}
```

7. 核心方法是从缓存中读取和写入缓存的方法。这里是从缓存中检索的方法。我们需要做的就是执行准备好的语句，执行一个`SELECT`，其中有一个`WHERE`子句，其中包含了key和group。

```php
public function getFromCache(
$key, $group = Constants::DEFAULT_GROUP)
{
  try {
      if (!$this->statementGetFromCache) 
          $this->prepareGetFromCache();
          $this->statementGetFromCache->execute(
            ['key' => $key, 'group' => $group]);
          while ($row = $this->statementGetFromCache
            ->fetch(PDO::FETCH_ASSOC)) {
            if ($row && count($row)) {
                yield unserialize($row[$this->dataColumnName]);
            }
          }
  } catch (Throwable $e) {
      error_log(__METHOD__ . ':' . $e->getMessage());
      throw new Exception(Constants::ERROR_GET);
  }
}
```

8. 当向缓存写入时，我们首先确定这个缓存键的条目是否存在。如果存在，我们执行`UPDATE`；否则，我们执行`INSERT`。

```php
public function saveToCache($key, $data, $group = Constants::DEFAULT_GROUP)
{
  $id = $this->hasKey($key);
  $result = 0;
  try {
      if ($id) {
          if (!$this->statementUpdateCache) 
              $this->prepareUpdateCache();
              $result = $this->statementUpdateCache
              ->execute(['key' => $key, 
              'data' => serialize($data), 
              'group' => $group, 
              'id' => $id]);
          } else {
              if (!$this->statementSaveToCache) 
              $this->prepareSaveToCache();
              $result = $this->statementSaveToCache
              ->execute(['key' => $key, 
              'data' => serialize($data), 
              'group' => $group]);
          }
      } catch (Throwable $e) {
          error_log(__METHOD__ . ':' . $e->getMessage());
          throw new Exception(Constants::ERROR_SAVE);
      }
      return $result;
   }
```

9. 然后，我们定义了两种方法，可以按键或按组删除缓存。如果有大量的项目需要删除，按组删除是一个方便机制。

```php
public function removeByKey($key)
{
  $result = 0;
  try {
      if (!$this->statementRemoveByKey) 
      $this->prepareRemoveByKey();
      $result = $this->statementRemoveByKey->execute(
        ['key' => $key]);
  } catch (Throwable $e) {
      error_log(__METHOD__ . ':' . $e->getMessage());
      throw new Exception(Constants::ERROR_REMOVE_KEY);
  }
  return $result;
}

public function removeByGroup($group)
{
  $result = 0;
  try {
      if (!$this->statementRemoveByGroup) 
          $this->prepareRemoveByGroup();
          $result = $this->statementRemoveByGroup->execute(
            ['group' => $group]);
      } catch (Throwable $e) {
          error_log(__METHOD__ . ':' . $e->getMessage());
          throw new Exception(Constants::ERROR_REMOVE_GROUP);
      }
      return $result;
  }
```

10. 最后，我们为每个属性定义getter和setter。为了节省篇幅，这里没有全部显示。

```php
public function setTable($name)
{
  $this->table = $name;
}
public function getTable()
{
  return $this->table;
}
// etc.
}
```

11. 文件系统缓存适配器定义了与前面定义的相同的方法。请注意`md5()`的使用，不是为了安全，而是作为一种从密钥中快速生成文本字符串的方法。

```php
namespace Application\Cache;
use RecursiveIteratorIterator;
use RecursiveDirectoryIterator;
class File implements CacheAdapterInterface
{
  protected $dir;
  protected $prefix;
  protected $suffix;
  public function __construct(
    $dir, $prefix = NULL, $suffix = NULL)
  {
    if (!file_exists($dir)) {
        error_log(__METHOD__ . ':' . Constants::ERROR_DIR_NOT);
        throw new Exception(Constants::ERROR_DIR_NOT);
    }
    $this->dir = $dir;
    $this->prefix = $prefix ?? Constants::DEFAULT_PREFIX;
    $this->suffix = $suffix ?? Constants::DEFAULT_SUFFIX;
  }

  public function hasKey($key)
  {
    $action = function ($name, $md5Key, &$item) {
      if (strpos($name, $md5Key) !== FALSE) {
        $item ++;
      }
    };

    return $this->findKey($key, $action);
  }

  public function getFromCache($key, $group = Constants::DEFAULT_GROUP)
  {
    $fn = $this->dir . '/' . $group . '/' 
    . $this->prefix . md5($key) . $this->suffix;
    if (file_exists($fn)) {
        foreach (file($fn) as $line) { yield $line; }
    } else {
        return array();
    }
  }

  public function saveToCache(
    $key, $data, $group = Constants::DEFAULT_GROUP)
  {
    $baseDir = $this->dir . '/' . $group;
    if (!file_exists($baseDir)) mkdir($baseDir);
    $fn = $baseDir . '/' . $this->prefix . md5($key) 
    . $this->suffix;
    return file_put_contents($fn, json_encode($data));
  }

  protected function findKey($key, callable $action)
  {
    $md5Key = md5($key);
    $iterator = new RecursiveIteratorIterator(
      new RecursiveDirectoryIterator($this->dir),
      RecursiveIteratorIterator::SELF_FIRST);
      $item = 0;
    foreach ($iterator as $name => $obj) {
      $action($name, $md5Key, $item);
    }
    return $item;
  }

  public function removeByKey($key)
  {
    $action = function ($name, $md5Key, &$item) {
      if (strpos($name, $md5Key) !== FALSE) {
        unlink($name);
        $item++;
      }
    };
    return $this->findKey($key, $action);
  }

  public function removeByGroup($group)
  {
    $removed = 0;
    $baseDir = $this->dir . '/' . $group;
    $pattern = $baseDir . '/' . $this->prefix . '*' 
    . $this->suffix;
    foreach (glob($pattern) as $file) {
      unlink($file);
      $removed++;
    }
    return $removed;
  }
}
```

12. 现在我们准备介绍核心的缓存机制。在构造函数中，我们接受一个实现`CacheAdapterInterface`的类作为参数。

```php
namespace Application\Cache;
use Psr\Http\Message\RequestInterface;
use Application\MiddleWare\ { Request, Response, TextStream };
class Core
{
  public function __construct(CacheAdapterInterface $adapter)
  {
    $this->adapter = $adapter;
  }
```

13. 接下来是一系列的包装方法，它们从适配器中调用相同名称的方法，但接受一个`Psr\Http\Message\RequestInterface`类作为参数，并返回一个`Psr\Http\Message\ResponseInterface`作为响应。我们从一个简单的：`hasKey()`开始。请注意我们如何从请求参数中提取密钥。

```php
public function hasKey(RequestInterface $request)
{
  $key = $request->getUri()->getQueryParams()['key'] ?? '';
  $result = $this->adapter->hasKey($key);
}
```

14. 为了从缓存中检索信息，我们需要从请求对象中提取键和组参数，然后从适配器中调用同样的方法。如果没有得到结果，我们设置一个`204`代码，表示请求成功，但没有产生内容。否则，我们设置一个`200`（成功）代码，并对结果进行迭代。然后，所有的内容都会被塞进一个响应对象，并被返回。

```php
public function getFromCache(RequestInterface $request)
{
  $text = array();
  $key = $request->getUri()->getQueryParams()['key'] ?? '';
  $group = $request->getUri()->getQueryParams()['group'] 
    ?? Constants::DEFAULT_GROUP;
  $results = $this->adapter->getFromCache($key, $group);
  if (!$results) { 
      $code = 204; 
  } else {
      $code = 200;
      foreach ($results as $line) $text[] = $line;
  }
  if (!$text || count($text) == 0) $code = 204;
  $body = new TextStream(json_encode($text));
  return (new Response())->withStatus($code)
                         ->withBody($body);
}
```

15. 奇怪的是，向缓存写入的结果几乎是一样的，只是希望结果是一个数字（即受影响的行数），或者是一个布尔结果。

```php
public function saveToCache(RequestInterface $request)
{
  $text = array();
  $key = $request->getUri()->getQueryParams()['key'] ?? '';
  $group = $request->getUri()->getQueryParams()['group'] 
    ?? Constants::DEFAULT_GROUP;
  $data = $request->getBody()->getContents();
  $results = $this->adapter->saveToCache($key, $data, $group);
  if (!$results) { 
      $code = 204;
  } else {
      $code = 200;
      $text[] = $results;
  }
      $body = new TextStream(json_encode($text));
      return (new Response())->withStatus($code)
                             ->withBody($body);
  }
```

16. 正如预期的那样，移除的方法非常相似。

```php
public function removeByKey(RequestInterface $request)
{
  $text = array();
  $key = $request->getUri()->getQueryParams()['key'] ?? '';
  $results = $this->adapter->removeByKey($key);
  if (!$results) {
      $code = 204;
  } else {
      $code = 200;
      $text[] = $results;
  }
  $body = new TextStream(json_encode($text));
  return (new Response())->withStatus($code)
                         ->withBody($body);
}

public function removeByGroup(RequestInterface $request)
{
  $text = array();
  $group = $request->getUri()->getQueryParams()['group'] 
    ?? Constants::DEFAULT_GROUP;
  $results = $this->adapter->removeByGroup($group);
  if (!$results) {
      $code = 204;
  } else {
      $code = 200;
      $text[] = $results;
  }
  $body = new TextStream(json_encode($text));
  return (new Response())->withStatus($code)
                         ->withBody($body);
  }
} // closing brace for class Core
```

## 如何运行...

为了演示`Acl`类的使用，你需要定义本示例中所描述的类，在这里总结一下。

| Class | 在这些步骤中讨论 |
| :--- | :--- |
| `Application\Cache\Constants` | 1 |
| `Application\Cache\CacheAdapterInterface` | 2 |
| `Application\Cache\Database` | 3 - 10 |
| `Application\Cache\File` | 11 |
| `Application\Cache\Core` | 12 - 16 |

接下来，定义一个测试程序，你可以把它叫做 `chap_09_middleware_cache_db.php`。在这个程序中，像往常一样，为必要的文件定义常量，设置自动加载，使用适当的类，哦......并写一个产生质数的函数（你可能在这一点上重新阅读最后一点。不用担心，我们可以帮助你！

```php
<?php
define('DB_CONFIG_FILE', __DIR__ . '/../config/db.config.php');
define('DB_TABLE', 'cache');
define('CACHE_DIR', __DIR__ . '/cache');
define('MAX_NUM', 100000);
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
use Application\Cache\{ Constants, Core, Database, File };
use Application\MiddleWare\ { Request, TextStream };
```

好吧，需要一个运行时间长的函数，那么质数生成器，我们来了! 数字1、2和3是质数。我们使用 PHP 7 的 `yield from` 语法来生成前三个数。然后，我们直接跳到 5，并继续往前走，直到要求的最大值。

```php
function generatePrimes($max)
{
  yield from [1,2,3];
  for ($x = 5; $x < $max; $x++)
  {
    if($x & 1) {
        $prime = TRUE;
        for($i = 3; $i < $x; $i++) {
            if(($x % $i) === 0) {
                $prime = FALSE;
                break;
            }
        }
        if ($prime) yield $x;
    }
  }
}
```

然后你可以设置一个数据库缓存适配器实例，作为核心的参数。

```php
$conn    = new Connection(include DB_CONFIG_FILE);
$dbCache = new Database(
  $conn, DB_TABLE, 'id', 'key', 'data', 'group');
$core    = new Core($dbCache);
```

另外，如果你想使用文件缓存适配器，这里有相应的代码。

```php
$fileCache = new File(CACHE_DIR);
$core    = new Core($fileCache);
```

如果你想清除缓存，可以这样做。

```php
$uriString = '/?group=' . Constants::DEFAULT_GROUP;
$cacheRequest = new Request($uriString, 'get');
$response = $core->removeByGroup($cacheRequest);
```

你可以使用 `time()` 和 `microtime()` 来查看这个脚本在有缓存和没有缓存的情况下运行了多久。

```php
$start = time() + microtime(TRUE);
echo "\nTime: " . $start;
```

接下来，生成一个缓存请求。状态码为200表示你能够从缓存中获得一个质数列表。

```php
$uriString = '/?key=Test1';
$cacheRequest = new Request($uriString, 'get');
$response = $core->getFromCache($cacheRequest);
$status   = $response->getStatusCode();
if ($status == 200) {
    $primes = json_decode($response->getBody()->getContents());
```

否则，你可以认为没有从缓存中获得任何东西，这意味着你需要生成质数，并将结果保存到缓存中。

```php
} else {
    $primes = array();
    foreach (generatePrimes(MAX_NUM) as $num) {
        $primes[] = $num;
    }
    $body = new TextStream(json_encode($primes));
    $response = $core->saveToCache(
    $cacheRequest->withBody($body));
}
```

然后，你可以检查停止时间，计算差值，并看看你的新质数列表。

```php
$time = time() + microtime(TRUE);
$diff = $time - $start;
echo "\nTime: $time";
echo "\nDifference: $diff";
var_dump($primes);
```

这里是在缓存中存储值之前的预期输出。

![](../../.gitbook/assets/image%20%28112%29.png)

现在你可以再次运行相同的程序，这次是从缓存中检索。

![](../../.gitbook/assets/image%20%28111%29.png)

考虑到我们的小质数发生器并不是世界上最高效的，也考虑到演示是在笔记本电脑上运行的，时间从30多秒降到了毫秒。

## 更多...

另一个可能的缓存适配器可以围绕着备用PHP缓存\(APC\)扩展中的命令来构建。这个扩展包括诸如 `apc_exists()`, `apc_store()`, `apc_fetch()` 和 `apc_clear_cache()`等函数。这些函数非常适合我们的`hasKey()`、`saveToCache()`、`getFromCache()`和`removeBy*()`函数。

## 另见

你可以考虑按照PSR-6对之前描述的缓存适配器类进行轻微的修改，这是一个针对缓存的标准建议。然而，这个标准的接受程度并不像PSR-7那样，所以我们决定在这里介绍的示例中不完全遵循这个标准。有关PSR-6的更多信息，请参考[http://www.php-fig.org/psr/psr-6/](http://www.php-fig.org/psr/psr-6/)。

