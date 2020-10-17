# 使用生成器编写自己的迭代器

在前面的示例中，我们演示了如何使用 PHP 7 SPL 中提供的迭代器。但是，如果示例不能提供给你一个给定项目所需要的东西呢？一个解决方案是开发一个函数，它不是建立一个数组然后返回，而是使用 `yield` 关键字通过迭代的方式逐步返回值。这样的函数被称为**生成器**。事实上，在后台，PHP引擎会自动将你的函数转换为一个名为 `Generator` 的特殊内置类。

这种方法有几个优点。主要的好处是当你有一个大的容器要遍历时（也就是解析一个庞大的文件）。传统的方法是建立一个数组，然后返回这个数组。这样做的问题是，你实际上是把所需的内存量增加了一倍! 同时，性能也会受到影响，因为只有在返回最终数组后才会有结果。

## 如何做...

1.在此示例中，我们在基于迭代器的函数库的基础上，添加了我们自己设计的生成器。 在这种情况下，我们将重复上面关于迭代器的部分中描述的功能，在这些迭代器中，我们堆叠了`ArrayIterator` ， `FilterIterator` 和 `LimitIterator`。

2.因为我们需要访问源数组，所需的过滤器，页数和每页项数，所以我们将适当的参数包含在单个 `filterResultsGenerator()` 函数中。 然后，我们根据页码和限制（即每页的项目数）计算偏移量。 接下来，我们遍历数组，应用过滤器，如果尚未达到偏移量，则继续循环；如果已达到极限，则中断：

```php
function filteredResultsGenerator(array $array, $filter, $limit = 10, $page = 0)
  {
    $max    = count($array);
    $offset = $page * $limit;
    foreach ($array as $key => $value) {
      if (!stripos($value, $filter) !== FALSE) continue;
      if (--$offset >= 0) continue;
      if (--$limit <= 0) break; 
      yield $value;
    }
  }
```

3.您会注意到此函数与其他函数之间的主要区别是 `yield` 关键字。 此关键字的作用是通知PHP引擎生成 `Generator` 实例并封装代码。

## 如何运行...

为了演示 `filteredResultsGenerator()` 函数的用法，我们将实现一个Web应用程序，扫描一个网页，并产生一个从 `HREF` 属性中筛选出来的URLs的分页列表。

首先，您需要将 `filteredResultsGenerator()` 函数的代码添加到先前示例中使用的库文件中，然后将前面描述的函数放入包含文件 `chap_03_developing_functions_iterators_library.php` 中。

接下来，定义一个测试脚本，`chap_03_developing_functions_using_generator.php` ，它既包括函数库，也包括定义 `Application\Web\Hoover` 的文件。

```php
include (__DIR__ . DIRECTORY_SEPARATOR . 'chap_03_developing_functions_iterators_library.php');
include (__DIR__ . '/../Application/Web/Hoover.php');
```

然后，您将需要从用户那里收集有关要扫描的URL，用作筛选器的字符串，每页多少个项目以及当前页码的输入。

{% hint style="warning" %}
空合并操作符（`??`）是从Web获取输入的理想选择。 如果未定义，它不会生成任何通知。 如果未从用户输入接收到该参数，则可以提供默认值。
{% endhint %}

```php
$url    = trim(strip_tags($_GET['url'] ?? ''));
$filter = trim(strip_tags($_GET['filter'] ?? ''));
$limit  = (int) ($_GET['limit'] ?? 10);
$page   = (int) ($_GET['page']  ?? 0);
```

{% hint style="info" %}
**最佳实践**

网络安全应该永远是一个优先考虑的问题。在这个例子中，您可以使用 `strip_tags()`，也可以强制将数据类型改为整数 `(int)`，作为对用户输入进行清理的措施。
{% endhint %}

然后您就可以定义分页列表中上一页和下一页的链接中使用的变量。请注意，您也可以应用理智性检查，以确保下一页不会从结果集的末尾离开。为了简洁起见，在这个例子中没有应用这种理智检查：

```php
$next   = $page + 1;
$prev   = $page - 1;
$base   = '?url=' . htmlspecialchars($url) 
        . '&filter=' . htmlspecialchars($filter) 
        . '&limit=' . $limit 
        . '&page=';
```

然后，我们需要创建一个 `Application\Web\Hoover`实例，并从目标URL中获取HREF属性：

```php
$vac    = new Application\Web\Hoover();
$list   = $vac->getAttribute($url, 'href');
```

最后，我们定义HTML输出，渲染一个输入表单，并通过前面描述的 `htmlList()` 函数运行我们的生成器：

```php
<form>
<table>
<tr>
<th>URL</th>
<td>
<input type="text" name="url" 
  value="<?= htmlspecialchars($url) ?>"/>
</td>
</tr>
<tr>
<th>Filter</th>
<td>
<input type="text" name="filter" 
  value="<?= htmlspecialchars($filter) ?>"/></td>
</tr>
<tr>
<th>Limit</th>
<td><input type="text" name="limit" value="<?= $limit ?>"/></td>
</tr>
<tr>
<th>&nbsp;</th><td><input type="submit" /></td>
</tr>
<tr>
<td>&nbsp;</td>
<td>
<a href="<?= $base . $prev ?>"><-- PREV | 
<a href="<?= $base . $next ?>">NEXT --></td>
</tr>
</table>
</form>
<hr>
<?= htmlList(filteredResultsGenerator(
$list, $filter, $limit, $page)); ?>
```

这是输出示例：

![](../../.gitbook/assets/image%20%2810%29.png)

