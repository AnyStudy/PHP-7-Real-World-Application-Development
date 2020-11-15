# 构建一个二分法查找类

传统的搜索通常是以顺序的方式在项目列表中进行的，这意味着要搜索的项目的最大数量可能与列表的长度相同! 这不是很有效率。如果你需要加快搜索速度，可以考虑实现二分法查找。

这个技术很简单：你在列表中找到中点，然后判断搜索项是小于、等于还是大于中点项。如果小于，则将上限设置为中点，只搜索列表的前半部分。如果大于，则将下限设置为中点，并且只搜索列表的后半部分。然后，你将继续把列表分成1/4、1/8、1/16，以此类推，直到找到搜索项（或未找到）。

{% hint style="info" %}
需要注意的是，虽然最大比较数比顺序搜索小得多（log n + 1，其中n是列表中的元素数，log是二进制对数），但搜索中涉及的列表必须先进行排序，这当然会降低性能。
{% endhint %}

## 如何做...

1.我们首先构造一个搜索类，`Application\Generic\Search`，它接受主列表作为参数。作为一个控制，我们还定义了一个属性，`$iterations`。

```php
namespace Application\Generic;
class Search
{
  protected $primary;
  protected $iterations;
  public function __construct($primary)
  {
    $this->primary = $primary;
  }
```

2. 接下来我们定义了一个方法，`binarySearch()`，它设置了搜索基础设施。首先要做的是建立一个单独的数组，`$search`，其中的 key 是搜索中包含的列的复合。然后我们按键进行排序。

```php
public function binarySearch(array $keys, $item)
{
  $search = array();
  foreach ($this->primary as $primaryKey => $data) {
    $searchKey = function ($keys, $data) {
      $key = '';
      foreach ($keys as $k) $key .= $data[$k];
          return $key;
    };
    $search[$searchKey($keys, $data)] = $primaryKey;
  }
  ksort($search);
```

3.  然后我们将键拉出到另一个数组 `$binary` 中，这样我们就可以基于数字键进行二进制排序。然后我们调用 `doBinarySearch()` ，其结果是我们的中间数组 `$search` 中的一个键，或者一个布尔值，FALSE。

```php
  $binary = array_keys($search);
  $result = $this->doBinarySearch($binary, $item);
  return $this->primary[$search[$result]] ?? FALSE;
}
```

4. 第一个 `doBinarySearch()` 初始化了一系列参数。`$iterations`、`$found`、`$loop`、`$done`和`$max`都是用来防止无尽循环的。`$upper`和`$lower`代表要检查的列表片断。

```php
public function doBinarySearch($binary, $item)
{
  $iterations = 0;
  $found = FALSE;
  $loop  = TRUE;
  $done  = -1;
  $max   = count($binary);
  $lower = 0;
  $upper = $max - 1;
```

5. 然后我们实现一个 `while()` 循环，并设置中点。

```php
 while ($loop && !$found) {
    $mid = (int) (($upper - $lower) / 2) + $lower;
```

6. 现在我们可以使用新的 PHP 7 **spaceship** 操作符，在一次比较中，它给我们提供了小于、等于或大于。如果小于，我们将上限设置为中点。如果大于，则将下限调整为中点。如果等于，我们就完成了，就可以自由回家了。

```php
switch ($item <=> $binary[$mid]) {
  // $item < $binary[$mid]
  case -1 :
  $upper = $mid;
  break;
  // $item == $binary[$mid]
  case 0 :
  $found = $binary[$mid];
  break;
  // $item > $binary[$mid]
  case 1 :
  default :
  $lower = $mid;
}
```

7. 现在来点循环控制。我们递增迭代次数，并确保它不会超过列表的大小。如果是这样，肯定有问题，我们需要退出。否则，我们检查上下限是否连续两次以上相同，在这种情况下，搜索项没有被找到。然后我们存储迭代次数，并返回找到的任何东西（或没有找到）。

```php
  $loop = (($iterations++ < $max) && ($done < 1));
    $done += ($upper == $lower) ? 1 : 0;
  }
  $this->iterations = $iterations;
  return $found;
}
```

## 如何运行...

首先，实现 `Application\Generic\Search` 类，定义本示例中描述的方法。接下来，定义一个调用程序，`chap_10_binary_search.php`，它设置自动加载并读取`customer.csv`文件作为搜索目标\(在前面的示例中讨论过\)。

```php
<?php
define('CUSTOMER_FILE', __DIR__ . '/../data/files/customer.csv');
include __DIR__ . '/chap_10_linked_list_include.php';
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Generic\Search;
$headers = array();
$customer = readCsv(CUSTOMER_FILE, $headers);
```

然后你可以创建一个新的搜索实例，并在列表中间的某个地方指定一个项目。在本例中，搜索基于第 1 列客户名称，项目为 `Todd Lindsey`。

```php
$search = new Search($customer);
$item = 'Todd Lindsey';
$cols = [1];
echo "Searching For: $item\n";
var_dump($search->binarySearch($cols, $item));
```

为了说明问题，在 `Application\Generic\Search::doBinarySearch()` 中的 `switch()` 之前添加这一行。

```php
echo 'Upper:Mid:Lower:<=> | ' . $upper . ':' . $mid . ':' . $lower . ':' . ($item <=> $binary[$mid]);
```

输出如图所示。请注意上、中、下限是如何调整的，直到找到项目。

![](../../.gitbook/assets/image%20%28118%29.png)

## 更多...

关于二分查找的更多信息，维基百科上有一篇很好的文章，介绍了基本的数学知识，网址是[https://en.wikipedia.org/wiki/Binary\_search\_algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm)。

