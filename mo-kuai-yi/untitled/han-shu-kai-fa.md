# 函数开发

最困难的方面是决定如何将编程逻辑分解成函数。另一方面，在 PHP 中开发一个函数的机制非常简单。只需使用 `function` 关键字，给它起个名字，然后在后面加上括号。

## 如何做...

1.代码本身放在大括号中，如下所示：

```php
function someName ($parameter)
{ 
  $result = 'INIT';
  // 一个或多个影响 $result 的语句
  $result .= ' and also ' . $parameter;
  return $result; 
}
```

2.您可以定义一个或多个**参数**。要使其中一个参数成为可选的，只需指定一个默认值。如果您不确定要分配什么默认值，请使用 `NULL` ：

```php
function someOtherName ($requiredParam, $optionalParam = NULL)
  { 
    $result = 0;
    $result += $requiredParam;
    $result += $optionalParam ?? 0;
    return $result; 
  }
```

{% hint style="info" %}
您无法在同一命名空间下重复定义相同名称的函数。 此定义将产生错误：

```php
function someTest()
{
  return 'TEST';
}
function someTest($a)
{
  return 'TEST:' . $a;
}
```
{% endhint %}

3.如果您不知道将为函数提供多少个参数，或者您想允许无限数量的参数时，请使用 `...` 后面跟着一个变量名。 提供的所有参数将在变量中显示为数组：

```php
function someInfinite(...$params)
{
  // 任何传入的参数都会进入到 $params 数组中
  return var_export($params, TRUE);
}
```

4.一个函数可以调用自己。这就是所谓的递归。以下函数执行递归目录扫描：

```php
function someDirScan($dir)
{
  // 使用 "static" 来保持 $list 的值
  static $list = array();
  // 获取此路径的文件和目录列表
  $list = glob($dir . DIRECTORY_SEPARATOR . '*');
  // 遍历
  foreach ($list as $item) {
    if (is_dir($item)) {
      $list = array_merge($list, someDirScan($item));
    }
  }
  return $list;
}
```

{% hint style="info" %}
`static` 关键字在函数中的使用在语言中已经超过12年了。 `static` 的作用是初始化变量一次（也就是在声明 `static` 的时候），然后在同一个请求中的函数调用之间保留其值。

如果你需要在 HTTP 请求之间保留一个变量的值，请确保 PHP 会话已经启动，并将该值存储在 `$_SESSION` 中。
{% endhint %}

5.当函数在 PHP 命名空间中定义时，函数是受限制的。这个特性可以被用来为函数库之间提供额外的逻辑分离。定义命名空间，您需要添加 `use` 关键字。 下面的例子被放置在不同的命名空间中。 请注意，即使函数名称相同，也不会发生冲突，因为它们彼此之间不可见。

6.我们在 `Alpha` 命名空间中定义了 `someFunction()` 。我们将其保存在一个单独的PHP文件中， `chap_03_developing_functions_namespace_alpha.php` ：

```php
<?php
namespace Alpha;

function someFunction()
{
  echo __NAMESPACE__ . ':' . __FUNCTION__ . PHP_EOL;
}
```

7.然后我们在`Beta` 命名空间中定义 `someFunction()`  。我们将其保存到一个单独的PHP文件中， `chap_03_developing_functions_namespace_beta.php` ：

```php
<?php
namespace Beta;

function someFunction()
{
  echo __NAMESPACE__ . ':' . __FUNCTION__ . PHP_EOL;
}
```

8.然后，我们可以在函数名前加上命名空间名来调用 `someFunction()` ：

```php
include (__DIR__ . DIRECTORY_SEPARATOR 
         . 'chap_03_developing_functions_namespace_alpha.php');
include (__DIR__ . DIRECTORY_SEPARATOR 
         . 'chap_03_developing_functions_namespace_beta.php');
      echo Alpha\someFunction();
      echo Beta\someFunction();
```

> **小贴士**
>
> **最佳实践**
>
> 最好的做法是将函数库（还有类！）放在单独的文件中：每个命名空间一个文件，每个文件一个类或函数库。
>
> 可以在单个命名空间中定义许多类或函数库。 之所以要开发成单独的命名空间，是因为希望促进功能的逻辑分离。

## 如何运行...

最好的做法是将所有逻辑上相关的函数放在一个单独的PHP文件中。创建一个名为 `chap_03_developing_functions_library.php` 的文件，并将这些函数（前面已经介绍过）放在里面：

* `someName()`
* `someOtherName()`
* `someInfinite()`
* `someDirScan()`
* `someTypeHint()`

然后在使用这些函数的代码中包含这个文件。

```php
include (__DIR__ . DIRECTORY_SEPARATOR . 'chap_03_developing_functions_library.php');
```

要调用 `someName()` 函数，使用名称并提供参数。

```php
echo someName('TEST');   // 返回 "INIT and also TEST"
```

你可以使用一个或两个参数调用 `someOtherName()` 函数，如下。

```php
echo someOtherName(1);    // 返回  1
echo someOtherName(1, 1);   // 返回 2
```

 `SomeInfinite()` 函数接受无限（或可变）数量的参数。这里有几个调用这个函数的例子。

```php
echo someInfinite(1, 2, 3);
echo PHP_EOL;
echo someInfinite(22.22, 'A', ['a' => 1, 'b' => 2]);
```

输出结果是这样的：

![](../../.gitbook/assets/image%20%2820%29.png)

我们可以如下调用 `someInfinite()` ：

```php
foreach (someDirScan(__DIR__ . DIRECTORY_SEPARATOR . '..') as $item) {
    echo $item . PHP_EOL;
}
```

输出结果如下：

![](../../.gitbook/assets/image%20%2814%29.png)

