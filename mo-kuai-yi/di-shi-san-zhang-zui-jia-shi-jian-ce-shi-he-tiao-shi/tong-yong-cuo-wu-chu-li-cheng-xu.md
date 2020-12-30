# 通用错误处理程序

开发通用错误处理程序的过程与前面的配方非常相似。但是有一些不同之处。首先，在 PHP 7 中，有些错误是被抛出的，并且可以被捕获，而其他错误则会让你的应用程序停止运行。更加让人困惑的是，有些错误被当作异常处理，而其他错误则是由新的 PHP 7 `Error` 类衍生出来的。幸运的是，在 PHP 7 中，`Error` 和 `Exception` 都实现了一个叫做 `Throwable` 的新接口。因此，如果你不确定你的代码会抛出`Exception`还是`Error`，只需捕获一个`Throwable`的实例，就可以同时捕获这两种情况。

## 如何做...

1.修改前面事例中定义的`Application\Error\Handler`类。在构造函数中，设置一个新的`errorHandler()`方法作为默认的错误处理程序。

```php
public function __construct($logFileDir = NULL, $logFile = NULL)
{
  $logFile    = $logFile    ?? date('Ymd') . '.log';
  $logFileDir = $logFileDir ?? __DIR__;
  $this->logFile = $logFileDir . '/' . $logFile;
  $this->logFile = str_replace('//', '/', $this->logFile);
  set_exception_handler([$this,'exceptionHandler']);
  set_error_handler([$this, 'errorHandler']);
}
```

2. 然后我们使用文档中的参数定义新方法。就像我们的异常处理程序一样，我们将信息记录到一个日志文件中。

```php
public function errorHandler($errno, $errstr, $errfile, $errline)
{
  $message = sprintf('ERROR: %s : %d : %s : %s : %s' . PHP_EOL,
    date('Y-m-d H:i:s'), $errno, $errstr, $errfile, $errline);
  file_put_contents($this->logFile, $message, FILE_APPEND);
}
```

3. 另外，为了能够区分错误和异常，在`exceptionHandler()`方法中向日志文件发送的消息中加入`exception`。

```php
public function exceptionHandler($ex)
{
  $message = sprintf('EXCEPTION: %19s : %20s : %s' . PHP_EOL,
    date('Y-m-d H:i:s'), get_class($ex), $ex->getMessage());
  file_put_contents($this->logFile, $message, FILE_APPEND);
}
```

## 如何运行...

首先，对前面定义的`Application\Error\Handler`进行修改。接下来，创建一个抛出错误的类，在这个例子中，可以定义为`Application\Error\ThrowsError`。例如，你可以有一个方法试图进行除以零的操作，另一个方法试图使用`eval()`来解析非PHP代码。

```php
<?php
namespace Application\Error;
class ThrowsError
{
  const NOT_PARSE = 'this will not parse';
  public function divideByZero()
  {
    $this->zero = 1 / 0;
  }
  public function willNotParse()
  {
    eval(self::NOT_PARSE);
  }
}
```

然后你可以定义一个名为`chap_13_error_throwable.php`的调用程序，设置自动加载，使用相应的类，并创建`ThrowsError`的实例。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Error\ { Handler, ThrowsError };
$error = new ThrowsError();
```

如果你再调用这两个方法，没有`try/catch`块，也没有定义通用错误处理程序，那么第一个方法会产生一个`Warning`，而第二个方法会抛出一个`ParseError`。

```php
$error->divideByZero();
$error->willNotParse();
echo 'Application continues ... ' . PHP_EOL;
```

因为这是一个错误，程序执行停止，你不会看到应用程序继续。

![](../../.gitbook/assets/image%20%28162%29.png)

如果将方法调用包裹在`try/catch`块中，并抓取`Throwable`，则代码会继续执行。

```php
try {
    $error->divideByZero();
} catch (Throwable $e) {
    echo 'Error Caught: ' . get_class($e) . ':' 
      . $e->getMessage() . PHP_EOL;
}
try {
    $error->willNotParse();
} catch (Throwable $e) {
    echo 'Error Caught: ' . get_class($e) . ':' 
    . $e->getMessage() . PHP_EOL;
}
echo 'Application continues ... ' . PHP_EOL;
```

从下面的输出中，你还会注意到，程序退出时，代码为0，这说明一切正常。

![](../../.gitbook/assets/image%20%28177%29.png)

最后，在`try/catch`块之后，再次运行错误，将echo语句移到最后。你将在输出中看到错误被捕获，但在日志文件中，注意到`DivisionByZeroError`是由异常处理程序捕获的，而`ParseError`是由错误处理程序捕获的。

```php
$handler = new Handler(__DIR__ . '/logs');
$error->divideByZero();
$error->willNotParse();
echo 'Application continues ... ' . PHP_EOL;
```

![](../../.gitbook/assets/image%20%28152%29.png)

## 更多...

PHP 7.1 允许在 `catch()`子句中指定多个类，因此，可以用 `catch (Exception | Error $e) { xxx }`代替单个 `Throwable`。

