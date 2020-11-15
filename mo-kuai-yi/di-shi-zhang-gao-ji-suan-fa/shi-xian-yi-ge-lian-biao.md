# 实现一个链表

链表是指一个列表中的键指向另一个列表中的键。用数据库术语来比喻，就是你有一个包含数据的表，和一个指向数据的单独索引。一个索引可能会产生一个按ID排列的项目列表。另一个索引可能会根据标题产生一个列表，以此类推。链表的突出特点是，你不必触及原来的项目列表。

例如，在下图中，主列表中包含ID号和水果的名称。如果直接输出主列表，水果名称将按以下顺序显示。苹果，葡萄，香蕉，橙子，樱桃。另一方面，如果你要使用链表作为索引，结果输出的水果名称将是苹果、香蕉、樱桃、葡萄和橙子。

![](../../.gitbook/assets/image%20%28118%29.png)

## 如何做...

1.链表的主要用途之一是以不同的顺序显示项目。一种方法是创建键值对的迭代，其中键代表新的顺序，而值包含主列表中键的值。这样的函数可能是这样的。

```php
function buildLinkedList(array $primary,
                         callable $makeLink)
{
  $linked = new ArrayIterator();
  foreach ($primary as $key => $row) {
    $linked->offsetSet($makeLink($row), $key);
  }
  $linked->ksort();
  return $linked;
}
```

2. 我们使用一个匿名函数来生成新的键，以提供额外的灵活性。你还会注意到，我们按键进行了排序（`ksort()`），这样链表就会按照键的顺序进行迭代。

3. 我们在使用链表时需要做的就是对它进行迭代，但是要从主列表中产生结果，在本例中是`$customer`。

```php
foreach ($linked as $key => $link) {
  $output .= printRow($customer[$link]);
}
```

4. 请注意，我们绝对不碰主列表。这使我们能够生成多个链表，每个列表代表不同的顺序，同时保留我们的原始数据集。

5.链表的另一个重要用途是用于过滤。该技术与前面所示的技术类似。唯一的区别是我们扩展了 `buildLinkedList()` 函数，增加了一个过滤列和过滤值。

```php
function buildLinkedList(array $primary,
                         callable $makeLink,
                         $filterCol = NULL,
                         $filterVal = NULL)
{
  $linked = new ArrayIterator();
  $filterVal = trim($filterVal);
  foreach ($primary as $key => $row) {
    if ($filterCol) {
      if (trim($row[$filterCol]) == $filterVal) {
        $linked->offsetSet($makeLink($row), $key);
      }
    } else {
      $linked->offsetSet($makeLink($row), $key);
    }
  }
  $linked->ksort();
  return $linked;
}
```

6. 我们只将主列表中 `$filterCol` 所代表的值与 `$filterVal` 相匹配的链表中的项目纳入。迭代逻辑与步骤2中所示的相同。

7. 最后，链表的另一种形式是双向链表。在这种情况下，列表的构造方式是可以正向或反向迭代。在PHP中，我们很幸运地有一个SPL类，`SplDoublyLinkedList` ，它可以很好地完成这个任务。下面是一个建立双向链表的函数。

```php
function buildDoublyLinkedList(ArrayIterator $linked)
{
  $double = new SplDoublyLinkedList();
  foreach ($linked as $key => $value) {
    $double->push($value);
  }
  return $double;
}
```

{% hint style="info" %}
`SplDoublyLinkedList` 的术语可能会引起误解。`SplDoublyLinkedList::top()` 实际上指向列表的结尾，而`SplDoublyLinkedList::bottom()` 则指向列表的开头!
{% endhint %}

## 如何运行...

将第一个步骤的代码复制到一个文件中，`chap_10_linked_list_include.php`。为了演示链接列表的使用，你需要一个数据源。在这个例子中，你可以使用前面提到的`customer.csv`文件。它是一个CSV文件，有以下几列。

```php
"id","name","balance","email","password","status","security_question",
"confirm_code","profile_id","level"
```

你可以在前面提到的include文件中添加以下函数来生成客户的主列表，并显示他们的信息。注意，我们使用第一列id作为主键。

```php
function readCsv($fn, &$headers)
{
  if (!file_exists($fn)) {
    throw new Error('File Not Found');
  }
  $fileObj = new SplFileObject($fn, 'r');
  $result = array();
  $headers = array();
  $firstRow = TRUE;
  while ($row = $fileObj->fgetcsv()) {
    // store 1st row as headers
    if ($firstRow) {
      $firstRow = FALSE;
      $headers = $row;
    } else {
      if ($row && $row[0] !== NULL && $row[0] !== 0) {
        $result[$row[0]] = $row;
      }
    }
  }
  return $result;
}

function printHeaders($headers)
{
  return sprintf('%4s : %18s : %8s : %32s : %4s' . PHP_EOL,
                 ucfirst($headers[0]),
                 ucfirst($headers[1]),
                 ucfirst($headers[2]),
                 ucfirst($headers[3]),
                 ucfirst($headers[9]));
}

function printRow($row)
{
  return sprintf('%4d : %18s : %8.2f : %32s : %4s' . PHP_EOL,
                 $row[0], $row[1], $row[2], $row[3], $row[9]);
}

function printCustomer($headers, $linked, $customer)
{
  $output = '';
  $output .= printHeaders($headers);
  foreach ($linked as $key => $link) {
    $output .= printRow($customer[$link]);
  }
  return $output;
}
```

然后你可以定义一个调用程序，`chap_10_linked_list_in_order.php`，其中包括之前定义的文件，并读取`customer.csv`。

```php
<?php
define('CUSTOMER_FILE', __DIR__ . '/../data/files/customer.csv');
include __DIR__ . '/chap_10_linked_list_include.php';
$headers = array();
$customer = readCsv(CUSTOMER_FILE, $headers);
```

然后你可以定义一个匿名函数，在链表中产生一个键。在本例中，定义一个函数，将第1列（name）分解为名和姓。

```php
$makeLink = function ($row) {
  list($first, $last) = explode(' ', $row[1]);
  return trim($last) . trim($first);
};
```

然后你可以调用该函数来建立链表，并使用 `printCustomer()` 来显示结果。

```php
$linked = buildLinkedList($customer, $makeLink);
echo printCustomer($headers, $linked, $customer);
```

下面是输出可能出现的方式。

![](../../.gitbook/assets/image%20%28119%29.png)

要产生一个过滤的结果，请修改步骤4中讨论的`buildLinkedList()`。然后你可以添加逻辑，检查过滤列的值是否与过滤器中的值相匹配。

```php
define('LEVEL_FILTER', 'INT');

$filterCol = 9;
$filterVal = LEVEL_FILTER;
$linked = buildLinkedList($customer, $makeLink, $filterCol, $filterVal);
```

## 更多...

PHP 7.1引入了`[ ]`作为`list()`的替代。如果看一下前面提到的匿名函数，可以在 PHP 7.1 中重写如下。

```php
$makeLink = function ($row) {
  [$first, $last] = explode(' ', $row[1]);
  return trim($last) . trim($first);
};
```

更多信息，见[https://wiki.php.net/rfc/short\_list\_syntax](https://wiki.php.net/rfc/short_list_syntax)。

