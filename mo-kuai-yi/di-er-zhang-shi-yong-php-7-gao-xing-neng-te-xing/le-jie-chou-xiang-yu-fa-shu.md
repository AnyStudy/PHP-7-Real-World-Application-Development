# 了解抽象语法树

作为一名开发人员，不受 PHP 5和更早版本中强加的某些语法限制可能会让您感兴趣。除了前面提到的语法的一致性之外，语法方面的最大改进是调用任何返回值的能力，只需添加一组括号即可调用。此外，当返回值为数组时，您还可以直接访问任何数组元素。

## 怎么做...

1.任何返回回调的函数或方法都可以通过简单地添加括号 `()` 来立即执行\(有或没有参数\)。通过简单地使用方括号 `[]`;来表示元素，可以立即从任何返回数组的函数或方法中导出元素。在下面这个简短（但琐碎）的例子中，函数 `test()` 返回一个数组。数组包含六个匿名函数。`$a` 的值为 `$t` 。`$$a` 被解释为 `$test`：

```php
function test()
{
    return [
        1 => function () { return [
            1 => function ($a) { return 'Level 1/1:' . ++$a; },
            2 => function ($a) { return 'Level 1/2:' . ++$a; },
        ];},
        2 => function () { return [
            1 => function ($a) { return 'Level 2/1:' . ++$a; },
            2 => function ($a) { return 'Level 2/2:' . ++$a; },
        ];}
    ];
}

$a = 't';
$t = 'test';
echo $$a()[1]()[2](100);
```

2.AST允许我们发出 `echo $$a()[1]()[2](100)` 这样的命令。这是由左到右的解析，其执行方式如下：

* `$$a()` 解释为 `test()`，它返回一个数组
* `[1]` 派生数组元素 `1`，返回回调
* `()` 执行这个回调，它返回一个由两个元素组成的数组
* `[2]` 派生数组元素 `2` ，返回回调
* `(100)` 执行这个回调，提供一个 `100` 的值，返回 `Level 1/2:101`。

> **小贴士**
>
> 这样的语句在 PHP 5 中是不可能的：会返回一个解析错误。

3.下面是一个更实质性的示例，该示例利用AST语法定义数据过滤和验证类。 首先，我们定义`Application\Web\Securityclass`。 在构造函数中，我们构建并定义两个数组。 第一个数组由过滤器回调组成。 第二个数组具有验证回调：

```php
public function __construct()
  {
    $this->filter = [
      'striptags' => function ($a) { return strip_tags($a); },
      'digits'    => function ($a) { return preg_replace(
      '/[^0-9]/', '', $a); },
      'alpha'     => function ($a) { return preg_replace(
      '/[^A-Z]/i', '', $a); }
    ];
    $this->validate = [
      'alnum'  => function ($a) { return ctype_alnum($a); },
      'digits' => function ($a) { return ctype_digit($a); },
      'alpha'  => function ($a) { return ctype_alpha($a); }
    ];
  }
```

4.我们希望能够以一种对开发者友好的方式来调用这个功能。因此，如果我们想过滤数字，那么最好是运行这样的命令：

```php
$security->filterDigits($item));
```

为此，我们定义了魔术方法 `__call()`，它使我们可以访问不存在的方法：

```php
public function __call($method, $params)
{

  preg_match('/^(filter|validate)(.*?)$/i', $method, $matches);
  $prefix   = $matches[1] ?? '';
  $function = strtolower($matches[2] ?? '');
  if ($prefix && $function) {
    return $this->$prefix[$function]($params[0]);
  }
  return $value;
}
```

我们使用 `preg_match()` 将 `$method` 参数与 `filter` 或 `validate` 进行匹配。 然后，第二个子匹配项将在 `$this->filter` 或 `$this-> validate` 中转换为数组键。 如果两个子模式都产生子匹配项，则将第一个子匹配项分配给 `$prefix`，将第二个子匹配项 `$function` 分配给 `$prefix`。 当执行适当的回调时，这些最终将作为变量参数。

> **小贴士**
>
> **不要对这些东西太着迷了！**
>
> 当你沉浸在AST所带来的新的表达自由中时，一定要记住，你最终写的代码，从长远来看，可能是非常“神秘“的，这最终会造成长期的维护问题。

## 如何运行...

首先，我们创建一个示例文件 `chap_02_web_filtering_ast_example.php`，利用第一章 "建立基础 "中定义的自动加载类，获取 `Application\Web\Security` 的实例：

```php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
$security = new Application\Web\Security();
```

接下来，我们定义一个测试数据块：

```php
$data = [
    '<ul><li>Lots</li><li>of</li><li>Tags</li></ul>',
    12345,
    'This is a string',
    'String with number 12345',
];
```

最后，我们为每个测试数据项调用每个过滤器和验证器：

```php
foreach ($data as $item) {
  echo 'ORIGINAL: ' . $item . PHP_EOL;
  echo 'FILTERING' . PHP_EOL;
  printf('%12s : %s' . PHP_EOL,'Strip Tags', $security->filterStripTags($item));
  printf('%12s : %s' . PHP_EOL, 'Digits', $security->filterDigits($item));
  printf('%12s : %s' . PHP_EOL, 'Alpha', $security->filterAlpha($item));
    
  echo 'VALIDATORS' . PHP_EOL;
  printf('%12s : %s' . PHP_EOL, 'Alnum',  
  ($security->validateAlnum($item))  ? 'T' : 'F');
  printf('%12s : %s' . PHP_EOL, 'Digits', 
  ($security->validateDigits($item)) ? 'T' : 'F');
  printf('%12s : %s' . PHP_EOL, 'Alpha',  
  ($security->validateAlpha($item))  ? 'T' : 'F');
}
```

下面是一些输入字符串的输出：

![](../../.gitbook/assets/image%20%2811%29.png)

## 参考

关于AST的更多信息，请查阅针对抽象语法树的RFC，可以在[https://wiki.php.net/rfc/abstract\_syntax\_tree](https://wiki.php.net/rfc/abstract_syntax_tree) 上看到。

