# 理解句法分析中的差异

在 PHP 5中，赋值操作右侧的表达式是从右向左解析的。在 PHP 7中，解析始终是从左到右的。

## 如何做...

1.变量-变量是间接引用值的一种方式。 在下面的示例中，第一个 `$$foo` 被解释为 `${$bar}` 。 因此，最终的返回值是 `$bar` 的值，而不是 `$foo` 的直接值（可能是 `bar`）：

```php
$foo = 'bar';
$bar = 'baz';
echo $$foo; // 返回  'baz'; 
```

2.在下一个示例中，我们有一个变量-变量 `$$foo` ，该变量引用带有 `bar`键和 `baz` 子键的多维数组：

```php
$foo = 'bar';
$bar = ['bar' => ['baz' => 'bat']];
// 返回 'bat'
echo $$foo['bar']['baz'];
```

3.在PHP 5中，解析是从右到左进行的，这意味着PHP引擎将寻找带有 `bar` 键和 `baz` 子键的 `$foo` 数组。 然后将解释该元素的返回值以获得最终值 `${$foo['bar']['baz']}`。

4.但是，在PHP 7中，解析始终是从左到右，这意味着 `$foo` 首先被解释 `($$foo)['bar']['baz']` 。

5.在下一个示例中，您可以看到与PHP 7相比，PHP 5中对 `$foo->$bar['bada']` 的解释完全不同。在下面的示例中，PHP 5将首先解释 `$bar['bada']` ，并针对 `$foo` 对象实例引用此返回值。 另一方面，在PHP 7中，解析始终是从左到右，这意味着 `$foo->$bar` 首先被解释，并且期望包含 `bada` 元素的数组。 您还将注意到，该示例使用了PHP 7匿名类功能：

```php
// PHP 5: $foo->{$bar['bada']}
// PHP 7: ($foo->$bar)['bada']
$bar = 'baz';
// $foo = new class 
{ 
    public $baz = ['bada' => 'boom']; 
};
// returns 'boom'
echo $foo->$bar['bada'];
```

6.最后一个示例与上面的示例相同，不同之处在于返回值是个回调，然后立即执行： 

```php
// PHP 5: $foo->{$bar['bada']}()
// PHP 7: ($foo->$bar)['bada']()
$bar = 'baz';
// 注意: 这个例子使用了新的 PHP 7匿名类特性
$foo = new class 
{ 
     public function __construct() 
    { 
        $this->baz = ['bada' => function () { return 'boom'; }]; 
    } 
};
// 返回 'boom'
echo $foo->$bar['bada']();
```

## 如何运行...

将1和2中所示的代码示例放在一个单独的PHP文件中，您可以命名为 `chap_02_understanding_diffs_in_parsing.php` 。 首先使用PHP 5执行该脚本，您将注意到一系列错误，如下所示：

![](../../.gitbook/assets/image%20%2824%29.png)

错误的原因是PHP 5解析不一致，并且就请求的变量-变量的状态得出了错误的结论（如前所述）。 现在，您可以继续添加其余示例，如步骤5和6所示。如果您随后在PHP 7中运行此脚本，则将显示所描述的结果，如下所示：

![](../../.gitbook/assets/image%20%2820%29.png)

## 参考

有关解析的更多信息，请参考 RFC，它提供了统一变量语法，并且可以在 [https://wiki.php.net/RFC/uniform\_variable\_syntax](https://wiki.php.net/RFC/uniform_variable_syntax) 查看。

