# 理解 foreach\(\) 处理中的差异

在某些相对不明显的情况下，`foreach()` 循环内的代码行为在 PHP 5 和 PHP 7 之间会有所不同。首先，内部有了很大的改进，这意味着在速度上，`foreach()` 循环内的处理在 PHP 7 下运行会比 PHP 5 快得多。 在 PHP 5 中注意到的问题包括在 `foreach()` 循环内对数组使用 `current()` 和 `unset()` 。其他问题还包括在操作数组本身时通过引用传递值。

## 如何做...

1.考虑以下代码块:

```php
$a = [1, 2, 3];
foreach ($a as $v) {
  printf("%2d\n", $v);
  unset($a[1]);
}
```

2.在 PHP 5和7中，输出如下:

```bash
1
2
3
```

3.然而，如果你在循环之前添加一个赋值，行为就会改变：

```php
$a = [1, 2, 3];
$b = &$a;
foreach ($a as $v) {
  printf("%2d\n", $v);
  unset($a[1]);
}
```

4.比较 PHP 5和7的输出：

<table>
  <thead>
    <tr>
      <th style="text-align:left">PHP 5</th>
      <th style="text-align:left">PHP 7</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">
        <p><b>1</b>
        </p>
        <p><b>3</b>
        </p>
        <p>&lt;b&gt;&lt;/b&gt;</p>
      </td>
      <td style="text-align:left">
        <p><b>1</b>
        </p>
        <p><b>2</b>
        </p>
        <p><b>3</b>
        </p>
      </td>
    </tr>
  </tbody>
</table>

5.在PHP 5中，使用引用内部数组指针的函数也会引起不一致的行为。 以下面的代码为例：

```php
$a = [1,2,3];
foreach($a as &$v) {
    printf("%2d - %2d\n", $v, current($a));
}
```

> **小贴士**
>
> 每个数组都有一个内部指针，从1开始指向它的当前元素 `current()` 返回数组中的当前元素。

6.请注意，在 PHP 7 中运行的输出是规范化和一致的：

<table>
  <thead>
    <tr>
      <th style="text-align:left">PHP 5</th>
      <th style="text-align:left">PHP 7</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">
        <p><b>1 - 2</b>
        </p>
        <p><b>2 - 3</b>
        </p>
        <p><b>3 - 0</b>
        </p>
      </td>
      <td style="text-align:left">
        <p><b>1 - 1</b>
        </p>
        <p><b>2 - 1</b>
        </p>
        <p><b>3 - 1</b>
        </p>
      </td>
    </tr>
  </tbody>
</table>

7.一旦数组引用迭代完成，在 `foreach()` 循环中添加新元素在PHP 5中也是有问题的。 这一行为在PHP 7中已经变得一致。下面的代码示例演示了这一点：

```php
$a = [1];
foreach($a as &$v) {
    printf("%2d -\n", $v);
    $a[1]=2;
}
```

8.我们将观察到以下输出：

<table>
  <thead>
    <tr>
      <th style="text-align:left">PHP 5</th>
      <th style="text-align:left">PHP 7</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">
        <p><b>1 -</b>
        </p>
        <p>&lt;b&gt;&lt;/b&gt;</p>
      </td>
      <td style="text-align:left">
        <p><b>1 -</b>
        </p>
        <p><b>2 -</b>
        </p>
      </td>
    </tr>
  </tbody>
</table>

9.另一个在PHP 7中解决的PHP 5不良行为的例子是，在通过引用进行数组迭代的过程中，使用了修改数组的函数，如 `array_push()` 、 `array_pop()` 、`array_shift()` 和 `array_unshift()`。

我们看以下这个例子：

```php
$a=[1,2,3,4];
foreach($a as &$v) {
    echo "$v\n";
    array_pop($a);
}
```

10.我们将观察到以下输出：

<table>
  <thead>
    <tr>
      <th style="text-align:left">PHP 5</th>
      <th style="text-align:left">PHP 7</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">
        <p><b>1</b>
        </p>
        <p><b>2</b>
        </p>
        <p><b>1</b>
        </p>
        <p><b>1</b>
        </p>
      </td>
      <td style="text-align:left">
        <p><b>1</b>
        </p>
        <p><b>2</b>
        </p>
        <p>&lt;b&gt;&lt;/b&gt;</p>
        <p>&lt;b&gt;&lt;/b&gt;</p>
      </td>
    </tr>
  </tbody>
</table>

11.最后，我们还有一种情况，就是通过引用迭代一个数组，并使用嵌套的 `foreach()` 循环，而这个循环本身也是通过引用迭代同一个数组。在 PHP 5 中，这个结构根本无法工作。在 PHP 7 中，这个问题得到了修正。下面的代码块演示了这种行为：

```php
$a = [0, 1, 2, 3];
foreach ($a as &$x) {
       foreach ($a as &$y) {
         echo "$x - $y\n";
         if ($x == 0 && $y == 1) {
           unset($a[1]);
           unset($a[2]);
         }
       }
}
```

12.下面是输出结果：

<table>
  <thead>
    <tr>
      <th style="text-align:left">PHP 5</th>
      <th style="text-align:left">PHP 7</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">
        <p><b>0 - 0</b>
        </p>
        <p><b>0 - 1</b>
        </p>
        <p><b>0 - 3</b>
        </p>
        <p>&lt;b&gt;&lt;/b&gt;</p>
        <p>&lt;b&gt;&lt;/b&gt;</p>
      </td>
      <td style="text-align:left">
        <p><b>0 - 0</b>
        </p>
        <p><b>0 - 1</b>
        </p>
        <p><b>0 - 3</b>
        </p>
        <p><b>3 - 0</b>
        </p>
        <p><b>3 - 3</b>
        </p>
      </td>
    </tr>
  </tbody>
</table>

## 如何运行...

将这些代码示例添加到一个PHP文件中，`chap_02_foreach.php` 。在PHP 5下通过命令行运行该脚本。预期的输出如下：

![](../../.gitbook/assets/image%20%2830%29.png)

在 PHP 7下运行相同的脚本，注意不同之处：

![](../../.gitbook/assets/image%20%2835%29.png)

## 参考

如需了解更多信息，请查阅解决这一问题的RFC，该RFC已被接受。关于该RFC的文章可在以下网址找到：[https://wiki.php.net/rfc/php7\_foreach](https://wiki.php.net/rfc/php7_foreach)。

