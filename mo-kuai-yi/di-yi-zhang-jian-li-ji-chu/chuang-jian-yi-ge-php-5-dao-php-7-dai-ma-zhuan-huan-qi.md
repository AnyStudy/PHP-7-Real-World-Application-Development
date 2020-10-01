# 创建一个 PHP 5 到 PHP 7 代码转换器

在大多数情况下，PHP 5. x 代码可以不加修改地在 PHP 7上运行。然而，有一些变化被归类为向后不兼容。这意味着，如果你的 PHP 5代码是以某种方式编写的，或者使用已经被删除的函数，你的代码就会中断，你就会得到一个严重的错误。

### 做好准备

PHP 5到PHP 7代码转换器有两个作用：

* 扫描你的代码文件，并将PHP 5中已被删除的功能转换为PHP 7中的相应功能。
* 在语言用法发生变化但无法重写的情况下，添加带//警告的注释。

> **小贴士**
>
> 请注意，在运行转换器后，你的代码不能保证在 PHP 7 中工作。你仍然需要检查添加的 `// WARNING` 标签。至少，这个配方将给你一个良好的开端，将你的 PHP 5 代码转换为在 PHP 7 中工作。

这个配方的核心是 PHP 7 的新函数 `preg_replace_callback_array()` 。这个神奇的函数允许你做的是将一个正则表达式数组作为键，其值代表一个独立的回调。然后，你可以将字符串通过一系列的转换。不仅如此，回调数组本身也可以是一个数组。

## 怎么做...

1.在一个新的 `Application\Parse\Convert` 类中，我们首先使用 `scan()` 方法，它接受一个文件名作为参数。它检查文件是否存在。如果存在，它调用PHP `file()` 函数，将文件加载到一个数组中，每个数组元素代表一行。

```php
public function scan($filename)
{
    if (!file_exists($filename)) {
        throw new Exception(
            self::EXCEPTION_FILE_NOT_EXISTS);
    }
    $contents = file($filename);
    echo 'Processing: ' . $filename . PHP_EOL;
    
    $result = preg_replace_callback_array( [
```

2.接下来，我们开始传递一系列的键/值对。键是一个正则表达式，它将针对字符串进行处理。任何匹配的结果都会被传递给回调，回调的值是键/值对的值部分。我们检查PHP 7中已被删除的打开和关闭标签：

```php
    // 替换不再支持的开始标签
    '!^\<\%(\n| )!' =>
        function ($match) {
            return '<?php' . $match[1];
        },

    // 替换不再支持的开始标签
    '!^\<\%=(\n| )!' =>
        function ($match) {
            return '<?php echo ' . $match[1];
        },

    // 替换不再支持的结束标签
    '!\%\>!' =>
        function ($match) {
            return '?>';
        },
```

3.接下来是一系列的警告，当检测到某些操作，并且在PHP 5和PHP 7中处理这些操作的方式之间存在潜在的代码断裂。在所有这些情况下，代码不会被重新编写。取而代之的是一个带有 `WARNING` 字样的内联注释：

```php
    // 改变 $xxx 解释的处理方式
    '!(.*?)\$\$!' =>
        function ($match) {
            return '// WARNING: variable interpolation 
                   . ' now occurs left-to-right' . PHP_EOL
                   . '// see: http://php.net/manual/en/'
                   . '// migration70.incompatible.php'
                   . $match[0];
        },

    // list()运算符处理方式的更改
    '!(.*?)list(\s*?)?\(!' =>
        function ($match) {
            return '// WARNING: changes have been made '
                   . 'in list() operator handling.'
                   . 'See: http://php.net/manual/en/'
                   . 'migration70.incompatible.php'
                   . $match[0];
        },

    // 实例 \u{
    '!(.*?)\\\u\{!' =>
        function ($match) {
        return '// WARNING: \\u{xxx} is now considered '
               . 'unicode escape syntax' . PHP_EOL
               . '// see: http://php.net/manual/en/'
               . 'migration70.new-features.php'
               . '#migration70.new-features.unicode-'
               . 'codepoint-escape-syntax' . PHP_EOL
               . $match[0];
    },

    // 依赖于 set_error_handler()
    '!(.*?)set_error_handler(\s*?)?.*\(!' =>
        function ($match) {
            return '// WARNING: might not '
                   . 'catch all errors'
                   . '// see: http://php.net/manual/en/'
                   . '// language.errors.php7.php'
                   . $match[0];
        },

    // session_set_save_handler(xxx)
    '!(.*?)session_set_save_handler(\s*?)?\((.*?)\)!' =>
        function ($match) {
            if (isset($match[3])) {
                return '// WARNING: a bug introduced in'
                       . 'PHP 5.4 which '
                       . 'affects the handler assigned by '
                       . 'session_set_save_handler() and '
                       . 'where ignore_user_abort() is TRUE 
                       . 'has been fixed in PHP 7.'
                       . 'This could potentially break '
                       . 'your code under '
                       . 'certain circumstances.' . PHP_EOL
                       . 'See: http://php.net/manual/en/'
                       . 'migration70.incompatible.php'
                       . $match[0];
            } else {
                return $match[0];
            }
        },

```

4.任何试图使用 `<<` 或 `>>` 与负运算符，或超过64的尝试，都会被包装在 `try { xxx } catch() { xxx }` 块中，寻找一个 `ArithmeticError` 并被抛出。

```php
    // 在 try/catch 中包装 bit shift 操作
    '!^(.*?)(\d+\s*(\<\<|\>\>)\s*-?\d+)(.*?)$!' =>
        function ($match) {
            return '// WARNING: negative and '
                   . 'out-of-range bitwise '
                   . 'shift operations will now 
                   . 'throw an ArithmeticError' . PHP_EOL
                   . 'See: http://php.net/manual/en/'
                   . 'migration70.incompatible.php'
                   . 'try {' . PHP_EOL
                   . "\t" . $match[0] . PHP_EOL
                   . '} catch (\\ArithmeticError $e) {'
                   . "\t" . 'error_log("File:" 
                   . $e->getFile() 
                   . " Message:" . $e->getMessage());'
                   . '}' . PHP_EOL;
        },
```

> **小贴士**
>
> PHP 7 改变了错误的处理方式。在某些情况下，错误被移到了类似于异常的分类中，并且可以被捕获!`Error` 和 `Exception` 类都实现了 `Throwable` 接口。如果你想捕获 `Error` 或 `Exception`，请捕获 `Throwable`。

5.接下来，转换器重写了在PHP 7中被删除的 `call_user_method*()` 的所有用法。这些都被等同的`call_user_func*()` 所取代。

```php
    // 将“call_user_method()”替换为
    // “call_user_func()”
    '!call_user_method\((.*?),(.*?)(,.*?)\)(\b|;)!' =>
        function ($match) {
            $params = $match[3] ?? '';
            return '// WARNING: call_user_method() has '
                      . 'been removed from PHP 7' . PHP_EOL
                      . 'call_user_func(['. trim($match[2]) . ',' 
                      . trim($match[1]) . ']' . $params . ');';
        },

    // 将“call_user_method_array()”
    // 替换为 “call_user_func_array()”
    '!call_user_method_array\((.*?),(.*?),(.*?)\)(\b|;)!' =>
        function ($match) {
            return '// WARNING: call_user_method_array()'
                   . 'has been removed from PHP 7'
                   . PHP_EOL
                   . 'call_user_func_array([' 
                   . trim($match[2]) . ',' 
                   . trim($match[1]) . '], ' 
                   . $match[3] . ');';
        },
```

6.最后，任何尝试使用 `/e` 修饰符的 `preg_replace()` 都会使用 `preg_replace_callback()` 重写：

```php
     '!^(.*?)preg_replace.*?/e(.*?)$!' =>
    function ($match) {
        $last = strrchr($match[2], ',');
        $arg2 = substr($match[2], 2, -1 * (strlen($last)));
        $arg1 = substr($match[0], 
                       strlen($match[1]) + 12, 
                       -1 * (strlen($arg2) + strlen($last)));
         $arg1 = trim($arg1, '(');
         $arg1 = str_replace('/e', '/', $arg1);
         $arg3 = '// WARNING: preg_replace() "/e" modifier 
                   . 'has been removed from PHP 7'
                   . PHP_EOL
                   . $match[1]
                   . 'preg_replace_callback('
                   . $arg1
                   . 'function ($m) { return ' 
                   .    str_replace('$1','$m', $match[1]) 
                   .      trim($arg2, '"\'') . '; }, '
                   .      trim($last, ',');
         return str_replace('$1', '$m', $arg3);
    },

        // 数组结
        ],

        // 这就是转变的目标
        $contents
    );
    // 以字符串形式返回结果
    return implode('', $result);
}
```

## 如何运行...

若要使用转换器，请从命令行运行以下代码。您需要提供 PHP 5代码的文件名作为参数进行扫描。

这段代码，`chap_01_php5_to_php7_code_converter.php`，从命令行运行，调用转换器：

```php
<?php
// 从命令行获取要扫描的文件名
$filename = $argv[1] ?? '';

if (!$filename) {
    echo 'No filename provided' . PHP_EOL;
    echo 'Usage: ' . PHP_EOL;
    echo __FILE__ . ' <filename>' . PHP_EOL;
    exit;
}

// 设置类自动加载
require __DIR__ . '/../Application/Autoload/Loader.php';

// 在路径中添加工作目录
Application\Autoload\Loader::init(__DIR__ . '/..');

// 获取深度扫描类
$convert = new Application\Parse\Convert();
echo $convert->scan($filename);
echo PHP_EOL;
```

## 参考

有关向后不兼容的更改的更多信息，请参考 [http://php.net/manual/en/migration70.incompatible.php](http://php.net/manual/en/migration70.incompatible.php)。

