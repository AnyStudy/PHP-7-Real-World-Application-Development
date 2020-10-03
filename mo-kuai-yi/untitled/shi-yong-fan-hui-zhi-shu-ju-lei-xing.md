# 使用返回值数据类型

PHP 7允许您为函数的返回值指定数据类型。 但是，与标量类型提示不同，您不需要添加任何特殊的声明。

## 如何做...

1.本示例说明如何将数据类型分配给函数返回值。 要分配返回数据类型，请首先按照通常的方式定义函数。 在右括号后，添加一个空格，然后添加数据类型和冒号：

```php
function returnsString(DateTime $date, $format) : string
{
  return $date->format($format);
}
```

{% hint style="warning" %}
PHP 7.1引入了关于返回数据类型的变体，称为**可空类型**。 您需要做的就是将 `string` 更改为 `?string` 。 这允许函数返回字符串或 `NULL` 。
{% endhint %}

2.无论函数内部的数据类型如何，函数返回的所有内容都将转换为声明的数据类型作为返回值。 注意，在此示例中，将 `$a` ， `$b` 和 `$c` 的值相加在一起以产生单个和，然后将其返回。 通常，您希望返回值是数字数据类型。 但是，在这种情况下，返回数据类型被声明为字符串，它覆盖了PHP的类型处理过程：

```php
function convertsToString($a, $b, $c) : string
      
  return $a + $b + $c;
}
```

3.您还可以将类分配为返回数据类型。 在此示例中，我们分配了`DateTime` 的返回类型，该类型是PHP  `DateTime` 扩展的一部分：

```php
function makesDateTime($year, $month, $day) : DateTime
{
  $date = new DateTime();
  $date->setDate($year, $month, $day);
  return $date;
}
```

{% hint style="warning" %}
`makeDateTime()` 函数可能是标量类型提示的潜在候选者。 如果 `$year` ， `$month` 或 `$day`不是整数，则在调用 `setDate()` 时会生成警告。 如果使用标量类型提示，并且传递了错误的数据类型，则会引发 `TypeError` 。 虽然是否发出警告或引发 `TypeError` 确实无关紧要，但至少 `TypeError` 会导致错误使用您代码的开发人员坐立不安！
{% endhint %}

4.如果一个函数具有返回数据类型，而您在函数代码中返回了错误的数据类型，则在运行时将引发 `TypeError` 。 此函数指定 `DateTime` 的返回类型，但返回一个字符串。 当PHP引擎检测到差异时，将引发 `TypeError` ，但要等到运行时才会报错：

```php
function wrongDateTime($year, $month, $day) : DateTime
{
  return date($year . '-' . $month . '-' . $day);
}
```

{% hint style="warning" %}
如果返回数据类型类不是内置的PHP类（也就是SPL中的一个类），你需要确保该类已经被自动加载，或者被包含。
{% endhint %}

## 如何运行...

首先，将前面提到的函数放入一个名为 `chap_03_developing_functions_return_types_library.php` 的库文件中。这个文件需要包含在调用这些函数的 `chap_03_developing_functions_return_types.php` 脚本中。

```php
include (__DIR__ . '/chap_03_developing_functions_return_types_library.php');
```

现在，您可以调用 `returnString()` ，提供一个 `DateTime` 实例和一个格式化字符串：

```php
$date   = new DateTime();
$format = 'l, d M Y';
$now    = returnsString($date, $format);
echo $now . PHP_EOL;
var_dump($now);
```

如预期的那样，输出为字符串：

![](../../.gitbook/assets/image%20%2819%29.png)

现在，您可以调用 `convertsToString()` 并提供三个整数作为参数。 请注意，返回类型为字符串：

```php
echo "\nconvertsToString()\n";
var_dump(convertsToString(2, 3, 4));
```

![](../../.gitbook/assets/image%20%2815%29.png)

为了说明这一点，您可以将一个类分配为返回值，请使用三个整数参数调用 `makeDateTime()` ：

```php
echo "\nmakesDateTime()\n";
$d = makesDateTime(2015, 11, 21);
var_dump($d);
```

![](../../.gitbook/assets/image%20%2814%29.png)

最后，使用三个整数参数调用 `rongDateTime()` ：

```php
try {
    $e = wrongDateTime(2015, 11, 21);
    var_dump($e);
} catch (TypeError $e) {
    echo $e->getMessage();
}
```

请注意，在运行时会引发 `TypeError` ：

![](../../.gitbook/assets/image%20%2818%29.png)

## 更多...

PHP 7.1添加了一个新的返回值类型 `void` 。 当您不希望从函数中返回任何值时，将使用此方法。 有关更多信息，请参阅[https://wiki.php.net/rfc/void\_return\_type](https://wiki.php.net/rfc/void_return_type)。

## 参考

有关返回类型声明的更多信息，请参见以下文章：

* [http://php.net/manual/en/functions.arguments.php\#functions.arguments.type-declaration.strict](http://php.net/manual/en/functions.arguments.php#functions.arguments.type-declaration.strict)
* [https://wiki.php.net/rfc/return\_types](https://wiki.php.net/rfc/return_types)

有关可为空的类型的信息，请参考本文：

[https://wiki.php.net/rfc/nullable\_types](https://wiki.php.net/rfc/nullable_types)

