# 将电子表格上传到数据库

尽管 PHP 没有任何直接读取特定电子表格格式的能力（如 XLSX、ODS 等），但它有读取（CSV 逗号分隔值）文件的能力，因此，为了处理客户的电子表格，需要要求他们提供 CSV 格式的文件，或者需要自己进行转换。

## 准备...

当要把电子表格（即CSV文件）上传到数据库中时，有三个主要的考虑因素：

* 迭代一个（潜在的）庞大的文件
* 将每个电子表格的行提取到一个PHP数组中
* 在数据库中插入PHP数组

大规模的文件迭代使用前面的示例来处理，我们使用 `fgetcsv()` 函数将CSV行转换为PHP数组。最后，我们使用\(**PDO PHP Data Objects**\)类建立数据库连接并进行插入。

## 如何做...

1.首先，我们定义一个 `Application\Database\Connection` 类，该类根据提供给构造函数的一组参数创建一个 PDO 实例。

```php
<?php
  namespace Application\Database;

  use Exception;
  use PDO;

  class Connection
  { 
    const ERROR_UNABLE = 'ERROR: Unable to create database connection';    
    public $pdo;

    public function __construct(array $config)
    {
      if (!isset($config['driver'])) {
        $message = __METHOD__ . ' : ' . self::ERROR_UNABLE . PHP_EOL;
        throw new Exception($message);
    }
    $dsn = $config['driver'] 
    . ':host=' . $config['host'] 
    . ';dbname=' . $config['dbname'];
    try {
      $this->pdo = new PDO($dsn, 
      $config['user'], 
      $config['password'], 
      [PDO::ATTR_ERRMODE => $config['errmode']]);
    } catch (PDOException $e) {
      error_log($e->getMessage());
    }
  }

}
```

2.然后，我们加入一个 `Application\Iterator\LargeFile` 的实例。我们在这个类中添加了一个新的方法，这个方法被设计用来迭代 CSV 文件：

```php
protected function fileIteratorCsv()
{
  $count = 0;
  while (!$this->file->eof()) {
    yield $this->file->fgetcsv();
    $count++;
  }
  return $count;        
}    
```

3.我们还需要将 Csv 添加到允许的迭代器方法列表中：

```php
  const ERROR_UNABLE = 'ERROR: Unable to open file';
  const ERROR_TYPE   = 'ERROR: Type must be "ByLength", "ByLine" or "Csv"';
     
  protected $file;
  protected $allowedTypes = ['ByLine', 'ByLength', 'Csv'];
```

## 如何运行...

首先我们定义一个配置文件， `/path/to/source/config/db.config.php` ，其中包含数据库连接参数：

```php
<?php
return [
  'driver'   => 'mysql',
  'host'     => 'localhost',
  'dbname'   => 'php7cookbook',
  'user'     => 'cook',
  'password' => 'book',
  'errmode'  => PDO::ERRMODE_EXCEPTION,
];
```

接下来，我们利用第一章 《建立基础》 中定义的自动加载类，获得 `Application\Database\DatabaseConnection` 和 `Application\Iterator\LargeFile` 的实例，定义一个调用程序，`chap_02_uploading_csv_to_database.php` 。

```php
define('DB_CONFIG_FILE', '/../data/config/db.config.php');
define('CSV_FILE', '/../data/files/prospects.csv');
require __DIR__ . '/../../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
```

之后，我们设置了一个 `try {...} catch () {...}` 块，它可以捕获 `Throwable` 。这样我们就可以同时捕获异常和错误：

```php
try {
  // ...  
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

在 `try {...} catch () {...}` 块里面，我们得到一个连接和大文件迭代器类的实例：

```php
$connection = new Application\Database\Connection(
include __DIR__ . DB_CONFIG_FILE);
$iterator  = (new Application\Iterator\LargeFile(__DIR__ . CSV_FILE))
->getIterator('Csv');
```

然后我们利用PDO的 prepare/execute 功能。准备语句 SQL 中的 `?` 表示参数标记，参数在SQL执行时会被替换：

```php
$sql = 'INSERT INTO `prospects` '
  . '(`id`,`first_name`,`last_name`,`address`,`city`,`state_province`,'
  . '`postal_code`,`phone`,`country`,`email`,`status`,`budget`,`last_updated`) '
  . ' VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?)';
$statement = $connection->pdo->prepare($sql);
```

然后我们使用 `foreach()` 来循环执行文件迭代器。每一条 `yield` 语句都会产生一个代表数据库中一行值的数组。然后，我们可以使用这些值与 `PDOStatement::execute()` 来执行准备好的语句，将该行的值插入到数据库中。

```php
foreach ($iterator as $row) {
  echo implode(',', $row) . PHP_EOL;
  $statement->execute($row);
}
```

然后可以检查数据库，以验证数据是否已成功插入。

