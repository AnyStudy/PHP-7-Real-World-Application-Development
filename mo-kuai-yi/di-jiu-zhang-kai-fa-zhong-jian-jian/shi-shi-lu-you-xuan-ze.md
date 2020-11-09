# 实施路由选择

路由是指接受用户友好的URL，将URL剖析成它的组成部分，然后做出决定应该调度哪个类和方法的过程。这种实现的好处是，不仅可以使你的URLs搜索引擎优化（SEO）友好，而且可以创建规则，结合正则表达式模式，可以提取参数的值。

## 如何做...

1.最流行的方法可能是利用支持URL重写的Web服务器。一个例子是配置为使用 `mod_rewrite` 的Apache网络服务器。然后你定义了重写规则，允许图形文件请求和CSS和JavaScript请求不受影响地通过。否则，请求将通过路由方法被漏过。

2. 另一种潜在的方法是简单地让您的Web服务器虚拟主机定义指向一个特定的路由脚本，然后调用路由类，做出路由决策，并适当地重定向。

3. 第一个要考虑的代码是如何定义路由配置。显而易见的答案是构造一个数组，其中每个键将指向一个正则表达式，URI路径将与之匹配，以及某种形式的操作。下面的代码片段显示了这种配置的一个例子。在这个例子中，我们定义了三个路径：主页、页面和默认值。默认值应该是最后一个，因为它将匹配任何之前没有匹配的内容。这个操作是以匿名函数的形式出现的，如果发生路由匹配，就会被执行。

```php
$config = [
  'home' => [
    'uri' => '!^/$!',
    'exec' => function ($matches) {
      include PAGE_DIR . '/page0.php'; }
  ],
  'page' => [
    'uri' => '!^/(page)/(\d+)$!',
      'exec' => function ($matches) {
        include PAGE_DIR . '/page' . $matches[2] . '.php'; }
  ],
  Router::DEFAULT_MATCH => [
    'uri' => '!.*!',
    'exec' => function ($matches) {
      include PAGE_DIR . '/sorry.php'; }
  ],
];
```

4. 接下来，我们定义我们的`Router`类。我们首先定义了在检查和匹配路由的过程中会用到的常量和属性。

```php
namespace Application\Routing;
use InvalidArgumentException;
use Psr\Http\Message\ServerRequestInterface;
class Router
{
  const DEFAULT_MATCH = 'default';
  const ERROR_NO_DEF  = 'ERROR: must supply a default match';
  protected $request;
  protected $requestUri;
  protected $uriParts;
  protected $docRoot;
  protected $config;
  protected $routeMatch;
```

5. 构造函数接受一个兼容`ServerRequestInterface`的类、文档根目录的路径和前面提到的配置文件。请注意，如果没有提供默认配置，我们会抛出一个异常。

```php
public function __construct(ServerRequestInterface $request, $docRoot, $config)
{
  $this->config = $config;
  $this->docRoot = $docRoot;
  $this->request = $request;
  $this->requestUri = 
    $request->getServerParams()['REQUEST_URI'];
  $this->uriParts = explode('/', $this->requestUri);
  if (!isset($config[self::DEFAULT_MATCH])) {
      throw new InvalidArgumentException(
        self::ERROR_NO_DEF);
  }
}
```

6. 接下来，我们有一系列的getter，可以让我们检索原始请求、文档根和最终的路由匹配。

```php
public function getRequest()
{
  return $this->request;
}
public function getDocRoot()
{
  return $this->docRoot;
}
public function getRouteMatch()
{
  return $this->routeMatch;
}
```

7. `isFileOrDir()`方法用于确定我们是否正在尝试与CSS、JavaScript或图形请求进行匹配（以及其他可能性）。

```php
public function isFileOrDir()
{
  $fn = $this->docRoot . '/' . $this->requestUri;
  $fn = str_replace('//', '/', $fn);
  if (file_exists($fn)) {
      return $fn;
  } else {
      return '';
  }
}
```

8. 最后，我们定义`match()`，它遍历配置数组，并通过`preg_match()`运行 `uri` 参数。如果是正值，那么由 `preg_match()` 填充的配置键和 `$matches` 数组将存储在 `$routeMatch` 中，并返回回调。如果没有匹配，则返回默认回调。

```php
public function match()
{
  foreach ($this->config as $key => $route) {
    if (preg_match($route['uri'], 
        $this->requestUri, $matches)) {
        $this->routeMatch['key'] = $key;
        $this->routeMatch['match'] = $matches;
        return $route['exec'];
    }
  }
  return $this->config[self::DEFAULT_MATCH]['exec'];
}
}
```

## 如何运行...

首先，改成 **`/path/to/source/for/this/chapter`**，并创建一个名为 **`routing`** 的目录。接下来，定义一个文件，`index.php`，它设置了自动加载并使用正确的类。可以定义一个常量`PAGE_DIR`，指向前面示例中创建的页面目录。

```php
<?php
define('DOC_ROOT', __DIR__);
define('PAGE_DIR', DOC_ROOT . '/../pages');

require_once __DIR__ . '/../../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/../..');
use Application\MiddleWare\ServerRequest;
use Application\Routing\Router;
```

接下来，添加本示例第3步中讨论的配置数组。请注意，可以在模式的结尾添加`(/)?`，以说明一个可选的尾部斜杠。另外，对于主页路由，可以提供两个选项：`/`或`/home`。

```php
$config = [
  'home' => [
    'uri' => '!^(/|/home)$!',
    'exec' => function ($matches) {
      include PAGE_DIR . '/page0.php'; }
  ],
  'page' => [
    'uri' => '!^/(page)/(\d+)(/)?$!',
    'exec' => function ($matches) {
      include PAGE_DIR . '/page' . $matches[2] . '.php'; }
  ],
  Router::DEFAULT_MATCH => [
    'uri' => '!.*!',
    'exec' => function ($matches) {
      include PAGE_DIR . '/sorry.php'; }
  ],
];
```

然后可以定义一个路由器实例，提供一个初始化的 `ServerRequest` 实例作为第一个参数。

```php
$router = new Router((new ServerRequest())
  ->initialize(), DOC_ROOT, $config);
$execute = $router->match();
$params  = $router->getRouteMatch()['match'];
```

然后需要检查请求是文件还是目录，还需要检查路径匹配是否为`/`。

```php
if ($fn = $router->isFileOrDir()
    && $router->getRequest()->getUri()->getPath() != '/') {
    return FALSE;
} else {
    include DOC_ROOT . '/main.php';
}
```

接下来，定义 `main.php`，类似这样。

```php
<?php // demo using middleware for routing ?>
<!DOCTYPE html>
<head>
  <title>PHP 7 Cookbook</title>
  <meta http-equiv="content-type" 
  content="text/html;charset=utf-8" />
</head>
<body>
    <?php include PAGE_DIR . '/route_menu.php'; ?>
    <?php $execute($params); ?>
</body>
</html>
```

最后，还需要修订使用用户友好路由的菜单。

```php
<?php // menu for routing ?>
<a href="/home">Home</a>
<a href="/page/1">Page 1</a>
<a href="/page/2">Page 2</a>
<a href="/page/3">Page 3</a>
<!-- etc. -->
```

要使用Apache测试配置，定义一个指向 `/path/to/source/for/this/chapter/routing` 的虚拟主机定义。另外，定义一个`.htaccess`文件，将任何不是文件、目录或链接的请求导向`index.php`。另外，您也可以使用内置的PHP webserver。在终端窗口或命令提示符下，键入这个命令。

```bash
cd /path/to/source/for/this/chapter/routing
php -S localhost:8080
```

在浏览器中，当请求[http://localhost:8080/home](http://localhost:8080/home) 时的输出是这样的。

![](../../.gitbook/assets/image%20%28116%29.png)

## 另见

关于使用NGINX web服务器重写的信息，请看这篇文章：[http://nginx.org/en/docs/http/ngx\_http\_rewrite\_module.html](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)。有很多复杂的 PHP 路由库可以提供比这里介绍的简单路由器更强大的功能。这些包括Altorouter \([http://altorouter.com/](http://altorouter.com/)\), TreeRoute \([https://github.com/baryshev/TreeRoute](https://github.com/baryshev/TreeRoute)\), FastRoute \([https://github.com/nikic/FastRoute](https://github.com/nikic/FastRoute)\), 和Aura.Router. \([https://github.com/auraphp/Aura.Router](https://github.com/auraphp/Aura.Router)\)。此外，大多数框架（例如，Zend Framework 2或CodeIgniter）都有自己的路由功能。

