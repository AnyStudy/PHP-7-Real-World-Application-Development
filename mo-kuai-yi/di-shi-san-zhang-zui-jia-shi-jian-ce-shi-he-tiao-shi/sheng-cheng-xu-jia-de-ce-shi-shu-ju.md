# 生成虚假的测试数据

测试和调试过程的一部分涉及纳入现实的测试数据。在某些情况下，特别是在测试数据库访问和制作基准时，需要大量的测试数据。实现这一点的一种方法是，纳入从网站上收集数据的过程，然后将数据以现实但随机的组合方式输入数据库。

## 如何做...

1.第一步是确定需要哪些数据来测试你的应用。另一个考虑因素是网站是否面向国际受众，还是市场主要来自单一国家？

2. 为了制作出一致的假造数据工具，将数据从源头转移到可用的数字格式是极其重要的。首选是一系列的数据库表。另一种没有那么吸引人的选择是CSV文件。

3. 您可能最终会分阶段转换数据。例如，您可以从一个列出国家代码和国家名称的网页中提取数据到一个文本文件中。

![](../../.gitbook/assets/image%20%28155%29.png)

4. 由于这个列表很短，所以很容易将其剪切并粘贴到文本文件中。

5. 我们可以搜索" "，然后用"\n"代替，这样就得到了。

![](../../.gitbook/assets/image%20%28157%29.png)

6. 这些数据可以导入到电子表格中，然后让你导出到CSV文件中。例如，phpMyAdmin就有这样的功能。

7. 为了便于说明，我们将假设我们生成的数据将最终进入`prospect`表。下面是用于创建该表的SQL语句。

```sql
CREATE TABLE 'prospects' (
  'id' int(11) NOT NULL AUTO_INCREMENT,
  'first_name' varchar(128) NOT NULL,
  'last_name' varchar(128) NOT NULL,
  'address' varchar(256) DEFAULT NULL,
  'city' varchar(64) DEFAULT NULL,
  'state_province' varchar(32) DEFAULT NULL,
  'postal_code' char(16) NOT NULL,
  'phone' varchar(16) NOT NULL,
  'country' char(2) NOT NULL,
  'email' varchar(250) NOT NULL,
  'status' char(8) DEFAULT NULL,
  'budget' decimal(10,2) DEFAULT NULL,
  'last_updated' datetime DEFAULT NULL,
  PRIMARY KEY ('id'),
  UNIQUE KEY 'UNIQ_35730C06E7927C74' ('email')
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

8. 现在是时候创建一个能够生成假数据的类了。然后，我们将为上面所示的每个字段创建方法来生成数据，除了id是自动生成的。

```php
namespace Application\Test;

use PDO;
use Exception;
use DateTime;
use DateInterval;
use PDOException;
use SplFileObject;
use InvalidArgumentsException;
use Application\Database\Connection;

class FakeData
{
  // data generation methods here
}

```

9. 接下来，我们定义了将作为过程的一部分使用的常量和属性。

```php
const MAX_LOOKUPS     = 10;
const SOURCE_FILE     = 'file';
const SOURCE_TABLE    = 'table';
const SOURCE_METHOD   = 'method';
const SOURCE_CALLBACK = 'callback';
const FILE_TYPE_CSV   = 'csv';
const FILE_TYPE_TXT   = 'txt';
const ERROR_DB        = 'ERROR: unable to read source table';
const ERROR_FILE      = 'ERROR: file not found';
const ERROR_COUNT     = 'ERROR: unable to ascertain count or ID column missing';
const ERROR_UPLOAD    = 'ERROR: unable to upload file';
const ERROR_LOOKUP    = 'ERROR: unable to find any IDs in the source table';

protected $connection;
protected $mapping;
protected $files;
protected $tables;
```

10. 然后，我们定义将用于生成随机字母、街道名称和电子邮件地址的属性。你可以把这些数组看作是种子，可以根据你的需要进行修改和/或扩展。举个例子，你可以用巴黎的街道名片段来代替法国人的名字。

```php
protected $alpha = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
protected $street1 = ['Amber','Blue','Bright','Broad','Burning',
  'Cinder','Clear','Dewy','Dusty','Easy']; // etc. 
protected $street2 = ['Anchor','Apple','Autumn','Barn','Beacon',
  'Bear','Berry','Blossom','Bluff','Cider','Cloud']; // etc.
protected $street3 = ['Acres','Arbor','Avenue','Bank','Bend',
  'Canyon','Circle','Street'];
protected $email1 = ['northern','southern','eastern','western',
  'fast','midland','central'];
protected $email2 = ['telecom','telco','net','connect'];
protected $email3 = ['com','net'];
```

11. 在构造函数中，我们接受一个`Connection`对象，用于数据库的访问，是一个假造数据的映射数组。

```php
public function __construct(Connection $conn, array $mapping)
{
  $this->connection = $conn;
  $this->mapping = $mapping;
}
```

12. 要生成街道名称，与其尝试创建一个数据库表，不如使用一组种子数组来生成随机组合，这样可能更有效。下面是一个例子，说明如何操作。

```php
public function getAddress($entry)
{
  return random_int(1,999)
   . ' ' . $this->street1[array_rand($this->street1)]
   . ' ' . $this->street2[array_rand($this->street2)]
   . ' ' . $this->street3[array_rand($this->street3)];
}

```

13. 根据所需的现实程度，你还可以建立一个数据库表，将邮政编码与城市相匹配。也可以随机生成邮政编码。下面是一个生成英国邮政编码的例子。

```php
public function getPostalCode($entry, $pattern = 1)
{
  return $this->alpha[random_int(0,25)]
   . $this->alpha[random_int(0,25)]
   . random_int(1, 99)
   . ' '
   . random_int(1, 9)
   . $this->alpha[random_int(0,25)]
   . $this->alpha[random_int(0,25)];
}
```

14. 伪造电子邮件的生成同样可以使用一组种子数组来产生随机结果。我们也可以通过编程让它接收一个现有的`$entry`数组，并带有参数，然后用这些参数来创建地址的名称部分。

```php
public function getEmail($entry, $params = NULL)
{
  $first = $entry[$params[0]] ?? $this->alpha[random_int(0,25)];
  $last  = $entry[$params[1]] ?? $this->alpha[random_int(0,25)];
  return $first[0] . '.' . $last
   . '@'
   . $this->email1[array_rand($this->email1)]
   . $this->email2[array_rand($this->email2)]
   . '.'
   . $this->email3[array_rand($this->email3)];
}
```

15. 对于日期生成，一种方法是接受一个现有的`$entry`数组作为参数。参数将是一个数组，其中第一个值是起始日期，第二个参数是要从起始日期中减去的最大天数。第二个参数是从起始日期中减去的最大天数。这有效地让你从一个范围中返回一个随机的日期。请注意，我们使用 `DateTime::sub()` 来减去随机的天数。`sub()` 需要一个 `DateInterval` 实例，我们使用 `P`、随机天数和`'D'`来构建。

```php
public function getDate($entry, $params)
{
  list($fromDate, $maxDays) = $params;
  $date = new DateTime($fromDate);
  $date->sub(new DateInterval('P' . random_int(0, $maxDays) . 'D'));
  return $date->format('Y-m-d H:i:s');
}
```

16. 正如本配方开头提到的，我们用于生成假造数据的数据源会有所不同。在某些情况下，如前面几步所示，我们使用种子数组，并建立假数据。在其他情况下，我们可能希望使用文本或CSV文件作为数据源。下面是这样的方法可能的样子。

```php
public function getEntryFromFile($name, $type)
{
  if (empty($this->files[$name])) {
      $this->pullFileData($name, $type);
  }
  return $this->files[$name][
  random_int(0, count($this->files[$name]))];
}
```

17. 你会注意到，我们首先需要将文件数据拉到一个数组中，形成返回值。下面是为我们做这件事的方法。如果没有找到指定的文件，我们会抛出一个`Exception`。文件类型被确定为我们的一个类常量。`FILE_TYPE_TEXT`或 `FILE_TYPE_CSV.` 根据文件类型，我们使用`fgetcsv()`或`fgets()`。

```php
public function pullFileData($name, $type)
{
  if (!file_exists($name)) {
      throw new Exception(self::ERROR_FILE);
  }
  $fileObj = new SplFileObject($name, 'r');
  if ($type == self::FILE_TYPE_CSV) {
      while ($data = $fileObj->fgetcsv()) {
        $this->files[$name][] = trim($data);
      }
  } else {
      while ($data = $fileObj->fgets()) {
        $this->files[$name][] = trim($data);
      }
  }
```

18. 这个过程中最复杂的可能就是从数据库表中随机抽取数据。我们接受表名、构成主键的列名、查找表中数据库列名和目标列名之间的映射数组作为参数。

```php
public function getEntryFromTable($tableName, $idColumn, $mapping)
{
  $entry = array();
  try {
      if (empty($this->tables[$tableName])) {
        $sql  = 'SELECT ' . $idColumn . ' FROM ' . $tableName 
          . ' ORDER BY ' . $idColumn . ' ASC LIMIT 1';
        $stmt = $this->connection->pdo->query($sql);
        $this->tables[$tableName]['first'] = 
          $stmt->fetchColumn();
        $sql  = 'SELECT ' . $idColumn . ' FROM ' . $tableName 
          . ' ORDER BY ' . $idColumn . ' DESC LIMIT 1';
        $stmt = $this->connection->pdo->query($sql);
        $this->tables[$tableName]['last'] = 
          $stmt->fetchColumn();
    }
```

19. 我们现在可以设置准备好的语句，并初始化一些关键变量。

```php
$result = FALSE;
$count = self::MAX_LOOKUPS;
$sql  = 'SELECT * FROM ' . $tableName 
  . ' WHERE ' . $idColumn . ' = ?';
$stmt = $this->connection->pdo->prepare($sql);
```

20. 实际的查找我们放在`do...while`循环里面。原因是我们至少需要运行一次查询才能得到结果。只有当我们没有得出结果的时候，我们才会继续循环。我们在最低ID和最高ID之间生成一个随机数，然后在查询中使用这个参数。注意，我们还递减了一个计数器，以防止无休止的循环。这是为了防止ID不连续，在这种情况下，我们可能会意外地生成一个不存在的ID。如果我们超过了最大尝试次数，仍然没有结果，我们就会抛出一个`Exception`。

```php
do {
  $id = random_int($this->tables[$tableName]['first'], 
    $this->tables[$tableName]['last']);
  $stmt->execute([$id]);
  $result = $stmt->fetch(PDO::FETCH_ASSOC);
} while ($count-- && !$result);
  if (!$result) {
      error_log(__METHOD__ . ':' . self::ERROR_LOOKUP);
      throw new Exception(self::ERROR_LOOKUP);
  }
} catch (PDOException $e) {
    error_log(__METHOD__ . ':' . $e->getMessage());
    throw new Exception(self::ERROR_DB);
}
```

21. 然后，我们使用映射数组从源表中使用目标表中预期的键来检索值。

```php
foreach ($mapping as $key => $value) {
  $entry[$value] = $result[$key] ?? NULL;
}
return $entry;
}
```

22. 这个类的核心是一个`getRandomEntry()`方法，它生成一个假数据的单数组。我们每次循环浏览`$mapping`一个条目，并检查各种参数。

```php
public function getRandomEntry()
{
  $entry = array();
  foreach ($this->mapping as $key => $value) {
    if (isset($value['source'])) {
      switch ($value['source']) {
```

23. 源参数用于实现有效的策略模式。我们支持四种不同的`source`可能性，都定义为类常数。第一个是`SOURCE_FILE`。在这种情况下，我们使用前面讨论的`getEntryFromFile()`方法。

```php
   case self::SOURCE_FILE :
            $entry[$key] = $this->getEntryFromFile(
            $value['name'], $value['type']);
          break;
```

24. 回调选项根据`$mapping`数组中提供的回调返回一个值。

```php
  case self::SOURCE_CALLBACK :
            $entry[$key] = $value['name']();
          break;
```

25. `SOURCE_TABLE`选项使用`$mapping`中定义的数据库表作为查找。注意，前面讨论过的`getEntryFromTable()`能够返回一个值的数组，这意味着我们需要使用`array_merge()`来整合结果。

```php
        case self::SOURCE_TABLE :
            $result = $this->getEntryFromTable(
            $value['name'],$value['idCol'],$value['mapping']);
            $entry = array_merge($entry, $result);
          break;
```

26.`SOURCE_METHOD`选项也是默认的，它使用的是这个类已经包含的方法。我们检查是否包含了参数，如果是，则将这些参数添加到方法调用中。注意使用`{}`来影响插值。如果我们在 PHP 7 中调用 `$this->$value['name']()` ，由于抽象语法树 \(AST\) 的重写，它就会像这样插值，`${$this->$value}['name']()` ，这不是我们想要的。

```php
        case self::SOURCE_METHOD :
        default :
          if (!empty($value['params'])) {
              $entry[$key] = $this->{$value['name']}(
                $entry, $value['params']);
          } else {
              $entry[$key] = $this->{$value['name']}($entry);
          }
        }
    }
  }
  return $entry;
}

```

27. 我们定义了一个方法，该方法循环使用`getRandomEntry()`来产生多行假数据。我们还添加了一个选项来插入到目标表。如果启用了这个选项，我们就会设置一个准备好的语句来插入，同时检查是否需要截断当前这个表中的任何数据。

```php
public function generateData(
$howMany, $destTableName = NULL, $truncateDestTable = FALSE)
{
  try {
      if ($destTableName) {
        $sql = 'INSERT INTO ' . $destTableName
          . ' (' . implode(',', array_keys($this->mapping)) 
          . ') '. ' VALUES ' . ' (:' 
          . implode(',:', array_keys($this->mapping)) . ')';
        $stmt = $this->connection->pdo->prepare($sql);
        if ($truncateDestTable) {
          $sql = 'DELETE FROM ' . $destTableName;
          $this->connection->pdo->query($sql);
        }
      }
  } catch (PDOException $e) {
      error_log(__METHOD__ . ':' . $e->getMessage());
      throw new Exception(self::ERROR_COUNT);
  }
```

28. 接下来，我们循环查看请求的数据行数，并运行`getRandomEntry()`。如果请求数据库插入，我们在`try/catch`块中执行准备好的语句。在任何情况下，我们使用`yield`关键字将这个方法变成一个生成器。

```php
for ($x = 0; $x < $howMany; $x++) {
  $entry = $this->getRandomEntry();
  if ($insert) {
    try {
        $stmt->execute($entry);
    } catch (PDOException $e) {
        error_log(__METHOD__ . ':' . $e->getMessage());
        throw new Exception(self::ERROR_DB);
    }
  }
  yield $entry;
}
}
```

{% hint style="info" %}
**最佳实践**

如果要返回的数据量很大，那么在数据产生的时候，最好是边产生边输出，这样可以节省数组所需的内存。
{% endhint %}

## 如何运行...

首先要做的是确保你已经为随机数据生成做好了数据准备。在这个配方中，我们将假定目标表是`prospects`，它的SQL数据库定义如下，如步骤7所示。

作为名字的数据源，你可以为名字和姓氏创建文本文件。在本例中，我们将引用`data/files`目录，以及文件`first_names.txt`和`surnames.txt`。对于城市、州或省、邮政编码和国家，可能需要从诸如http://www.geonames.org/ 这样的源头下载数据，然后上传到`world_city_data`表中。对于其余的字段，如地址、电子邮件、状态等，你可以使用`FakeData`内置的方法，或者定义回调。

接下来，一定要定义`Application\Test\FakeData`，添加步骤8到29中讨论的内容。完成后，创建一个名为`chap_13_fake_data.php`的调用程序，它设置了自动加载并使用了相应的类。你还应该定义与数据库配置路径相匹配的常量，以及命名文件。

```php
<?php
define('DB_CONFIG_FILE', __DIR__ . '/../config/db.config.php');
define('FIRST_NAME_FILE', __DIR__ . '/../data/files/first_names.txt');
define('LAST_NAME_FILE', __DIR__ . '/../data/files/surnames.txt');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Test\FakeData;
use Application\Database\Connection;
```

接下来，定义一个映射数组，使用目标表（prospects）中的列名作为键。然后，你需要为源、名称和任何其他需要的参数定义子键。首先，`'first_name'`和`'last_name'`都将使用一个文件作为源，`'name'`指向文件的名称，`'type'`表示文件的文本类型。

```php
$mapping = [
  'first_name'   => ['source' => FakeData::SOURCE_FILE,
  'name'         => FIRST_NAME_FILE,
  'type'         => FakeData::FILE_TYPE_TXT],
  'last_name'    => ['source' => FakeData::SOURCE_FILE,
  'name'         => LAST_NAME_FILE,
  'type'         => FakeData::FILE_TYPE_TXT],
```

`'address'`、`'email'`和`'last_updated'`都使用内置方法作为数据源。最后两个方法也定义了要传递的参数。

```php
  'address'      => ['source' => FakeData::SOURCE_METHOD,
  'name'         => 'getAddress'],
  'email'        => ['source' => FakeData::SOURCE_METHOD,
  'name'         => 'getEmail',
  'params'       => ['first_name','last_name']],
  'last_updated' => ['source' => FakeData::SOURCE_METHOD,
  'name'         => 'getDate',
  'params'       => [date('Y-m-d'), 365*5]]
```

`'电话'`、`'状态'`和`'预算'`都可以使用回调来提供假数据。

```php
  'phone'        => ['source' => FakeData::SOURCE_CALLBACK,
  'name'         => function () {
                    return sprintf('%3d-%3d-%4d', random_int(101,999),
                    random_int(101,999), random_int(0,9999)); }],
  'status'       => ['source' => FakeData::SOURCE_CALLBACK,
  'name'         => function () { $status = ['BEG','INT','ADV']; 
                    return $status[rand(0,2)]; }],
  'budget'       => ['source' => FakeData::SOURCE_CALLBACK,
                     'name' => function() { return random_int(0, 99999) 
                     + (random_int(0, 99) * .01); }]
```

最后，`'city'`从一个查找表中提取数据，该表也为你提供了`'mapping'`参数中所列字段的数据。然后你可以不定义这些键。注意，你还应该指定代表表主键的列。

```php
'city' => ['source' => FakeData::SOURCE_TABLE,
'name' => 'world_city_data',
'idCol' => 'id',
'mapping' => [
'city' => 'city', 
'state_province' => 'state_province',
'postal_code_prefix' => 'postal_code', 
'iso2' => 'country']
],
  'state_province'=> [],
  'postal_code'  => [],
  'country'    => [],
];
```

然后你可以定义目标表，一个`Connection`实例，并创建`FakeData`实例。一个`foreach()`循环将足以显示给定数量的条目。

```php
$destTableName = 'prospects';
$conn = new Connection(include DB_CONFIG_FILE);
$fake = new FakeData($conn, $mapping);
foreach ($fake->generateData(10) as $row) {
  echo implode(':', $row) . PHP_EOL;
}
```

10行的输出结果是这样的。

![](../../.gitbook/assets/image%20%28148%29.png)

## 更多...

下面是一个网站的总结，其中有各种数据清单，在生成测试数据时可以使用

| 数据类型 | URL | 备注 |
| :--- | :--- | :--- |
| Names | [http://nameberry.com/](http://nameberry.com/) |  |
|  | [http://www.babynamewizard.com/international-names-lists-popular-names-from-around-the-world](http://www.babynamewizard.com/international-names-lists-popular-names-from-around-the-world) |  |
| Raw Name Lists | [http://deron.meranda.us/data/census-dist-female-first.txt](http://deron.meranda.us/data/census-dist-female-first.txt) | US female first names |
|  | [http://deron.meranda.us/data/census-dist-male-first.txt](http://deron.meranda.us/data/census-dist-male-first.txt) | US male first names |
|  | [http://www.avss.ucsb.edu/NameFema.HTM](http://www.avss.ucsb.edu/NameFema.HTM) | US female first names |
|  | [http://www.avss.ucsb.edu/namemal.htm](http://www.avss.ucsb.edu/namemal.htm) | US male first names |
| Last Names | [http://names.mongabay.com/data/1000.html](http://names.mongabay.com/data/1000.html) | US surnames from census |
|  | [http://surname.sofeminine.co.uk/w/surnames/most-common-surnames-in-great-britain.html](http://surname.sofeminine.co.uk/w/surnames/most-common-surnames-in-great-britain.html) | British surnames |
|  | [https://gist.github.com/subodhghulaxe/8148971](https://gist.github.com/subodhghulaxe/8148971) | List of US surnames in the form of a PHP array |
|  | [http://www.dutchgenealogy.nl/tng/surnames-all.php](http://www.dutchgenealogy.nl/tng/surnames-all.php) | Dutch surnames |
|  | [http://www.worldvitalrecords.com/browsesurnames.aspx?l=A](http://www.worldvitalrecords.com/browsesurnames.aspx?l=A) | International surnames; just change the last letter\(s\) to get a list of names starting with that letter\(s\) |
| Cities | [http://www.travelgis.com/default.asp?framesrc=/cities/](http://www.travelgis.com/default.asp?framesrc=/cities/) | World cities |
|  | [https://www.maxmind.com/en/free-world-cities-database](https://www.maxmind.com/en/free-world-cities-database) |  |
|  | [https://github.com/David-Haim/CountriesToCitiesJSON](https://github.com/David-Haim/CountriesToCitiesJSON) |  |
|  | [http://www.fallingrain.com/world/index.html](http://www.fallingrain.com/world/index.html) |  |
| Postal Codes | [https://boutell.com/zipcodes/](https://boutell.com/zipcodes/) | US only; includes cities, postal codes, latitude and longitude |
|  | [http://www.geonames.org/export/](http://www.geonames.org/export/) | International; city names, postal codes, EVERYTHING!; free download |

