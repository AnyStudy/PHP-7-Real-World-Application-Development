# 实现一个搜索引擎

为了实现搜索引擎，我们需要为搜索中包含的多个列做出规定。此外，重要的是要认识到，搜索项可能会在字段中间被发现，而用户很少会提供足够的信息来进行精确匹配。相应地，我们将严重依赖SQL `LIKE %value%` 子句。

## 如何做...

1.首先，我们定义一个基本的类来存放搜索标准。这个对象包含三个属性：key，它最终代表一个数据库列；操作符（`LIKE`、`<`、`>`等）；还有一个可选的项目。之所以说项是可选的，是因为有些操作符，如`IS NOT NULL`，不需要特定的数据。

```php
namespace Application\Database\Search;
class Criteria
{
  public $key;
  public $item;
  public $operator;
  public function __construct($key, $operator, $item = NULL)
  {
    $this->key  = $key;
    $this->operator = $operator;
    $this->item = $item;
  }
}
```

2. 接下来我们需要定义一个类，`Application\Database\Search\Engine`，并提供必要的类常量和属性。`$columns`和`$mapping`之间的区别在于，`$columns`持有的信息将最终出现在HTML `SELECT`字段（或等价物）中。出于安全考虑，我们不想暴露数据库列的实际名称，因此需要另一个数组`$mapping`。

```php
namespace Application\Database\Search;
use PDO;
use Application\Database\Connection;
class Engine
{
  const ERROR_PREPARE = 'ERROR: unable to prepare statement';
  const ERROR_EXECUTE = 'ERROR: unable to execute statement';
  const ERROR_COLUMN  = 'ERROR: column name not on list';
  const ERROR_OPERATOR= 'ERROR: operator not on list';
  const ERROR_INVALID = 'ERROR: invalid search criteria';

  protected $connection;
  protected $table;
  protected $columns;
  protected $mapping;
  protected $statement;
  protected $sql = '';
```

3. 接下来，我们定义一组我们愿意支持的操作符。键代表实际的SQL。值是将出现在表格中的内容。

```php
  protected $operators = [
      'LIKE'     => 'Equals',
      '<'        => 'Less Than',
      '>'        => 'Greater Than',
      '<>'       => 'Not Equals',
      'NOT NULL' => 'Exists',
  ];
```

4. 构造函数接受一个数据库连接实例作为参数。对于我们的目的，我们将使用`Application\Database\Connection`，定义在第5章，与数据库的交互。我们还需要提供数据库表的名称，以及 `$columns`，一个包含任意列键和标签的数组，它们将出现在HTML表单中。这将引用`$mapping`，其中键与`$columns`匹配，但值代表实际的数据库列名。

```php
public function __construct(Connection $connection, 
                            $table, array $columns, array $mapping)
{
  $this->connection  = $connection;
  $this->setTable($table);
  $this->setColumns($columns);
  $this->setMapping($mapping);
}
```

5. 在构造函数之后，我们提供了一系列有用的getter和setter。

```php
public function setColumns($columns)
{
  $this->columns = $columns;
}
public function getColumns()
{
  return $this->columns;
}
// etc.
```

6. 最关键的方法可能是建立要准备的SQL语句。在最初的 `SELECT` 设置之后，我们添加一个 `WHERE` 子句，使用 `$mapping` 来添加实际的数据库列名。然后我们添加操作符，并实现`switch()`，根据操作符，可以添加或不添加代表搜索项的命名占位符。

```php
public function prepareStatement(Criteria $criteria)
{
  $this->sql = 'SELECT * FROM ' . $this->table . ' WHERE ';
  $this->sql .= $this->mapping[$criteria->key] . ' ';
  switch ($criteria->operator) {
    case 'NOT NULL' :
      $this->sql .= ' IS NOT NULL OR ';
      break;
    default :
      $this->sql .= $criteria->operator . ' :' 
      . $this->mapping[$criteria->key] . ' OR ';
  }
```

7. 现在已经定义了核心的`SELECT`，我们删除任何尾部的`OR`关键字，并添加一个子句，使结果根据搜索列进行排序。然后将该语句发送到数据库中进行准备。

```php
  $this->sql = substr($this->sql, 0, -4)
    . ' ORDER BY ' . $this->mapping[$criteria->key];
  $statement = $this->connection->pdo->prepare($this->sql);
  return $statement;
}
```

8. 现在我们准备好了，进入正题，`search()`方法。我们接受一个`Application\Database\Search\Criteria`对象作为参数。这确保了我们至少有一个项目键和操作符。为了安全起见，我们添加一个`if()`语句来检查这些属性。

```php
public function search(Criteria $criteria)
{
  if (empty($criteria->key) || empty($criteria->operator)) {
    yield ['error' => self::ERROR_INVALID];
    return FALSE;
  }
```

9. 然后，我们使用 `try` / `catch` 调用 `prepareStatement()` 来捕获错误。

```php
try {
    if (!$statement = $this->prepareStatement($criteria)) {
      yield ['error' => self::ERROR_PREPARE];
      return FALSE;
}
```

10. 接下来我们建立一个将提供给 `execute()` 的参数数组。键代表数据库列名，它在准备好的语句中被用作占位符。注意，我们不使用 `=` ，而是使用`LIKE %value% construct`。

```php
$params = array();
switch ($criteria->operator) {
  case 'NOT NULL' :
    // do nothing: already in statement
    break;
    case 'LIKE' :
    $params[$this->mapping[$criteria->key]] = 
    '%' . $criteria->item . '%';
    break;
    default :
    $params[$this->mapping[$criteria->key]] = 
    $criteria->item;
}
```

11. 执行该语句，并使用 `yield` 关键字返回结果，这实际上是把这个方法变成了一个生成器。

```php
 $statement->execute($params);
    while ($row = $statement->fetch(PDO::FETCH_ASSOC)) {
      yield $row;
    }
  } catch (Throwable $e) {
    error_log(__METHOD__ . ':' . $e->getMessage());
    throw new Exception(self::ERROR_EXECUTE);
  }
  return TRUE;
}
```

## 如何运行...

将这个示例中的代码放在 `Application\Database\Search`下的`Criteria.php`和`Engine.php`文件中。然后你可以定义一个调用脚本，`chap_10_search_engine.php`，用来设置自动加载。你可以利用第5章，与数据库的交互中讨论的 `Application\Database\Connection` 类，以及第6章，构建可扩展网站中涉及的表单元素类。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');

use Application\Database\Connection;
use Application\Database\Search\ { Engine, Criteria };
use Application\Form\Generic;
use Application\Form\Element\Select;
```

现在可以定义哪些数据库列将出现在表格中，以及一个匹配的映射文件。

```php
$dbCols = [
  'cname' => 'Customer Name',
  'cbal' => 'Account Balance',
  'cmail' => 'Email Address',
  'clevel' => 'Level'
];

$mapping = [
  'cname' => 'name',
  'cbal' => 'balance',
  'cmail' => 'email',
  'clevel' => 'level'
];
```

现在可以设置数据库连接并创建搜索引擎实例。

```php
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
$engine = new Engine($conn, 'customer', $dbCols, $mapping);
```

为了显示适当的下拉式 `SELECT` 元素，我们基于 `Application\Form\*` 类定义了包装器和元素。

```php
$wrappers = [
  Generic::INPUT => ['type' => 'td', 'class' => 'content'],
  Generic::LABEL => ['type' => 'th', 'class' => 'label'],
  Generic::ERRORS => ['type' => 'td', 'class' => 'error']
];

// define elements
$fieldElement = new Select('field',
                Generic::TYPE_SELECT,
                'Field',
                $wrappers,
                ['id' => 'field']);
                $opsElement = new Select('ops',
                Generic::TYPE_SELECT,
                'Operators',
                $wrappers,
                ['id' => 'ops']);
                $itemElement = new Generic('item',
                Generic::TYPE_TEXT,
                'Searching For ...',
                $wrappers,
                ['id' => 'item','title' => 'If more than one item, separate with commas']);
                $submitElement = new Generic('submit',
                Generic::TYPE_SUBMIT,
                'Search',
                $wrappers,
                ['id' => 'submit','title' => 'Click to Search', 'value' => 'Search']);
```

然后我们获取输入参数（如果定义了），设置表单元素选项，创建搜索条件，并运行搜索。

```php
$key  = (isset($_GET['field'])) 
? strip_tags($_GET['field']) : NULL;
$op   = (isset($_GET['ops'])) ? $_GET['ops'] : NULL;
$item = (isset($_GET['item'])) ? strip_tags($_GET['item']) : NULL;
$fieldElement->setOptions($dbCols, $key);
$itemElement->setSingleAttribute('value', $item);
$opsElement->setOptions($engine->getOperators(), $op);
$criteria = new Criteria($key, $op, $item);
$results = $engine->search($criteria);
?>
```

显示逻辑主要面向渲染表单。更详细的介绍在第6章《构建可扩展网站》中讨论，但我们在这里展示的是核心逻辑。

```php
  <form name="search" method="get">
  <table class="display" cellspacing="0" width="100%">
    <tr><?= $fieldElement->render(); ?></tr>
    <tr><?= $opsElement->render(); ?></tr>
    <tr><?= $itemElement->render(); ?></tr>
    <tr><?= $submitElement->render(); ?></tr>
    <tr>
    <th class="label">Results</th>
      <td class="content" colspan=2>
      <span style="font-size: 10pt;font-family:monospace;">
      <table>
      <?php foreach ($results as $row) : ?>
        <tr>
          <td><?= $row['id'] ?></td>
          <td><?= $row['name'] ?></td>
          <td><?= $row['balance'] ?></td>
          <td><?= $row['email'] ?></td>
          <td><?= $row['level'] ?></td>
        </tr>
      <?php endforeach; ?>
      </table>
      </span>
      </td>
    </tr>
  </table>
  </form>
```

以下是浏览器的输出示例。

![](../../.gitbook/assets/image%20%28127%29.png)

