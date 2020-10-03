# 数据类型提示

在很多情况下，在开发函数时，您可能会在其他项目中重复使用相同的函数库。另外，如果您和一个团队一起工作，您的代码可能会被其他开发者使用。为了控制您的代码的使用，使用类型提示可能是合适的。这涉及到指定您的函数对该特定参数的数据类型。

## 如何做...

1.函数中的参数可以用类型提示作为前缀。下面的类型提示在PHP 5和PHP 7中都是可用的：

* Array
* Class
* Callable

2.如果对函数进行了调用，并且传递了错误的参数类型，程序就会抛出 `TypeError`。下面的例子需要一个数组、一个 `DateTime` 的实例和一个匿名函数：

```php
function someTypeHint(Array $a, DateTime $t, Callable $c)
{
  $message = '';
  $message .= 'Array Count: ' . count($a) . PHP_EOL;
  $message .= 'Date: ' . $t->format('Y-m-d') . PHP_EOL;
  $message .= 'Callable Return: ' . $c() . PHP_EOL;
  return $message;
}
```

> **小贴士**
>
> 您不必为每个参数都提供类型提示。 仅在提供其他数据类型会对功能处理产生负面影响的地方使用此技术。 例如，如果函数使用 `foreach()` 循环，不提供数组或实现 `Traversable` 的，将抛出错误。

3.在 PHP 7 中，如果适当的使用了 `declare()` 指令，则允许使用标量（即整数、浮点数、布尔值和字符串）类型提示。另一个函数演示了如何实现这一点。在包含您希望使用标量类型提示的函数的代码库文件的顶部，在PHP开头的标签后添加这个声明 `declare()` 指令：

```php
declare(strict_types=1);
```

4.现在您可以定义一个包含标量类型提示的函数了：

```php
function someScalarHint(bool $b, int $i, float $f, string $s)
{
  return sprintf("\n%20s : %5s\n%20s : %5d\n%20s " . 
                 ": %5.2f\n%20s : %20s\n\n",
                 'Boolean', ($b ? 'TRUE' : 'FALSE'),
                 'Integer', $i,
                 'Float',   $f,
                 'String',  $s);
}
```

5.在PHP 7中，假定已经声明了严格类型提示，则布尔类型提示的工作方式与其他三个标量类型（即整数，浮点型和字符串）有所不同。 您可以提供任何标量作为参数，并且不会引发 `TypeError` ！ 但是，传入的值一旦传递给函数将自动转换为布尔数据类型。 如果传递标量以外的任何数据类型（即数组或对象），则将引发 `TypeError` 。 这是定义布尔数据类型的函数的示例。 请注意，返回值将自动转换为布尔值：

```php
function someBoolHint(bool $b)
{
  return $b;
}
```

## 如何运行...

首先，您可以将三个函数 `someTypeHint()` ，`someScalarHint()` 和 `someBoolHint()` 放入同一个文件中。 在此示例中，我们将文件命名为 `chap_03_developing_functions_type_hints_library.php` 。 不要忘记在顶部添加 `declare(strict_types=1)` !

在我们的调用代码中，需要包含该文件：

```php
include (__DIR__ . DIRECTORY_SEPARATOR . 'chap_03_developing_functions_type_hints_library.php');
```

要测试 `someTypeHint()` ，请调用函数两次，一次调用正确的数据类型，第二次调用不正确的类型。 但是，这将引发 `TypeError` ，因此您需要将函数调用包装在 `try {...} catch（）{...}` 块中：

```php
try {
    $callable = function () { return 'Callback Return'; };
    echo someTypeHint([1,2,3], new DateTime(), $callable);
    echo someTypeHint('A', 'B', 'C');
} catch (TypeError $e) {
    echo $e->getMessage();
    echo PHP_EOL;
}
```

从本小节末尾显示的输出中可以看到，传递正确的数据类型时没有问题。 传递不正确的类型时，将引发 `TypeError` 。

{% hint style="warning" %}
在 PHP 7 中，某些错误已经被转换为 `Error` 类，它的处理方式与 `Exception` 有点类似。这意味着你可以捕获一个 `Error` 。 `TypeError` 是 `Error` 的一个特殊的子类，当传递给函数的数据类型不正确时，就会被抛出。

所有的 PHP 7  `Error` 类和 `Exception` 类都实现了 `Throwable` 接口。如果不确定是否需要捕获 `Error` 或 `Exception` ，可以添加一个捕获 `Throwable` 的块。
{% endhint %}

接下来你可以测试 `someScalarHint()` ，用正确和不正确的值调用两次，用 `try { .... } catch () { ...}` 块：

```php
try {
    echo someScalarHint(TRUE, 11, 22.22, 'This is a string');
    echo someScalarHint('A', 'B', 'C', 'D');
} catch (TypeError $e) {
    echo $e->getMessage();
}
```

正如预期的那样，对函数的第一次调用可以正常工作，第二次调用抛出 `TypeError` 。

当类型提示是布尔值时，传递的任何标量值都不会引发 `TypeError` ！ 取而代之的是，该值将被解释为其布尔等效值。 如果随后返回此值，则数据类型将更改为布尔值。

为了测试这一点，调用前面定义的 `someBoolHint()` 函数，并将任何标量值作为参数传入。 `var_dump()` 方法显示数据类型总是布尔值：

```php
try {
    // 正向结果
    $b = someBooleanHint(TRUE);
    $i = someBooleanHint(11);
    $f = someBooleanHint(22.22);
    $s = someBooleanHint('X');
    var_dump($b, $i, $f, $s);
    // 负向结果
    $b = someBooleanHint(FALSE);
    $i = someBooleanHint(0);
    $f = someBooleanHint(0.0);
    $s = someBooleanHint('');
    var_dump($b, $i, $f, $s);
} catch (TypeError $e) {
    echo $e->getMessage();
}
```

如果您现在尝试相同的函数调用，但传入非标量数据类型，则会抛出 \`TypeError\` ：

```php
try {
    $a = someBoolHint([1,2,3]);
    var_dump($a);
} catch (TypeError $e) {
    echo $e->getMessage();
}
try {
    $o = someBoolHint(new stdClass());
    var_dump($o);
} catch (TypeError $e) {
    echo $e->getMessage();
}
```

这是整体的输出：

![](../../.gitbook/assets/image%20%2813%29.png)

## 参考

PHP 7.1 引入了一个新的类型提示 `iterable` ，它允许数组、迭代器或生成器作为参数。更多信息请看这里。

{% embed url="https://wiki.php.net/rfc/iterable" %}

对于有关实现标量类型提示的基本原理的背景讨论，请查看本文：

{% embed url="https://wiki.php.net/rfc/scalar\_type\_hints\_v5" %}

