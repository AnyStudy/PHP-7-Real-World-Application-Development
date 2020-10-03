# 使用迭代器

**迭代器**是一种特殊类型的类，允许您**遍历**容器或列表。这里的关键词是遍历。这意味着迭代器提供了遍历一个列表的方法，但它不执行遍历本身。

SPL 提供了多种针对不同上下文而设计的通用迭代器和专用迭代器。 例如，`ArrayIterator` 旨在允许对数组进行面向对象的遍历。 `DirectoryIterator` 设计用于文件系统扫描。

某些 SPL 迭代器旨在与其他迭代器一起使用并增加价值。 示例包括 `FilterIterator` 和 `LimitIterator` 。 前者使您能够从父迭代器中删除不需要的值。 后者提供了分页功能，您可以在其中指定要遍历的项目数以及确定从何处开始的偏移量。

最后，还有一系列递归迭代器，它允许您重复调用父迭代器。一个例子是 `RecursiveDirectoryIterator` ，它从一个起点一直扫描到最后一个可能的子目录。

## 如何做...

1.我们首先检查 `ArrayIterator` 类。 它非常易于使用。 您需要做的就是提供一个数组作为构造函数的参数。 之后，您可以使用所有基于 SPL 的迭代器标准的任何方法，例如 `current()` ，`next()` 等。

```php
$iterator = new ArrayIterator($array);
```

{% hint style="warning" %}
使用 `ArrayIterator` 可以将标准的 PHP 数组转换为迭代器。 从某种意义上说，这在过程编程和 OOP 之间架起了一座桥梁。
{% endhint %}

2.作为迭代器的实际使用示例，请看一下此示例。 它需要一个迭代器，并产生一系列HTML `<ul>` 和 `<li>` 标记：

```php
function htmlList($iterator)
{
  $output = '<ul>';
  while ($value = $iterator->current()) {
    $output .= '<li>' . $value . '</li>';
    $iterator->next();
  }
  $output .= '</ul>';
  return $output;
}
```

3.另外，您可以简单地将 `ArrayIterator` 实例包装到一个简单的 `foreach()` 循环中：

```php
function htmlList($iterator)
{
  $output = '<ul>';
  foreach($iterator as $value) {
    $output .= '<li>' . $value . '</li>';
  }
  $output .= '</ul>';
  return $output;
}
```

4. `CallbackFilterIterator` 是为您可能正在使用的任何现有迭代器增加价值的好方法。 它允许您包装任何现有的迭代器并筛选输出。 在此示例中，我们将定义 `fetchCountryName()` ，该操作将循环访问数据库查询，该查询将生成国家/地区名称列表。 首先，我们使用第一章，《建立基础》中定义的 `Application\ Database\Connection` 类，通过查询定义一个 `ArrayIterator` 实例：

```php
function fetchCountryName($sql, $connection)
{
  $iterator = new ArrayIterator();
  $stmt = $connection->pdo->query($sql);
  while($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    $iterator->append($row['name']);
  }
  return $iterator;
}
```

5.接下来，我们定义了一个过滤器方法 `nameFilterIterator()` ，该方法接受部分国家名称和 `ArrayIterator` 实例作为参数：

```php
function nameFilterIterator($innerIterator, $name)
{
  if (!$name) return $innerIterator;
  $name = trim($name);
  $iterator = new CallbackFilterIterator($innerIterator, 
    function($current, $key, $iterator) use ($name) {
      $pattern = '/' . $name . '/i';
      return (bool) preg_match($pattern, $current);
    }
  );
  return $iterator;
}
```

6. `LimitIterator` 为您的应用程序添加了基本的分页功能。 要使用此迭代器，您只需提供父迭代器，偏移量和限制。 然后，`LimitIterator` 将仅产生从偏移量开始的整个数据集的子集。 以第2步中提到的相同示例为例，我们将对来自数据库查询的结果进行分页。 通过将由 `fetchCountryName()` 方法产生的迭代器包装在 `LimitIterator` 实例中，我们可以非常简单地完成此操作：

```php
$pagination = new LimitIterator(fetchCountryName(
$sql, $connection), $offset, $limit);
```

{% hint style="warning" %}
使用 `LimitIterator` 时要小心。 它需要在内存中设置整个数据集，以实现限制。 因此，当遍历大数据集时，这不是一个好工具。
{% endhint %}

7.迭代器可以堆叠。 在此简单示例中，`ArrayIterator` 由 `FilterIterator` 处理，而后者又受 `LimitIterator` 限制。 首先我们建立一个 `ArrayIterator` 的实例：

```php
$i = new ArrayIterator($a);
```

8.接下来，我们将 `ArrayIterator` 插入 `FilterIterator` 实例。 请注意，我们正在使用新的 PHP 7 匿名类功能。 在这种情况下，匿名类扩展了 `FilterIterator` 并覆盖了 `accept()` 方法，仅允许使用偶数ASCII码的字母：

```php
$f = new class ($i) extends FilterIterator { 
  public function accept()
  {
    $current = $this->current();
    return !(ord($current) & 1);
  }
};
```

9.最后，我们提供 `FilterIterator` 实例作为 `LimitIterator` 的参数，并提供一个偏移量（在此示例中为2）和一个限制（在此示例中为6）：

```php
$l = new LimitIterator($f, 2, 6);
```

10.然后我们可以定义一个简单的函数来显示输出，依次调用每个迭代器，在 `range('A'，'Z')` 产生的简单数组上查看结果：

```php
function showElements($iterator)
{
  foreach($iterator as $item)  echo $item . ' ';
  echo PHP_EOL;
}

$a = range('A', 'Z');
$i = new ArrayIterator($a);
showElements($i);
```

11.这是通过将 `FilterIterator` 堆叠在 `ArrayIterator` 之上而产生的其他字母的变体：

```php
$f = new class ($i) extends FilterIterator {
public function accept()
  {
    $current = $this->current();
    return !(ord($current) & 1);
  }
};
showElements($f);
```

12.这是另一个仅产生 `F H J L N P` 的变体，它演示了一个 `LimitIterator` 消耗了一个 `FilterIterator`，而 `FilterIterator` 消耗了一个 `ArrayIterator`。 这三个示例的输出如下：

```php
$l = new LimitIterator($f, 2, 6);
showElements($l);
```

![](../../.gitbook/assets/image%20%2819%29.png)

13.回到产生国家名称列表的示例，假设我们希望遍历由国家名称和ISO代码组成的多维数组，而不仅仅是国家名称。 到目前为止提到的简单迭代器是不够的。 相反，我们将使用所谓的递归迭代器。

14.首先，我们需要定义一个方法，使用前面提到的数据库连接类从数据库中提取所有列。和之前一样，我们返回一个 `ArrayIterator` 实例，该实例由查询中的数据填充：

```php
function fetchAllAssoc($sql, $connection)
{
  $iterator = new ArrayIterator();
  $stmt = $connection->pdo->query($sql);
  while($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    $iterator->append($row);
  }
  return $iterator;
}
```

15.乍一看，人们很想将一个标准 `ArrayIterator` 实例包装在 `RecursiveArrayIterator` 中。 不幸的是，这种方法仅执行一个浅层迭代，而没有提供我们想要的结果：对从数据库查询返回的多维数组的所有元素进行的迭代：

```php
$iterator = fetchAllAssoc($sql, $connection);
$shallow  = new RecursiveArrayIterator($iterator);
```

16.尽管这将返回一个迭代，其中每个项代表数据库查询中的一行，但在这种情况下，我们希望提供一个迭代，该迭代将迭代查询返回的所有行的所有列。 为了实现这一点，我们需要使用 `RecursiveIteratorIterator`。

17.Monty Python 粉丝会沉迷于此类名称的丰富讽刺意味，因为它让人对《Department of Redundancy Department》产生了美好的回忆。恰如其分地，这个类使我们的老朋友 `RecursiveArrayIterator` 类加班加点地工作，并对数组的所有级别进行**深度**迭代：

```php
$deep     = new RecursiveIteratorIterator($shallow);
```

## 如何运行...

作为一个实际示例，您可以开发一个测试脚本，该脚本使用迭代器实现过滤和分页。 对于此示例，您可以调用 `chap_03_developing_functions_filtered_and_paginated.php` 测试代码文件。

首先，按照最佳实践，将上述功能放入一个名为 `chap_03_developing_functions_iterators_library.php` 的包含文件中。 在测试脚本中，请确保包含此文件。

数据源是一个名为 `iso_country_codes` 的表，其中包含ISO2，ISO3和国家/地区名称。 数据库连接可以在 `config /db.config.php` 文件中。 您还可以包括上一章中讨论的 `Application\Database\Connection`类：

```php
define('DB_CONFIG_FILE', '/../config/db.config.php');
define('ITEMS_PER_PAGE', [5, 10, 15, 20]);
include (__DIR__ . '/chap_03_developing_functions_iterators_library.php');
include (__DIR__ . '/../Application/Database/Connection.php');
```

{% hint style="warning" %}
在PHP 7中，您可以将常量定义为数组。 在此示例中，`ITEMS_PER_PAGE` 被定义为一个数组，并用于生成 `HTML SELECT` 元素。
{% endhint %}

接下来，您可以处理国家名称和每页项目数的输入参数。 当前页码将从0开始，可以递增（下一页）或递减（上一页）：

```php
$name = strip_tags($_GET['name'] ?? '');
$limit  = (int) ($_GET['limit'] ?? 10);
$page   = (int) ($_GET['page']  ?? 0);
$offset = $page * $limit;
$prev   = ($page > 0) ? $page - 1 : 0;
$next   = $page + 1;
```

现在，您可以启动数据库连接并运行一个简单的 `SELECT` 查询。 这应该放在 `try {} catch {}` 块中。 然后，您可以将迭代器放置在 `try {}` 块中：

```php
try {
    $connection = new Application\Database\Connection(
      include __DIR__ . DB_CONFIG_FILE);
    $sql    = 'SELECT * FROM iso_country_codes';
    $arrayIterator    = fetchCountryName($sql, $connection);
    $filteredIterator = nameFilterIterator($arrayIterator, $name);
    $limitIterator    = pagination(
    $filteredIterator, $offset, $limit);
} catch (Throwable $e) {
    echo $e->getMessage();
}
```

现在我们准备好使用HTML。 在这个简单的示例中，我们提供了一个表单，该表单使用户可以选择每页的项目数和国家/地区名称：

```php
<form>
  Country Name:
  <input type="text" name="name" 
         value="<?= htmlspecialchars($name) ?>">
  Items Per Page: 
  <select name="limit">
    <?php foreach (ITEMS_PER_PAGE as $item) : ?>
      <option<?= ($item == $limit) ? ' selected' : '' ?>>
      <?= $item ?></option>
    <?php endforeach; ?>
  </select>
  <input type="submit" />
</form>
  <a href="?name=<?= $name ?>&limit=<?= $limit ?>
    &page=<?= $prev ?>">
  << PREV</a> 
  <a href="?name=<?= $name ?>&limit=<?= $limit ?>
    &page=<?= $next ?>">
  NEXT >></a>
<?= htmlList($limitIterator); ?>
```

输出结果如下所示：

![](../../.gitbook/assets/image%20%2832%29.png)

最后，为了测试国家数据库查询的递归迭代，你需要包含迭代器的库文件，以及 `Application\Database\Connection` 类：

```php
define('DB_CONFIG_FILE', '/../config/db.config.php');
include (__DIR__ . '/chap_03_developing_functions_iterators_library.php');
include (__DIR__ . '/../Application/Database/Connection.php');
```

与前面一样，应该将数据库查询封装在 `try {} catch {}` 块中。然后您可以将代码放在 `try {}` 块中测试递归迭代：

```php
try {
    $connection = new Application\Database\Connection(
    include __DIR__ . DB_CONFIG_FILE);
    $sql    = 'SELECT * FROM iso_country_codes';
    $iterator = fetchAllAssoc($sql, $connection);
    $shallow  = new RecursiveArrayIterator($iterator);
    foreach ($shallow as $item) var_dump($item);
    $deep     = new RecursiveIteratorIterator($shallow);
    foreach ($deep as $item) var_dump($item);     
} catch (Throwable $e) {
    echo $e->getMessage();
}
```

下面是你期望从 `RecursiveArrayIterator` 输出中看到的：

![](../../.gitbook/assets/image%20%2823%29.png)

下面是使用 `RecursiveIteratorIterator` 后的输出：

![](../../.gitbook/assets/image%20%2826%29.png)

