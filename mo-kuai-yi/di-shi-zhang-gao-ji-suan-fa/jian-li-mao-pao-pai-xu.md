# 建立冒泡排序

经典的冒泡排序是经常分配给大学生的练习。尽管如此，掌握这个算法还是很重要的，因为在很多场合，PHP内置的排序函数并不适用。一个例子是对一个多维数组进行排序，而排序键不是第一列。

冒泡排序的工作方式是递归迭代列表，并将当前值与下一个值交换。如果你想让项目按升序排列，如果下一个项目小于当前项目，就会发生交换。对于降序，如果相反，则会发生交换。当不再发生交换时，排序就结束了。

在下图中，第一次通关后，葡萄和香蕉互换，橙子和樱桃也互换。第2次通过后，葡萄和樱桃被交换。最后一关不再发生交换，冒泡排序结束。

![](../../.gitbook/assets/image%20%28128%29.png)

## 如何做...

1.我们不希望实际移动数组中的值，那样在资源使用方面会非常昂贵。取而代之的是，我们将使用一个链表，这在前面的示例中已经讨论过了。

​2. 首先我们使用前面示例中讨论的 `buildLinkedList()` 函数建立一个链表。

3. 然后我们定义了一个新的函数 `bubbleSort()`，它通过引用接受链表、主列表、排序字段和一个代表排序顺序（升序或降序）的参数。

```php
function bubbleSort(&$linked, $primary, $sortField, $order = 'A')
{
```

4. 需要的变量包括一个代表迭代次数、交换次数和基于链表的迭代器。

```php
  static $iterations = 0;
  $swaps = 0;
  $iterator = new ArrayIterator($linked);
```

5. 在 `while()` 循环中，我们只有在迭代仍然有效的情况下才会继续，也就是说仍然在进行中。然后我们获得当前的键和值，以及下一个键和值。请注意额外的`if()`语句，以确保迭代仍然有效（也就是说，确保我们不会从列表末尾掉下来！）。

```php
while ($iterator->valid()) {
  $currentLink = $iterator->current();
  $currentKey  = $iterator->key();
  if (!$iterator->valid()) break;
  $iterator->next();
  $nextLink = $iterator->current();
  $nextKey  = $iterator->key();
```

6. 接下来我们检查排序是升序还是降序。根据方向的不同，我们检查下一个值是否大于或小于当前值。比较的结果存储在 `$expr` 中。

```php
if ($order == 'A') {
    $expr = $primary[$linked->offsetGet
            ($currentKey)][$sortField] > 
            $primary[$linked->offsetGet($nextKey)][$sortField];
} else {
    $expr = $primary[$linked->offsetGet
            ($currentKey)][$sortField] < 
            $primary[$linked->offsetGet($nextKey)][$sortField];
}
```

7. 如果`$expr` 的值为`TRUE`，并且我们有有效的当前键和下一个键，那么这些值就会在链表中进行交换。我们也会对 `$swaps` 进行增量。

```php
if ($expr && $currentKey && $nextKey 
    && $linked->offsetExists($currentKey) 
    && $linked->offsetExists($nextKey)) {
    $tmp = $linked->offsetGet($currentKey);
    $linked->offsetSet($currentKey, 
    $linked->offsetGet($nextKey));
    $linked->offsetSet($nextKey, $tmp);
    $swaps++;
  }
}
```

8. 最后，如果发生了任何交换，我们需要再次进行迭代，直到不再有交换。据此，我们对同一方法进行递归调用。

```php
if ($swaps) bubbleSort($linked, $primary, $sortField, $order);
```

9. 真正的返回值是重新组织的链表。我们还返回迭代次数，只是为了参考。

```php
  return ++$iterations;
}
```

## 如何运行...

将前面讨论过的 `bubbleSort()` 函数添加到前面的示例中创建的 `include` 文件中。你可以使用前面的示例中讨论的相同逻辑来读取 `customer.csv` 文件，生成一个主列表。

```php
<?php
define('CUSTOMER_FILE', __DIR__ . '/../data/files/customer.csv');
include __DIR__ . '/chap_10_linked_list_include.php';
$headers = array();
$customer = readCsv(CUSTOMER_FILE, $headers);
```

然后，可以使用第一列作为排序键生成一个链表。

```php
$makeLink = function ($row) {
  return $row[0];
};
$linked = buildLinkedList($customer, $makeLink);
```

最后，调用 `bubbleSort()` 函数，提供链表和客户列表作为参数。你也可以提供一个排序列，在这个插图中第2列，代表账户余额，用字母 `A` 表示升序。`printCustomer()` 函数可以用来显示输出。

```php
echo 'Iterations: ' . bubbleSort($linked, $customer, 2, 'A') . PHP_EOL;
echo printCustomer($headers, $linked, $customer);
```

下面是一个输出的例子。

![](../../.gitbook/assets/image%20%28119%29.png)

