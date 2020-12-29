# 通用异常处理程序

当异常与`try/catch`块中的代码结合使用时，异常特别有用。然而，在某些情况下，使用这种结构可能会很笨拙，使代码几乎不可读。另一个考虑因素是，许多类最终会抛出你没有预料到的异常。在这种情况下，有某种回退异常处理程序是非常可取的。

## 如何做...

1.首先，我们定义一个通用的异常处理类，`Application\Error\Handler`。

```php
namespace Application\Error;
class Handler
{
  // code goes here
}
```

2. 我们定义了代表一个日志文件的属性。如果没有提供名称，则以年、月、日命名。在构造函数中，我们使用`set_exception_handler()`来分配`exceptionHandler()`方法（在这个类中）作为后备处理程序。

```php
protected $logFile;
public function __construct(
  $logFileDir = NULL, $logFile = NULL)
{
  $logFile = $logFile    ?? date('Ymd') . '.log';
  $logFileDir = $logFileDir ?? __DIR__;
  $this->logFile = $logFileDir . '/' . $logFile;
  $this->logFile = str_replace('//', '/', $this->logFile);
  set_exception_handler([$this,'exceptionHandler']);
}
```

3. 接下来，我们定义`exceptionHandler()`方法，它以一个`Exception`对象作为参数。我们在日志文件中记录了日期和时间、异常的类名以及它的消息。

```php
public function exceptionHandler($ex)
{
  $message = sprintf('%19s : %20s : %s' . PHP_EOL,
    date('Y-m-d H:i:s'), get_class($ex), $ex->getMessage());
  file_put_contents($this->logFile, $message, FILE_APPEND); 
}
```

4. 如果我们在代码中专门放置了`try/catch`块，这将覆盖我们的通用异常处理程序。另一方面，如果我们不使用`try/catch`，而发生了异常，通用异常处理程序就会发挥作用。

{% hint style="info" %}
**最佳实践**

你应该总是使用try/catch来捕获异常，并可能继续你的应用程序。这里描述的异常处理程序只是为了让您的应用程序在抛出的异常没有被捕获的情况下 "优雅地 "结束。
{% endhint %}

## 如何运行...

首先，将前面配方中的代码放入`Application\Error`文件夹中的`Handler.php`文件。接下来，定义一个测试类来抛出异常。为了说明问题，创建一个`Application\Error\ThrowsException`类来抛出一个异常。作为一个例子，设置一个PDO实例，错误模式设置为`PDO::ERRMODE_EXCEPTION`。然后，你可以制作一个保证失败的SQL语句。

```php
namespace Application\Error;
use PDO;
class ThrowsException
{
  protected $result;
  public function __construct(array $config)
  {
    $dsn = $config['driver'] . ':';
    unset($config['driver']);
    foreach ($config as $key => $value) {
      $dsn .= $key . '=' . $value . ';';
    }
    $pdo = new PDO(
      $dsn, 
      $config['user'],
      $config['password'],
      [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]);
      $stmt = $pdo->query('This Is Not SQL');
      while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
        $this->result[] = $row;
      }
  }
}
```

接下来，定义一个名为`chap_13_exception_handler.php`的调用程序，设置自动加载，使用相应的类。

```php
<?php
define('DB_CONFIG_FILE', __DIR__ . '/../config/db.config.php');
$config = include DB_CONFIG_FILE;
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Error\ { Handler, ThrowsException };
```

此时，如果你在没有实现通用处理程序的情况下创建一个`ThrowsException`实例，就会产生一个`Fatal Error`，因为一个异常已经被抛出但没有被捕获。

```php
$throws1 = new ThrowsException($config);
```

![](../../.gitbook/assets/image%20%28156%29.png)

另一方面，如果你使用`try/catch`块，如果你的应用程序足够稳定，那么异常就会被捕获并允许继续。

```php
try {
    $throws1 = new ThrowsException($config);
} catch (Exception $e) {
    echo 'Exception Caught: ' . get_class($e) . ':' . $e->getMessage() . PHP_EOL;
}
echo 'Application Continues ...' . PHP_EOL;
```

您将看到以下输出。

![](../../.gitbook/assets/image%20%28165%29.png)

为了演示异常处理程序的使用，在`try/catch` 块之前，定义一个 `Handler` 实例，传递一个代表包含日志文件的目录的参数。在 `try/catch` 块之后，在该块之外，创建另一个 `ThrowsException` 实例。当你运行这个示例程序时，你会注意到第一个异常是在`try/catch`块内捕获的，而第二个异常是由处理程序捕获的。你还会注意到，在处理程序之后，应用程序结束。

```php
$handler = new Handler(__DIR__ . '/logs');
try {
    $throws1 = new ThrowsException($config);
} catch (Exception $e) {
    echo 'Exception Caught: ' . get_class($e) . ':' 
      . $e->getMessage() . PHP_EOL;
}
$throws1 = new ThrowsException($config);
echo 'Application Continues ...' . PHP_EOL;
```

下面是完成的示例程序的输出，以及日志文件的内容。

![](../../.gitbook/assets/image%20%28177%29.png)

## 更多...

也许你可以回顾一下`set_exception_handler()`函数的文档。特别是看一下匿名者的评论（7年前发布的，但仍然是相关的），它澄清了这个函数的工作原理：http://php.net/manual/en/function.set-exception-handler.php。

