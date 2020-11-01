# 使用中间件实现访问控制

顾名思义，中间件位于函数或方法调用序列的中间。相应地，中间件很适合做 "守门人 "的任务。你可以很容易地用中间件类实现访问控制列表（ACL）机制，中间件类读取ACL，并允许或拒绝对序列中下一个函数或方法调用的访问。

## 如何做...

1.这个过程中最困难的部分可能是确定哪些因素要包括在ACL中。为了说明问题，我们假设我们的用户都被分配了一个 `level` 和一个 `status`。在这个例子中，级别的定义如下。

```php
  'levels' => [0, 'BEG', 'INT', 'ADV']
```

2. 状态可以表明他们在会员注册过程中的进展情况。例如，状态为0表示他们已经启动了会员注册程序，但尚未得到确认。状态为1时，表示他们的电子邮件地址已经确认，但他们还没有支付月费，以此类推。

3. 接下来，我们需要定义我们计划控制的资源。在本例中，我们将假设需要控制对网站上一系列网页的访问。据此，我们需要定义一个此类资源的数组。在ACL中，我们就可以参考键。

```php
'pages'  => [0 => 'sorry', 'logout' => 'logout', 'login'  => 'auth',
             1 => 'page1', 2 => 'page2', 3 => 'page3',
             4 => 'page4', 5 => 'page5', 6 => 'page6',
             7 => 'page7', 8 => 'page8', 9 => 'page9']
```

4. 最后，最重要的一项配置是根据级别和状态对页面进行分配。配置数组中使用的通用模板可能是这样的。

```php
status => ['inherits' => <key>, 'pages' => [level => [pages allowed], etc.]]
```

5. 现在我们可以定义Acl类了。和之前一样，我们使用一些类，并定义适合访问控制的常量和属性。

```php
namespace Application\Acl;

use InvalidArgumentException;
use Psr\Http\Message\RequestInterface;
use Application\MiddleWare\ { Constants, Response, TextStream };

class Acl
{
  const DEFAULT_STATUS = '';
  const DEFAULT_LEVEL  = 0;
  const DEFAULT_PAGE   = 0;
  const ERROR_ACL = 'ERROR: authorization error';
  const ERROR_APP = 'ERROR: requested page not listed';
  const ERROR_DEF = 
    'ERROR: must assign keys "levels", "pages" and "allowed"';
  protected $default;
  protected $levels;
  protected $pages;
  protected $allowed; 
```

6. 在 `__construct()` 方法中，我们将分配数组分解为 `$pages`（要控制的资源）、`$levels` 和 `$allowed`（实际的分配）。如果数组不包括这三个子组件中的一个，就会抛出一个异常。

```php
public function __construct(array $assignments)
{
  $this->default = $assignments['default'] 
    ?? self::DEFAULT_PAGE;
  $this->pages   = $assignments['pages'] ?? FALSE;
  $this->levels  = $assignments['levels'] ?? FALSE;
  $this->allowed = $assignments['allowed'] ?? FALSE;
  if (!($this->pages && $this->levels && $this->allowed)) {
      throw new InvalidArgumentException(self::ERROR_DEF);
  }
}
```

7. 您可能已经注意到，我们允许继承。在`$allowed`中，继承的键可以设置为数组中的另一个键。如果是这样，我们需要将它的值与当前正在检查的值合并。我们反向迭代`$allowed`，每次通过循环合并任何继承的值。顺便说一下，这个方法也只是隔离适用于某个状态和级别的规则。

```php
protected function mergeInherited($status, $level)
{
  $allowed = $this->allowed[$status]['pages'][$level] 
    ?? array();
  for ($x = $status; $x > 0; $x--) {
    $inherits = $this->allowed[$x]['inherits'];
    if ($inherits) {
        $subArray = 
          $this->allowed[$inherits]['pages'][$level] 
          ?? array();
        $allowed = array_merge($allowed, $subArray);
    }
  }
  return $allowed;
}
```

8. 在处理授权时，我们会初始化一些变量，然后从原始请求URI中提取所请求的页面。如果页面参数不存在，我们设置一个400代码。

```php
public function isAuthorized(RequestInterface $request)
{
  $code = 401;    // unauthorized
  $text['page'] = $this->pages[$this->default];
  $text['authorized'] = FALSE;
  $page = $request->getUri()->getQueryParams()['page'] 
    ?? FALSE;
  if ($page === FALSE) {
      $code = 400;    // bad request
```

9. 否则，我们对请求体内容进行解码，并获得状态和级别。然后我们就可以调用`mergeInherited()`，它返回一个可以访问这个状态和级别的页面数组。

```php
} else {
    $params = json_decode(
      $request->getBody()->getContents());
    $status = $params->status ?? self::DEFAULT_LEVEL;
    $level  = $params->level  ?? '*';
    $allowed = $this->mergeInherited($status, $level);
```

10. 如果请求的页面在`$allowed`数组中，我们就将状态码设置为200，并将授权设置连同请求的页面代码对应的网页一起返回。

```php
if (in_array($page, $allowed)) {
    $code = 200;    // OK
    $text['authorized'] = TRUE;
    $text['page'] = $this->pages[$page];
} else {
    $code = 401;            }
}
```

11. 然后我们返回JSON编码的响应，我们就完成了。

```php
$body = new TextStream(json_encode($text));
return (new Response())->withStatus($code)
->withBody($body);
}

}
```

## 如何运行...

之后，你将需要定义`ApplicationA\cl\Acl`，这将在本文中讨论。现在移动到`/path/to/source/for/this/chapter`文件夹，创建两个目录：`public`和`pages`。在`pages`中，创建一系列PHP文件，如`page1.php`、`page2.php`等。下面是这些页面中的一个例子。

```php
<?php // page 1 ?>
<h1>Page 1</h1>
<hr>
<p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. etc.</p>
```

你也可以定义一个 `menu.php`页面，它可以被包含在输出中。

```php
<?php // menu ?>
<a href="?page=1">Page 1</a>
<a href="?page=2">Page 2</a>
<a href="?page=3">Page 3</a>
// etc.
```

`logout.php`页面应该销毁会话。

```php
<?php
  $_SESSION['info'] = FALSE;
  session_destroy();
?>
<a href="/">BACK</a>
```

`auth.php`页面将显示一个登录界面（如前所述）。

```php
<?= $auth->getLoginForm($action) ?>
```

然后可以创建一个配置文件，允许根据级别和状态来访问网页。为了便于说明，调用`chap_09_middleware_acl_config.php`，并返回一个数组，看起来像这样。

```php
<?php
$min = [0, 'logout'];
return [
  'default' => 0,     // default page
  'levels' => [0, 'BEG', 'INT', 'ADV'],
  'pages'  => [0 => 'sorry', 
  'logout' => 'logout', 
  'login' => 'auth',
               1 => 'page1', 2 => 'page2', 3 => 'page3',
               4 => 'page4', 5 => 'page5', 6 => 'page6',
               7 => 'page7', 8 => 'page8', 9 => 'page9'],
  'allowed' => [
               0 => ['inherits' => FALSE,
                     'pages' => [ '*' => $min, 'BEG' => $min,
                     'INT' => $min,'ADV' => $min]],
               1 => ['inherits' => FALSE,
                     'pages' => ['*' => ['logout'],
                    'BEG' => [1, 'logout'],
                    'INT' => [1,2, 'logout'],
                    'ADV' => [1,2,3, 'logout']]],
               2 => ['inherits' => 1,
                     'pages' => ['BEG' => [4],
                     'INT' => [4,5],
                     'ADV' => [4,5,6]]],
               3 => ['inherits' => 2,
                     'pages' => ['BEG' => [7],
                     'INT' => [7,8],
                     'ADV' => [7,8,9]]]
    ]
];
```

最后，在公共文件夹中，定义`index.php`，它设置了自动加载，并最终调用`Authenticate`和`Acl`类。就像其他示例一样，定义配置文件，设置自动加载，并使用某些类。另外，别忘了启动会话。

```php
<?php
session_start();
session_regenerate_id();
define('DB_CONFIG_FILE', __DIR__ . '/../../config/db.config.php');
define('DB_TABLE', 'customer_09');
define('PAGE_DIR', __DIR__ . '/../pages');
define('SESSION_KEY', 'auth');
require __DIR__ . '/../../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/../..');

use Application\Database\Connection;
use Application\Acl\ { Authenticate, Acl };
use Application\MiddleWare\ { ServerRequest, Request, Constants, TextStream };
```

{% hint style="info" %}
**最佳实践**

最好的做法是保护你的会话。一个简单的方法是使用 `session_regenerate_id()`，它可以使现有的 PHP 会话标识符无效并生成一个新的标识符。因此，如果攻击者通过非法手段获得会话标识符，任何给定的会话标识符的有效时间窗口都会被保持在最小范围内。
{% endhint %}

现在你可以调出ACL配置，并为`Authenticate`以及`Acl`创建实例。

```php
$config = require __DIR__ . '/../chap_09_middleware_acl_config.php';
$acl    = new Acl($config);
$conn   = new Connection(include DB_CONFIG_FILE);
$dbAuth = new DbTable($conn, DB_TABLE);
$auth   = new Authenticate($dbAuth, SESSION_KEY);
```

接下来，定义入站和出站请求实例。

```php
$incoming = new ServerRequest();
$incoming->initialize();
$outbound = new Request();
```

如果传入的请求方法是`post`，则调用`login()`方法处理认证。

```php
if (strtolower($incoming->getMethod()) == Constants::METHOD_POST) {
    $body = new TextStream(json_encode(
    $incoming->getParsedBody()));
    $response = $auth->login($outbound->withBody($body));
}
```

如果为认证定义的会话密钥被填充，就意味着用户已经成功认证。如果没有，我们编写一个匿名函数，后面会调用，其中包括认证登录页面。

```php
$info = $_SESSION[SESSION_KEY] ?? FALSE;
if (!$info) {
    $execute = function () use ($auth) {
      include PAGE_DIR . '/auth.php';
    };
```

否则，你可以继续进行ACL检查。首先需要从原始查询中找到用户想要访问的网页。

```php
} else {
    $query = $incoming->getServerParams()['QUERY_STRING'] ?? '';
```

然后，您可以重新编写 `$outbound` 请求，以包含这些信息。

```php
$outbound->withBody(new TextStream(json_encode($info)));
$outbound->getUri()->withQuery($query);
```

接下来，你就可以检查授权了，提供出站请求作为参数。

```php
$response = $acl->isAuthorized($outbound);
```

然后可以检查返回响应的授权参数，并编程匿名函数，如果OK则包含返回页参数，否则包含抱歉页。

```php
$params   = json_decode($response->getBody()->getContents());
$isAllowed = $params->authorized ?? FALSE;
if ($isAllowed) {
    $execute = function () use ($response, $params) {
      include PAGE_DIR .'/' . $params->page . '.php';
      echo '<pre>', var_dump($response), '</pre>';
      echo '<pre>', var_dump($_SESSION[SESSION_KEY]);
      echo '</pre>';
    };
} else {
    $execute = function () use ($response) {
      include PAGE_DIR .'/sorry.php';
      echo '<pre>', var_dump($response), '</pre>';
      echo '<pre>', var_dump($_SESSION[SESSION_KEY]);
      echo '</pre>';
    };
}
}
```

现在你需要做的就是设置表单动作，并将匿名函数包装在HTML中。

```php
$action = $incoming->getServerParams()['PHP_SELF'];
?>
<!DOCTYPE html>
<head>
  <title>PHP 7 Cookbook</title>
  <meta http-equiv="content-type" content="text/html;charset=utf-8" />
</head>
<body>
  <?php $execute(); ?>
</body>
</html>
```

为了测试它，你可以使用内置的PHP web服务器，但你需要使用-t标志来表明文档根是公开的。

```bash
cd /path/to/source/for/this/chapter
php -S localhost:8080 -t public
```

从浏览器中，您可以访问[http://localhost:8080/](http://localhost:8080/) 。

如果试图访问任何页面，您会被重定向回登录页面。根据配置，status=1，level=BEG的用户只能访问第1页并注销。如果你以该用户身份登录时，试图访问第2页，这里是输出结果。

![](../../.gitbook/assets/image%20%28110%29.png)

## 更多...

这个例子依靠`$_SESSION`作为用户登录后唯一的认证手段。关于如何保护PHP会话的好例子，请参见第12章，提高网站安全，特别是题为保护PHP会话的示例。

