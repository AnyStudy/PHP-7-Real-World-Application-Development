# 使用session\_start参数自定义会话

在 PHP 7 之前，为了覆盖 `php.ini` 的安全会话管理设置，你必须使用一系列 `ini_set()` 命令。这种方法非常烦人，因为你需要知道哪些设置是可用的，而且很难在其他应用程序中重复使用相同的设置。然而，从 PHP 7 开始，你可以向 `session_start()` 命令提供一个参数数组，它可以立即设置这些值。

## 如何做...

1.我们首先开发一个`Application\Security\SessOptions`类，该类将持有会话参数，并有能力启动会话。我们还定义了一个类常量，以防传递无效的会话选项。

```php
namespace Application\Security;
use ReflectionClass;
use InvalidArgumentsException;
class SessOptions
{
  const ERROR_PARAMS = 'ERROR: invalid session options';
```

2. 接下来，我们扫描`php.ini`会话指令列表\(在http://php.net/manual/en/session.configuration.php\)。我们特别要寻找那些在`Changeable`列中标记为`PHP_INI_ALL`的指令。这些指令可以在运行时被覆盖，因此可以作为 `session_start()` 的参数。

![](../../.gitbook/assets/image%20%28160%29.png)

3. 然后，我们将这些定义为类常量，这将使这个类在开发时更加可用。大多数像样的代码编辑器都能够扫描该类，并给出一个常量列表，从而使管理会话设置变得容易。请注意，为了节省本书的空间，并没有显示所有的设置。

```php
const SESS_OP_NAME         = 'name';
const SESS_OP_LAZY_WRITE   = 'lazy_write';  // AVAILABLE // SINCE PHP 7.0.0.
const SESS_OP_SAVE_PATH    = 'save_path';
const SESS_OP_SAVE_HANDLER = 'save_handler';
// etc.
```

4. 然后我们就可以定义构造函数，它接受一个包含`php.ini`会话设置的数组作为参数。我们使用`ReflectionClass`来获取一个类常量列表，并通过循环运行`$options`参数来确认设置是否被允许。同时注意到`array_flip()`的使用，它可以翻转键和值，这样我们的类常量的实际值就形成了数组键，而类常量的名字就变成了值。

```php
protected $options;
protected $allowed;
public function __construct(array $options)
{
  $reflect = new ReflectionClass(get_class($this));
  $this->allowed = $reflect->getConstants();
  $this->allowed = array_flip($this->allowed);
  unset($this->allowed[self::ERROR_PARAMS]);
  foreach ($options as $key => $value) {
    if(!isset($this->allowed[$key])) {
      error_log(__METHOD__ . ':' . self::ERROR_PARAMS);
      throw new InvalidArgumentsException(
      self::ERROR_PARAMS);
    }
  }
  $this->options = $options;
}
```

5. 然后，我们再用两个方法来结束；一个方法让我们可以访问外部允许的参数，而另一个方法则启动会话。

```php
public function getAllowed()
{
  return $this->allowed;
}

public function start()
{
  session_start($this->options);
}
```

## 如何运行...

将本事例中讨论的所有代码放入`Application\Security`目录下的`SessOptions.php`文件中。然后你可以定义一个名为`chap_13_session_options.php`的调用程序来测试新的类，它设置了自动加载并使用了该类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Security\SessOptions;
```

接下来，定义一个数组，使用类常量作为键，其值根据需要来管理会话。注意，在这里显示的例子中，会话信息存储在一个子目录`session`中，你需要创建这个子目录。

```php
$options = [
  SessOptions::SESS_OP_USE_ONLY_COOKIES => 1,
  SessOptions::SESS_OP_COOKIE_LIFETIME => 300,
  SessOptions::SESS_OP_COOKIE_HTTPONLY => 1,
  SessOptions::SESS_OP_NAME => 'UNLIKELYSOURCE',
  SessOptions::SESS_OP_SAVE_PATH => __DIR__ . '/session'
];
```

现在你可以创建`SessOptions`实例并运行`start()`来启动会话。你可以在这里使用`phpinfo()`来显示会话的一些信息。

```php
$sessOpt = new SessOptions($options);
$sessOpt->start();
$_SESSION['test'] = 'TEST';
phpinfo(INFO_VARIABLES);
```

如果你用浏览器的开发工具查找cookie的信息，你会注意到名称被设置为`UNLIKELYSOURCE`，到期时间是5分钟以后。

![](../../.gitbook/assets/image%20%28178%29.png)

如果你对会话目录进行扫描，你会看到会话信息已经存储在那里。

![](../../.gitbook/assets/image%20%28176%29.png)

## 更多...

关于更多与session相关的php.ini指令的信息，请参见这个摘要：http://php.net/manual/en/session.configuration.php。

