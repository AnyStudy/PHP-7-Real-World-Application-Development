# 使用中间件来跨语言

除了在不同版本的 PHP 之间进行通信的情况下，PSR-7 中间件的作用很小。回忆一下这个缩写代表什么。PHP 标准建议。因此，如果你需要向一个用其他语言编写的应用程序发出请求，就像对待任何其他 Web 服务的 HTTP 请求一样。

## 如何做...

1.在 PHP 4 的情况下，实际上有机会，因为对面向对象编程的支持是有限的。因此，最好的方法是将前三个示例中描述的基本PSR-7类降级。没有足够的篇幅来涵盖所有的变化，但是我们介绍一个潜在的PHP 4版本的`Application\MiddleWare\ServerRequest`。首先要注意的是没有命名空间！因此，我们使用了一个类名。因此，我们使用带下划线的类名  `_` 来代替命名空间分隔符。

```php
class Application_MiddleWare_ServerRequest
extends Application_MiddleWare_Request
implements Psr_Http_Message_ServerRequestInterface
{
```

2. 在PHP 4中，所有的属性都是用关键字`var`来识别的。

```php
var $serverParams;
var $cookies;
var $queryParams;
// not all properties are shown
```

3. `initialize()`方法几乎是一样的，只是在PHP 4中不允许使用`$this->getServerParams()['REQUEST_URI']`这样的语法。因此，我们需要将其拆分为一个单独的变量。

```php
function initialize()
{
  $params = $this->getServerParams();
  $this->getCookieParams();
  $this->getQueryParams();
  $this->getUploadedFiles;
  $this->getRequestMethod();
  $this->getContentType();
  $this->getParsedBody();
  return $this->withRequestTarget($params['REQUEST_URI']);
}
```

4. 所有的 `$_XXX` 超级全局都在 PHP 4 的以后版本中出现。

```php
function getServerParams()
{
  if (!$this->serverParams) {
      $this->serverParams = $_SERVER;
  }
  return $this->serverParams;
}
// not all getXXX() methods are shown to conserve space
```

5. null coalesce 操作符在 PHP 7 中才被引入，我们需要使用代替 `isset(XXX) ? XXX : ""`。

```php
function getRequestMethod()
{
  $params = $this->getServerParams();
  $method = isset($params['REQUEST_METHOD']) 
    ? $params['REQUEST_METHOD'] : '';
  $this->method = strtolower($method);
  return $this->method;
}
```

6. JSON扩展直到PHP 5才被引入，因此，我们需要满足于原始输入。我们也可以使用 `serialize()` 或`unserialize()`来代替`json_encode()`和`json_decode()`。

```php
function getParsedBody()
{
  if (!$this->parsedBody) {
      if (($this->getContentType() == 
           Constants::CONTENT_TYPE_FORM_ENCODED
           || $this->getContentType() == 
           Constants::CONTENT_TYPE_MULTI_FORM)
           && $this->getRequestMethod() == 
           Constants::METHOD_POST)
      {
          $this->parsedBody = $_POST;
      } elseif ($this->getContentType() == 
                Constants::CONTENT_TYPE_JSON
                || $this->getContentType() == 
                Constants::CONTENT_TYPE_HAL_JSON)
      {
          ini_set("allow_url_fopen", true);
          $this->parsedBody = 
            file_get_contents('php://stdin');
      } elseif (!empty($_REQUEST)) {
          $this->parsedBody = $_REQUEST;
      } else {
          ini_set("allow_url_fopen", true);
          $this->parsedBody = 
            file_get_contents('php://stdin');
      }
  }
  return $this->parsedBody;
}
```

7. `withXXX()`方法在PHP 4中的工作原理基本相同。

```php
function withParsedBody($data)
{
  $this->parsedBody = $data;
  return $this;
}
```

8. 同样，`withoutXXX()`方法也是一样的。

```php
function withoutAttribute($name)
{
  if (isset($this->attributes[$name])) {
      unset($this->attributes[$name]);
  }
  return $this;
}

}
```

9. 对于使用其他语言的网站，我们可以使用PSR-7类来制定请求和响应，但这时需要使用HTTP客户端与其他网站进行通信。作为一个例子，请回忆一下本章的 "开发PSR-7请求类的示例 "中讨论的Request的演示。下面是如何做...部分的例子。

```php
$request = new Request(
  TARGET_WEBSITE_URL,
  Constants::METHOD_POST,
  new TextStream($contents),
  [Constants::HEADER_CONTENT_TYPE => 
  Constants::CONTENT_TYPE_FORM_ENCODED,
  Constants::HEADER_CONTENT_LENGTH => $body->getSize()]
);

$data = http_build_query(['data' => 
$request->getBody()->getContents()]);

$defaults = array(
  CURLOPT_URL => $request->getUri()->getUriString(),
  CURLOPT_POST => true,
  CURLOPT_POSTFIELDS => $data,
);
$ch = curl_init();
curl_setopt_array($ch, $defaults);
$response = curl_exec($ch);
curl_close($ch);
```

