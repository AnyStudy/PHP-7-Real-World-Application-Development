# 实现类的自动加载

当使用`面向对象编程`（`OOP`）方法开发PHP时，建议将每个类放在自己的文件中。遵循这个建议的好处是便于长期维护和提高可读性。缺点是每个类的定义文件必须被包含（即使用 `include` 或其变体）。为了解决这个问题，PHP语言中内置了一个机制，它将自动加载任何没有被特别包含的类。

## 做好准备

PHP 自动加载的最低要求是定义一个全局 **\_\_autoload\(\)** 函数。这是一个神奇的函数，当一个类被请求时，PHP引擎会自动调用，但这个类还没有被包含**。**当调用 **\_\_autoload\(\)** 函数时，被请求的类的名称将作为一个参数出现（假设你已经定义了它！）。如果你使用的是 PHP 命名空间，那么类的完整命名空间名称将被传递。因为 **\_\_autoload\(\)** 是一个函数，它必须在全局命名空间中；然而，它的使用是有限制的。因此，在这个配方中，我们将使用**spl\_autoload\_register\(\)** 函数，它给了我们更大的灵活性。

## 怎么做...

1.我们将在本示例介绍的类是 `Application \Autoload \Loader`。 为了利用 PHP 名称空间和自动加载之间的关系，我们将文件命名为 `Loader.php` 并将其放置在 `/path/to/cookbook/files/Application/Autoload`文件夹中。

2.我们将介绍的第一个方法只是简单地加载一个文件。在运行 `require_once()` 之前，我们使用 `file_exists()` 来检查。这样做的原因是，如果没有找到文件，`require_once()` 会产生一个致命的错误，而这个错误是不能用 PHP 7 新的错误处理功能来捕获：

```php
protected static function loadFile($file)
{
    if (file_exists($file)) {
        require_once $file;
        return TRUE;
    }
    return FALSE;
}
```

3.然后，我们可以在调用程序中测试 `loadFile()` 的返回值，如果最终无法加载文件，则在抛出 `Exception` 之前，循环查看备用目录列表。

> **小贴士**
>
> 您会注意到，此类中的方法和属性是静态的。 这在注册自动加载方法时为我们提供了更大的灵活性，并且还使我们像对待 `Singleton` 一样对待 `Loader` 类。

4.接下来，我们定义调用 `loadFile()` 的方法，并根据命名空间的类名实际执行逻辑来定位文件。 该方法通过将PHP名称空间分隔符 `\` 转换为适合此服务器的目录分隔符并附加 `.php` 来获取文件名：

```php
public static function autoLoad($class)
{
    $success = FALSE;
    $fn = str_replace('\\', DIRECTORY_SEPARATOR, $class) 
          . '.php';
    foreach (self::$dirs as $start) {
        $file = $start . DIRECTORY_SEPARATOR . $fn;
        if (self::loadFile($file)) {
            $success = TRUE;
            break;
        }
    }
    if (!$success) {
        if (!self::loadFile(__DIR__ 
            . DIRECTORY_SEPARATOR . $fn)) {
            throw new \Exception(
                self::UNABLE_TO_LOAD . ' ' . $class);
        }
    }
    return $success;
}
```

5.接下来，该方法遍历目录数组，我们称之为 `self::$dirs`，使用每个目录作为派生文件名的起点。 如果未成功，则最后一种方法是，该方法尝试从当前目录加载文件。 如果那还不成功，则抛出异常。

6.接下来，我们需要一个方法，可以将更多的目录添加到我们的目录列表中进行测试。请注意，如果提供的值是一个数组，则使用 `array_merge()` 。否则，我们只需将目录字符串添加到 `self::$dirs` 数组中：

```php
public static function addDirs($dirs)
{
    if (is_array($dirs)) {
        self::$dirs = array_merge(self::$dirs, $dirs);
    } else {
        self::$dirs[] = $dirs;
    }
}  
```

7.然后，我们来到最重要的部分；我们需要将我们的 `autoload()` 方法注册为**标准PHP库**（**SPL**）的自动加载器。这可以通过使用 `spl_autoload_register()`  和 `init()` 方法来完成：

```php
public static function init($dirs = array())
{
    if ($dirs) {
        self::addDirs($dirs);
    }
    if (self::$registered == 0) {
        spl_autoload_register(__CLASS__ . '::autoload');
        self::$registered++;
    }
}
```

8.此时，我们可以定义 `__construct()` ，它调用 `self::init($dirs)` 。这允许我们在需要的时候创建一个 `Loader` 的实例：

```php
public function __construct($dirs = array())
{
    self::init($dirs);
}
```

## 它是如何运行的...

为了使用我们刚刚定义的 autoloader 类，您将需要 `require Loader.php` 。 如果名称空间文件位于当前目录以外的目录中，则还应该运行 `Loader :: init()` 并提供其他目录路径。

为了确保自动加载器工作，我们还需要一个测试类。下面是`/path/to/cookbook/files/Application/Test/TestClass.php` 的定义：

```php
<?php
namespace Application\Test;
class TestClass
{
    public function getTest()
    {
        return __METHOD__;
    }
}
```

现在创建一个 `chap_01_autoload_test.php` 代码文件以测试自动加载器：

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
```

接下来，获取尚未加载的类的实例：

```php
$test = new Application\Test\TestClass();
echo $test->getTest();
```

最后，尝试获得一个不存在的 `fake` 类。 请注意，这将引发错误：

```php
$fake = new Application\Test\FakeClass();
echo $fake->getTest();
```

