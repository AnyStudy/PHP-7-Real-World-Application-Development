# 进行框架间的系统调用

开发 PSR-7（和中间件）的主要原因之一是对框架之间调用的需求越来越大。值得注意的是，PSR-7 的主要文档由 **PHP Framework Interop Group** \(**PHP-FIG**\) 主持。

## 如何做...

1.在中间件框架间调用中使用的主要机制是创建一个驱动程序，连续执行框架调用，维护一个共同的请求和响应对象。请求和响应对象有望分别代表 `Psr\Http\Message\ServerRequestInterface` 和`Psr\Http\Message\ResponseInterface`。

2. 为了这个说明的目的，我们定义了一个中间件会话验证器。常量和属性反映了会话指纹，这是我们用来整合网站访问者的 IP 地址、浏览器和语言设置等因素的术语。

```php
namespace Application\MiddleWare\Session;
use InvalidArgumentException;
use Psr\Http\Message\ { 
  ServerRequestInterface, ResponseInterface };
use Application\MiddleWare\ { Constants, Response, TextStream };
class Validator
{
  const KEY_TEXT = 'text';
  const KEY_SESSION = 'thumbprint';
  const KEY_STATUS_CODE = 'code';
  const KEY_STATUS_REASON = 'reason';
  const KEY_STOP_TIME = 'stop_time';
  const ERROR_TIME = 'ERROR: session has exceeded stop time';
  const ERROR_SESSION = 'ERROR: thumbprint does not match';
  const SUCCESS_SESSION = 'SUCCESS: session validates OK';
  protected $sessionKey;
  protected $currentPrint;
  protected $storedPrint;
  protected $currentTime;
  protected $storedTime;
```

3. 构造函数把`ServerRequestInterface`实例和`session`作为参数。如果`session`是一个数组\(比如`$_SESSION`\)，我们就把它封装在一个类中。我们这样做的原因是为了防止传递给我们一个会话对象，比如Joomla中使用的`JSession`。然后，我们使用前面提到的因素创建拇指指纹。如果存储的拇指印不可用，我们就假设这是第一次，如果设置了这个参数，就存储当前打印以及停止时间。我们使用了`md5()`，因为它是一种快速的哈希值，不对外暴露，因此对这个应用很有用。

```php
public function __construct(
  ServerRequestInterface $request, $stopTime = NULL)
{
  $this->currentTime  = time();
  $this->storedTime   = $_SESSION[self::KEY_STOP_TIME] ?? 0;
  $this->currentPrint = 
    md5($request->getServerParams()['REMOTE_ADDR']
      . $request->getServerParams()['HTTP_USER_AGENT']
      . $request->getServerParams()['HTTP_ACCEPT_LANGUAGE']);
        $this->storedPrint  = $_SESSION[self::KEY_SESSION] 
      ?? NULL;
  if (empty($this->storedPrint)) {
      $this->storedPrint = $this->currentPrint;
      $_SESSION[self::KEY_SESSION] = $this->storedPrint;
      if ($stopTime) {
          $this->storedTime = $stopTime;
          $_SESSION[self::KEY_STOP_TIME] = $stopTime;
      }
  }
}
```

4. 虽然不需要定义`__invoke()`，但这个神奇的方法对于独立的中间件类来说相当方便。按照惯例，我们接受 `ServerRequestInterface` 和 `ResponseInterface` 实例作为参数。在这个方法中，我们只需检查当前的拇指印是否与存储的拇指印一致。第一次，当然，它们会匹配。但在随后的请求中，意图劫持会话的攻击者有可能会被发现。此外，如果会话时间超过了停止时间（如果设置了），同样，也会发送401代码。

```php
public function __invoke(
  ServerRequestInterface $request, Response $response)
{
  $code = 401;  // unauthorized
  if ($this->currentPrint != $this->storedPrint) {
      $text[self::KEY_TEXT] = self::ERROR_SESSION;
      $text[self::KEY_STATUS_REASON] = 
        Constants::STATUS_CODES[401];
  } elseif ($this->storedTime) {
      if ($this->currentTime > $this->storedTime) {
          $text[self::KEY_TEXT] = self::ERROR_TIME;
          $text[self::KEY_STATUS_REASON] = 
            Constants::STATUS_CODES[401];
      } else {
          $code = 200; // success
      }
  }
  if ($code == 200) {
      $text[self::KEY_TEXT] = self::SUCCESS_SESSION;
      $text[self::KEY_STATUS_REASON] = 
        Constants::STATUS_CODES[200];
  }
  $text[self::KEY_STATUS_CODE] = $code;
  $body = new TextStream(json_encode($text));
  return $response->withStatus($code)->withBody($body);
}
```

5. 现在我们可以把我们新的中间件类用起来了。这里总结了框架间调用的主要问题，至少在这一点上是这样。相应地，我们如何实现中间件，很大程度上取决于最后一点。

* 并非所有的PHP框架都符合PSR-7标准
* 现有的PSR-7实施工作不完整 
* 所有框架都想当 "老大"

6. 举个例子，看看 `Zend Expressive` 的配置文件，它是一个自称 PSR7 的中间件微框架。这里有一个文件， `middleware-pipeline.global.php`，它位于标准Expressive应用程序的`config/autoload`文件夹中。依赖关系键用于识别将在管道中激活的中间件包装类。

```php
<?php
use Zend\Expressive\Container\ApplicationFactory;
use Zend\Expressive\Helper;
return [  
  'dependencies' => [
     'factories' => [
        Helper\ServerUrlMiddleware::class => 
        Helper\ServerUrlMiddlewareFactory::class,
        Helper\UrlHelperMiddleware::class => 
        Helper\UrlHelperMiddlewareFactory::class,
        // insert your own class here
     ],
  ],
```

7. 在 `middleware_pipline` 键下，您可以确定将在路由过程发生之前或之后执行的类。可选参数包括路径、错误和优先级。

```php
'middleware_pipeline' => [
   'always' => [
      'middleware' => [
         Helper\ServerUrlMiddleware::class,
      ],
      'priority' => 10000,
   ],
   'routing' => [
      'middleware' => [
         ApplicationFactory::ROUTING_MIDDLEWARE,
         Helper\UrlHelperMiddleware::class,
         // insert reference to middleware here
         ApplicationFactory::DISPATCH_MIDDLEWARE,
      ],
      'priority' => 1,
   ],
   'error' => [
      'middleware' => [
         // Add error middleware here.
      ],
      'error'    => true,
      'priority' => -10000,
    ],
  ],
];
```

8. 另一种技术是修改现有框架模块的源代码，并向符合PSR-7的中间件应用程序提出请求。下面是一个修改Joomla！安装的例子，以包含一个中间件会话验证器。

9. 接下来，在 `/path/to/joomla` 文件夹的 `index.php` 文件末尾添加这段代码。由于Joomla！使用Composer，我们可以利用Composer的自动加载器。

```php
session_start();    // to support use of $_SESSION
$loader = include __DIR__ . '/libraries/vendor/autoload.php';
$loader->add('Application', __DIR__ . '/libraries/vendor');
$loader->add('Psr', __DIR__ . '/libraries/vendor');
```

10. 然后我们可以创建一个中间件会话验证器的实例，并在 `$app = JFactory::getApplication('site')` 之前发出验证请求。

```php
$session = JFactory::getSession();
$request = 
  (new Application\MiddleWare\ServerRequest())->initialize();
$response = new Application\MiddleWare\Response();
$validator = new Application\Security\Session\Validator(
  $request, $session);
$response = $validator($request, $response);
if ($response->getStatusCode() != 200) {
  // take some action
}
```

## 如何运行...

首先，创建步骤2-5中描述的 `Application\MiddleWare\Session\Validator` 测试中间件类。然后，你需要去 [https://getcomposer.org/](https://getcomposer.org/)，并按照指示获得Composer。将其下载到`/path/to/source/for/this/chapter`文件夹中。接下来，构建一个基本的Zend Expressive应用程序，如下图所示。当提示需要最小框架时，一定要选择No。

```bash
cd /path/to/source/for/this/chapter
php composer.phar create-project zendframework/zend-expressive-skeleton expressive
```

这将创建一个文件夹`/path/to/source/for/this/chapter/expressive`。改成这个目录。修改`public/index.php`如下。

```php
<?php
if (php_sapi_name() === 'cli-server'
    && is_file(__DIR__ . parse_url(
$_SERVER['REQUEST_URI'], PHP_URL_PATH))
) {
    return false;
}
chdir(dirname(__DIR__));
session_start();
$_SESSION['time'] = time();
$appDir = realpath(__DIR__ . '/../../..');
$loader = require 'vendor/autoload.php';
$loader->add('Application', $appDir);
$container = require 'config/container.php';
$app = $container->get(\Zend\Expressive\Application::class);
$app->run();
```

然后，需要创建一个包装器类来调用我们的会话验证器中间件。创建一个`SessionValidateAction.php`文件，需要放在`/path/to/source/for/this/chapter/expressive/src/App/Action`文件夹中。在本例中，将停止时间参数设置为一个较短的持续时间。在本例中，`time() + 10`得到10秒。

```php
namespace App\Action;
use Application\MiddleWare\Session\Validator;
use Zend\Diactoros\ { Request, Response };
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
class SessionValidateAction
{
  public function __invoke(ServerRequestInterface $request, 
  ResponseInterface $response, callable $next = null)
  {
    $inbound   = new Response();
    $validator = new Validator($request, time()+10);
    $inbound   = $validator($request, $response);
    if ($inbound->getStatusCode() != 200) {
        session_destroy();
        setcookie('PHPSESSID', 0, time()-300);
        $params = json_decode(
          $inbound->getBody()->getContents(), TRUE);
        echo '<h1>',$params[Validator::KEY_TEXT],'</h1>';
        echo '<pre>',var_dump($inbound),'</pre>';
        exit;
    }
    return $next($request,$response);
  }
}
```

现在需要将新类添加到中间件管道中。修改`config/autoload/middleware-pipeline.global.php`如下。修改的内容以粗体显示。

```php
<?php
use Zend\Expressive\Container\ApplicationFactory;
use Zend\Expressive\Helper;
return [
  'dependencies' => [
     'invokables' => [
        App\Action\SessionValidateAction::class => 
        App\Action\SessionValidateAction::class,
     ],
   'factories' => [
      Helper\ServerUrlMiddleware::class => 
      Helper\ServerUrlMiddlewareFactory::class,
      Helper\UrlHelperMiddleware::class => 
      Helper\UrlHelperMiddlewareFactory::class,
    ],
  ],
  'middleware_pipeline' => [
      'always' => [
         'middleware' => [
            Helper\ServerUrlMiddleware::class,
         ],
         'priority' => 10000,
      ],
      'routing' => [
         'middleware' => [
            ApplicationFactory::ROUTING_MIDDLEWARE,
            Helper\UrlHelperMiddleware::class,
            App\Action\SessionValidateAction::class,
            ApplicationFactory::DISPATCH_MIDDLEWARE,
         ],
         'priority' => 1,
      ],
    'error' => [
       'middleware' => [
          // Add error middleware here.
       ],
       'error'    => true,
       'priority' => -10000,
    ],
  ],
];
```

也可以考虑修改主页模板来显示`$_SESSION`的状态。这个文件是`/path/to/source/for/this/chapter/expressive/templates/app/home-page.phtml`。只需添加`var_dump($_SESSION)`就可以了。

最初，应该看到这样的东西。

![](../../.gitbook/assets/image%20%28114%29.png)

10秒后，刷新浏览器。现在你应该看到这个。

![](../../.gitbook/assets/image%20%28113%29.png)

