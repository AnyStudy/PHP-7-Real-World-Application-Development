# 定义一个映射器

映射器或数据映射器的工作方式与 hydrator 大致相同：将数据从一个模型（无论是数组还是对象）转换为另一个模型。一个关键的区别是，hydrator是通用的，不需要预先编程的对象属性名，而mapper则相反：它需要两个模型的属性名的精确信息。在本事例中，我们将演示如何使用映射器将数据从一个数据库表转换到另一个数据库表。

## 如何做...

1.我们首先定义一个 `Application\Database\Mapper\FieldConfig` 类，它包含了各个字段的映射指令。我们还定义了相应的类常量。

```php
namespace Application\Database\Mapper;
use InvalidArgumentException;
class FieldConfig
{
  const ERROR_SOURCE = 
    'ERROR: need to specify destTable and/or source';
  const ERROR_DEST   = 'ERROR: need to specify either '
    . 'both destTable and destCol or neither';
```

2. 键属性与相应的类常量一起被定义。`$key`用于标识对象。`$source`代表源数据库表中的列。`$destTable`和`$destCol`代表目标数据库表和列。如果定义了`$default`，则包含一个默认值或一个产生适当值的回调。

```php
public $key;
public $source;
public $destTable;
public $destCol;
public $default;
```

3. 现在我们将注意力转移到构造函数上，它分配默认值，构建键，并检查是否定义了`$source`或`$destTable`和`$destCol`。

```php
public function __construct($source = NULL,
                            $destTable = NULL,
                            $destCol   = NULL,
                            $default   = NULL)
{
  // generate key from source + destTable + destCol
  $this->key = $source . '.' . $destTable . '.' . $destCol;
  $this->source = $source;
  $this->destTable = $destTable;
  $this->destCol = $destCol;
  $this->default = $default;
  if (($destTable && !$destCol) || 
      (!$destTable && $destCol)) {
      throw new InvalidArgumentException(self::ERROR_DEST);
  }
  if (!$destTable && !$source) {
      throw new InvalidArgumentException(
        self::ERROR_SOURCE);
  }
}
```

{% hint style="info" %}
注意，我们允许源列和目的列为NULL。这样做的原因是，我们可能有一个源列在目的表中没有位置。同样，在目标表中可能有一些强制性的列，而这些列在源表中没有表示。
{% endhint %}

4. 在默认情况下，我们需要检查该值是否为回调值。如果是，我们运行回调；否则，我们返回直接值。注意，回调的定义应该使它们接受一个数据库表行作为参数。

```php
public function getDefault()
{
  if (is_callable($this->default)) {
      return call_user_func($this->default, $row);
  } else {
      return $this->default;
  }
}
```

5. 最后，为了总结这个类，我们为五个属性分别定义了`getter`和`setter`。

```php
public function getKey()
{
  return $this->key;
}

public function setKey($key)
{
  $this->key = $key;
}

// etc.
```

6. 接下来，我们定义一个`Application\Database\Mapper\Mapping`映射类，它接受源表和目的表的名称以及`FieldConfig`对象的数组作为参数。稍后你会看到，我们允许目标表属性是一个数组，因为映射可能是对两个或多个目标表的映射。

```php
namespace Application\Database\Mapper;
class Mapping
{
  protected $sourceTable;
  protected $destTable;
  protected $fields;
  protected $sourceCols;
  protected $destCols;

  public function __construct(
    $sourceTable, $destTable, $fields = NULL)
  {
    $this->sourceTable = $sourceTable;
    $this->destTable = $destTable;
    $this->fields = $fields;
  }
```

7. 然后我们为这些属性定义了获取器和设置器。

```php
public function getSourceTable()
{
  return $this->sourceTable;
}
public function setSourceTable($sourceTable)
{
  $this->sourceTable = $sourceTable;
}
// etc.
```

8. 对于字段配置，我们还需要提供添加单个字段的功能。没有必要提供键作为单独的参数，因为这可以从`FieldConfig`实例中获得。

```php
public function addField(FieldConfig $field)
{
  $this->fields[$field->getKey()] = $field;
  return $this;
}
```

9. 获取源列名的数组是极其重要的。问题是，源列名是埋藏在`FieldConfig`对象中的一个属性。相应地，当调用这个方法时，我们会循环浏览`FieldConfig`对象的数组，并对每个对象调用`getSource()`来获取源列名。

```php
public function getSourceColumns()
{
  if (!$this->sourceCols) {
      $this->sourceCols = array();
      foreach ($this->getFields() as $field) {
        if (!empty($field->getSource())) {
            $this->sourceCols[$field->getKey()] = 
              $field->getSource();
        }
      }
  }
  return $this->sourceCols;
}
```

10. 我们对`getDestColumns()`使用了类似的方法。与获取源列列表相比，最大的不同是我们只需要一个特定的目标表的列，如果定义了多个这样的表，这一点是非常关键的，我们不需要检查`$destCol`是否被设置，因为这一点已经在`FieldConfig`的构造函数中处理好了。

```php
public function getDestColumns($table)
{
  if (empty($this->destCols[$table])) {
      foreach ($this->getFields() as $field) {
        if ($field->getDestTable()) {
          if ($field->getDestTable() == $table) {
              $this->destCols[$table][$field->getKey()] = 
                $field->getDestCol();
          }
        }
      }
  }
  return $this->destCols[$table];
}
```

11. 最后，我们定义了一个方法，它的第一个参数是接受一个数组，代表源表的一行数据。第二个参数是目标表的名称。该方法产生一个准备插入到目标表中的数据数组。

12. 我们必须决定哪个优先：默认值（可以由回调提供），还是源表的数据。我们决定先测试一个默认值。如果默认值为NULL，则使用源表的数据。需要注意的是，如果需要进一步处理，默认值应该定义为回调。

```php
public function mapData($sourceData, $destTable)
{
  $dest = array();
  foreach ($this->fields as $field) {
    if ($field->getDestTable() == $destTable) {
        $dest[$field->getDestCol()] = NULL;
        $default = $field->getDefault($sourceData);
        if ($default) {
            $dest[$field->getDestCol()] = $default;
        } else {
            $dest[$field->getDestCol()] = 
                  $sourceData[$field->getSource()];
        }
    }
  }
  return $dest;
}
}
```

{% hint style="info" %}
请注意，在目标插入中会出现一些在源行中不存在的列。在这种情况下，`FieldConfig`对象的`$source`属性被保留为NULL，并提供一个默认值，作为一个标量值或回调。
{% endhint %}

13. 我们现在准备定义两个将生成SQL的方法。第一个这样的方法将生成一个SQL语句来从源表中读取。该语句将包括要准备的占位符（例如，使用`PDO::prepare()`）。

```php
public function getSourceSelect($where = NULL)
{
  $sql = 'SELECT ' 
  . implode(',', $this->getSourceColumns()) . ' ';
  $sql .= 'FROM ' . $this->getSourceTable() . ' ';
  if ($where) {
    $where = trim($where);
    if (stripos($where, 'WHERE') !== FALSE) {
        $sql .= $where;
    } else {
        $sql .= 'WHERE ' . $where;
    }
  }
  return trim($sql);
}
```

14. 另一种SQL生成方法会产生一个要为特定目标表准备的语句。请注意，占位符与": "前面的列名相同。

```php
public function getDestInsert($table)
{
  $sql = 'INSERT INTO ' . $table . ' ';
  $sql .= '( ' 
  . implode(',', $this->getDestColumns($table)) 
  . ' ) ';
  $sql .= ' VALUES ';
  $sql .= '( :' 
  . implode(',:', $this->getDestColumns($table)) 
  . ' ) ';
  return trim($sql);
}
```

## 如何运行...

使用步骤1至5中显示的代码来生成一个`Application\Database\Mapper\FieldConfig`类。将步骤6至14中显示的代码放入第二个`Application\Database\Mapper\Mapping`类中。

定义执行映射的调用程序之前，必须考虑源数据库表和目的数据库表。源表 `prospects_11` 的定义如下。

```php
CREATE TABLE `prospects_11` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `first_name` varchar(128) NOT NULL,
  `last_name` varchar(128) NOT NULL,
  `address` varchar(256) DEFAULT NULL,
  `city` varchar(64) DEFAULT NULL,
  `state_province` varchar(32) DEFAULT NULL,
  `postal_code` char(16) NOT NULL,
  `phone` varchar(16) NOT NULL,
  `country` char(2) NOT NULL,
  `email` varchar(250) NOT NULL,
  `status` char(8) DEFAULT NULL,
  `budget` decimal(10,2) DEFAULT NULL,
  `last_updated` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UNIQ_35730C06E7927C74` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

在这个例子中，你可以使用两个目标表，`customer_11`和`profile_11`，它们之间是1:1的关系。

```php
CREATE TABLE `customer_11` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(256) CHARACTER SET latin1 
     COLLATE latin1_general_cs NOT NULL,
  `balance` decimal(10,2) NOT NULL,
  `email` varchar(250) NOT NULL,
  `password` char(16) NOT NULL,
  `status` int(10) unsigned NOT NULL DEFAULT '0',
  `security_question` varchar(250) DEFAULT NULL,
  `confirm_code` varchar(32) DEFAULT NULL,
  `profile_id` int(11) DEFAULT NULL,
  `level` char(3) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UNIQ_81398E09E7927C74` (`email`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=utf8 COMMENT='Customers';

CREATE TABLE `profile_11` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `address` varchar(256) NOT NULL,
  `city` varchar(64) NOT NULL,
  `state_province` varchar(32) NOT NULL,
  `postal_code` varchar(10) NOT NULL,
  `country` varchar(3) NOT NULL,
  `phone` varchar(16) NOT NULL,
  `photo` varchar(128) NOT NULL,
  `dob` datetime NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=utf8 COMMENT='Customers';
```

现在，您可以定义一个名为`chap_11_mapper.php`的调用程序，该程序将设置自动加载并使用前面提到的两个类。 您还可以使用第5章与数据库交互中定义的`Connection`类。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
define('DEFAULT_PHOTO', 'person.gif');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Mapper\ { FieldConfig, Mapping };
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
```

为了演示的目的，在确定两个目标表存在之后，你可以截断两个表，这样出现的任何数据都是干净的。

```php
$conn->pdo->query('DELETE FROM customer_11');
$conn->pdo->query('DELETE FROM profile_11');
```

现在，已经准备好构建 `Mapping` 实例并将其填充为 `FieldConfig` 对象。每个 `FieldConfig` 对象都代表了源表和目标表之间的映射。在构造函数中，以数组的形式提供源表和两个目标表的名称。

```php
$mapper = new Mapping('prospects_11', ['customer_11','profile_11']);
```

你可以简单地从`profrom_11`和`customer_11`之间的字段映射开始，这里没有默认值。

```php
$mapper>addField(new FieldConfig('email','customer_11','email'))
```

请注意，`addField()`会返回当前的映射实例，所以不需要一直指定`$mapper->addField()`。这种技术被称为`fluent`接口。

名字字段比较棘手，在`profors_11`表中，它由两列表示，但在`customer_11`表中只有一列。相应地，你可以为`first_name`添加一个回调作为默认值，将两个字段合并为一个。你还需要为`last_name`定义一个条目，但其中没有目标映射。

```php
->addField(new FieldConfig('first_name','customer_11','name',
  function ($row) { return trim(($row['first_name'] ?? '') 
. ' ' .  ($row['last_name'] ?? ''));}))
->addField(new FieldConfig('last_name'))
```

`customer_11::status`字段可以使用null coalesce操作符\(??\)来判断它是否被设置。

```php
->addField(new FieldConfig('status','customer_11','status',
  function ($row) { return $row['status'] ?? 'Unknown'; }))
```

`customer_11::level` 字段在源表中没有表示，因此可以对源字段进行NULL录入，但要确保目的表和列的设置。同样，`customer_11::password`在源表中也不存在。在这种情况下，回调使用电话号码作为临时密码。

```php
->addField(new FieldConfig(NULL,'customer_11','level','BEG'))
->addField(new FieldConfig(NULL,'customer_11','password',
  function ($row) { return $row['phone']; }))
```

您也可以按以下方式设置`prospects_11`到`profile_11`的映射。请注意，由于`prospects_11`中不存在源照片和出生日期列，您可以设置任何适当的默认值。

```php
->addField(new FieldConfig('address','profile_11','address'))
->addField(new FieldConfig('city','profile_11','city'))
->addField(new FieldConfig('state_province','profile_11', 
'state_province', function ($row) { 
  return $row['state_province'] ?? 'Unknown'; }))
->addField(new FieldConfig('postal_code','profile_11',
'postal_code'))
->addField(new FieldConfig('phone','profile_11','phone'))
->addField(new FieldConfig('country','profile_11','country'))
->addField(new FieldConfig(NULL,'profile_11','photo',
DEFAULT_PHOTO))
->addField(new FieldConfig(NULL,'profile_11','dob',
date('Y-m-d')));
```

为了建立`profile_11`和`customer_11`表之间的1:1关系，我们使用回调将customer\_11::id、`customer_11::profile_id`和`profile_11::id`的值设置为`$row['id']`的值。

```php
$idCallback = function ($row) { return $row['id']; };
$mapper->addField(new FieldConfig('id','customer_11','id',
$idCallback))
->addField(new FieldConfig(NULL,'customer_11','profile_id',
$idCallback))
->addField(new FieldConfig('id','profile_11','id',$idCallback));
```

现在可以调用相应的方法生成三条SQL语句，一条从源表读取，两条插入两个目标表。

```php
$sourceSelect  = $mapper->getSourceSelect();
$custInsert    = $mapper->getDestInsert('customer_11');
$profileInsert = $mapper->getDestInsert('profile_11');
```

这三条语句可以立即准备以后执行。

```php
$sourceStmt  = $conn->pdo->prepare($sourceSelect);
$custStmt    = $conn->pdo->prepare($custInsert);
$profileStmt = $conn->pdo->prepare($profileInsert);
```

然后我们执行`SELECT`语句，从源表产生行。然后在循环中，我们为每个目标表生成`INSERT`数据，并执行相应的准备语句。

```php
$sourceStmt->execute();
while ($row = $sourceStmt->fetch(PDO::FETCH_ASSOC)) {
  $custData = $mapper->mapData($row, 'customer_11');
  $custStmt->execute($custData);
  $profileData = $mapper->mapData($row, 'profile_11');
  $profileStmt->execute($profileData);
  echo "Processing: {$custData['name']}\n";
}
```

下面是产生的三条SQL语句。

![](../../.gitbook/assets/image%20%28140%29.png)

然后，我们可以使用SQL `JOIN`直接从数据库中查看数据，以确保关系得到维护。

![](../../.gitbook/assets/image%20%28139%29.png)

