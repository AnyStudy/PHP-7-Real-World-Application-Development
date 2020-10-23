# 创建一个简单的REST客户端

REST客户端使用超文本传输协议\(HTTP\)来生成对外部Web服务的请求。通过改变HTTP方法，我们可以使外部服务执行不同的操作。虽然有不少方法（或动词）可以使用，但我们将只关注`GET`和`POST`。本文中，我们将使用`Adapter`软件设计模式来介绍实现REST客户端的两种不同方式。

## 如何做...

1.在我们定义REST客户端适配器之前，我们需要定义通用类来表示请求和响应信息。首先，我们将从一个抽象类开始，该类具有请求或响应所需的方法和属性。

```php
namespace Application\Web;

class AbstractHttp
{
```

2. 接下来，我们定义代表HTTP信息的类常量。

```php
const METHOD_GET = 'GET';
const METHOD_POST = 'POST';
const METHOD_PUT = 'PUT';
const METHOD_DELETE = 'DELETE';
const CONTENT_TYPE_HTML = 'text/html';
const CONTENT_TYPE_JSON = 'application/json';
const CONTENT_TYPE_FORM_URL_ENCODED = 
  'application/x-www-form-urlencoded';
const HEADER_CONTENT_TYPE = 'Content-Type';
const TRANSPORT_HTTP = 'http';
const TRANSPORT_HTTPS = 'https';
const STATUS_200 = '200';
const STATUS_401 = '401';
const STATUS_500 = '500';
```

3. 然后我们定义请求或响应所需的属性。

```php
protected $uri;      // i.e. http://xxx.com/yyy
protected $method;    // i.e. GET, PUT, POST, DELETE
protected $headers;  // HTTP headers
protected $cookies;  // cookies
protected $metaData;  // information about the transmission
protected $transport;  // i.e. http or https
protected $data = array();
```

4. 顺理成章地为这些属性定义getter和setter。

```php
public function setMethod($method)
{
  $this->method = $method;
}
public function getMethod()
{
  return $this->method ?? self::METHOD_GET;
}
// etc.
```

5. 有些属性需要通过键来访问。为此，我们定义了`getXxxByKey()`和`setXxxByKey()`方法。

```php
public function setHeaderByKey($key, $value)
{
  $this->headers[$key] = $value;
}
public function getHeaderByKey($key)
{
  return $this->headers[$key] ?? NULL;
}
public function getDataByKey($key)
{
  return $this->data[$key] ?? NULL;
}
public function getMetaDataByKey($key)
{
  return $this->metaData[$key] ?? NULL;
}
```

6. 在某些情况下，请求会需要参数，我们假设参数是以PHP数组的形式存储在$data属性中。然后我们可以使用`http_build_query()`函数构建请求的URL。

```php
public function setUri($uri, array $params = NULL)
{
  $this->uri = $uri;
  $first = TRUE;
  if ($params) {
    $this->uri .= '?' . http_build_query($params);
  }
}
public function getDataEncoded()
{
  return http_build_query($this->getData());
}
```

7. 最后，我们根据原始请求设置`$transport`。

```php
public function setTransport($transport = NULL)
{
  if ($transport) {
      $this->transport = $transport;
  } else {
      if (substr($this->uri, 0, 5) == self::TRANSPORT_HTTPS) {
          $this->transport = self::TRANSPORT_HTTPS;
      } else {
          $this->transport = self::TRANSPORT_HTTP;
      }
    }
  }
```

8. 在这个配方中，我们将定义一个`Application\Web\Request`类，当我们希望生成一个请求时，该类可以接受参数，或者，在实现一个接受请求的服务器时，用传入的请求信息填充属性。

```php
namespace Application\Web;
class Request extends AbstractHttp
{
  public function __construct(
    $uri = NULL, $method = NULL, array $headers = NULL, 
    array $data = NULL, array $cookies = NULL)
    {
      if (!$headers) $this->headers = $_SERVER ?? array();
      else $this->headers = $headers;
      if (!$uri) $this->uri = $this->headers['PHP_SELF'] ?? '';
      else $this->uri = $uri;
      if (!$method) $this->method = 
        $this->headers['REQUEST_METHOD'] ?? self::METHOD_GET;
      else $this->method = $method;
      if (!$data) $this->data = $_REQUEST ?? array();
      else $this->data = $data;
      if (!$cookies) $this->cookies = $_COOKIE ?? array();
      else $this->cookies = $cookies;
      $this->setTransport();
    }  
}
```

9. 现在我们可以把注意力转移到响应类上。在这种情况下，我们将定义一个`Application\Web\Received`类。这个名字反映了一个事实，即我们正在重新打包从外部Web服务接收的数据。

```php
namespace Application\Web;
class Received extends AbstractHttp
{
  public function __construct(
    $uri = NULL, $method = NULL, array $headers = NULL, 
    array $data = NULL, array $cookies = NULL)
  {
    $this->uri = $uri;
    $this->method = $method;
    $this->headers = $headers;
    $this->data = $data;
    $this->cookies = $cookies;
    $this->setTransport();
  }  
}
```

### 创建一个基于STREAMS的REST CLIENT

我们现在准备考虑两种不同的方式来实现REST客户端。第一种方法是使用底层的PHP I/O层，称为Streams。该层提供了一系列的包装器，提供对外部流资源的访问。默认情况下，任何PHP文件命令都会使用文件包装器，它提供对本地文件系统的访问。我们将使用`http://` 或 `https://` 包装器来实现 `Application\Web\Client\Streams` 适配器。

1.首先，我们定义一个`Application\Web\Client\Streams`类。

```php
namespace Application\Web\Client;
use Application\Web\ { Request, Received };
class Streams
{
  const BYTES_TO_READ = 4096;
```

2.接下来，我们定义一个方法来将请求发送到外部的Web服务。在`GET`的情况下，我们将参数添加到URI中。在`POST`的情况下，我们创建一个包含元数据的流上下文，指示远程服务我们正在提供数据。使用PHP Streams，发出请求只是一个组成URI的问题，在`POST`的情况下，设置流上下文。然后我们使用一个简单的`fopen()`。

```php
public static function send(Request $request)
{
  $data = $request->getDataEncoded();
  $received = new Received();
  switch ($request->getMethod()) {
    case Request::METHOD_GET :
      if ($data) {
        $request->setUri($request->getUri() . '?' . $data);
      }
      $resource = fopen($request->getUri(), 'r');
      break;
    case Request::METHOD_POST :
      $opts = [
        $request->getTransport() => 
        [
          'method'  => Request::METHOD_POST,
          'header'  => Request::HEADER_CONTENT_TYPE 
          . ': ' . Request::CONTENT_TYPE_FORM_URL_ENCODED,
          'content' => $data
        ]
      ];
      $resource = fopen($request->getUri(), 'w', 
      stream_context_create($opts));
      break;
    }
    return self::getResults($received, $resource);
}
```

3. 最后，我们来看看如何将结果检索和打包成`Received`对象。你会注意到，我们增加了一个规定，对以JSON格式接收的数据进行解码。

```php
protected static function getResults(Received $received, $resource)
{
  $received->setMetaData(stream_get_meta_data($resource));
  $data = $received->getMetaDataByKey('wrapper_data');
  if (!empty($data) && is_array($data)) {
    foreach($data as $item) {
      if (preg_match('!^HTTP/\d\.\d (\d+?) .*?$!', 
          $item, $matches)) {
          $received->setHeaderByKey('status', $matches[1]);
      } else {
          list($key, $value) = explode(':', $item);
          $received->setHeaderByKey($key, trim($value));
      }
    }
  }
  $payload = '';
  while (!feof($resource)) {
    $payload .= fread($resource, self::BYTES_TO_READ);
  }
  if ($received->getHeaderByKey(Received::HEADER_CONTENT_TYPE)) {
    switch (TRUE) {
      case stripos($received->getHeaderByKey(
                   Received::HEADER_CONTENT_TYPE), 
                   Received::CONTENT_TYPE_JSON) !== FALSE:
        $received->setData(json_decode($payload));
        break;
      default :
        $received->setData($payload);
        break;
          }
    }
    return $received;
}
```

### 定义一个基于CURL的REST客户端

现在我们来看看我们第二个REST客户端的方法，其中一个是基于cURL扩展。

1.对于这种方法，我们将假设相同的请求和响应类。初始类的定义与前面讨论的Streams客户端的定义基本相同。

```php
namespace Application\Web\Client;
use Application\Web\ { Request, Received };
class Curl
{
```

2. `send()`方法比使用`Streams`时要简单得多。我们需要做的就是定义一个选项数组，然后让`cURL`来完成剩下的工作。

```php
public static function send(Request $request)
{
  $data = $request->getDataEncoded();
  $received = new Received();
  switch ($request->getMethod()) {
    case Request::METHOD_GET :
      $uri = ($data) 
        ? $request->getUri() . '?' . $data 
        : $request->getUri();
          $options = [
            CURLOPT_URL => $uri,
            CURLOPT_HEADER => 0,
            CURLOPT_RETURNTRANSFER => TRUE,
            CURLOPT_TIMEOUT => 4
          ];
          break;
```

3. `POST`需要的`cURL`参数略有不同

```php
case Request::METHOD_POST :
  $options = [
    CURLOPT_POST => 1,
    CURLOPT_HEADER => 0,
    CURLOPT_URL => $request->getUri(),
    CURLOPT_FRESH_CONNECT => 1,
    CURLOPT_RETURNTRANSFER => 1,
    CURLOPT_FORBID_REUSE => 1,
    CURLOPT_TIMEOUT => 4,
    CURLOPT_POSTFIELDS => $data
  ];
  break;
}
```

4. 然后我们执行一系列的cURL函数，并通过`getResults()`来运行结果。

```php
$ch = curl_init();
curl_setopt_array($ch, ($options));
if( ! $result = curl_exec($ch))
{
  trigger_error(curl_error($ch));
}
$received->setMetaData(curl_getinfo($ch));
curl_close($ch);
return self::getResults($received, $result);
}
```

5.`getResults()`方法将结果打包成一个`Received`对象。

```php
protected static function getResults(Received $received, $payload)
{
  $type = $received->getMetaDataByKey('content_type');
  if ($type) {
    switch (TRUE) {
      case stripos($type, 
          Received::CONTENT_TYPE_JSON) !== FALSE):
          $received->setData(json_decode($payload));
          break;
      default :
          $received->setData($payload);
          break;
    }
  }
  return $received;
}
```

## 如何运行...

请确保将前面所有的代码复制到这些类中。

* `Application\Web\AbstractHttp`
* `Application\Web\Request`
* `Application\Web\Received`
* `Application\Web\Client\Streams`
* `Application\Web\Client\Curl`

在这个例子中，您可以向 Google Maps API 提出 REST 请求，以获取两点之间的驾驶方向。您还需要按照 [https://developers.google.com/maps/documentation/directions/get-api-key](https://developers.google.com/maps/documentation/directions/get-api-key) 给出的说明，为此创建一个 API 密钥。

然后你可以定义一个`chap_07_simple_rest_client_google_maps_curl.php`调用脚本，使用Curl客户端发出请求。再定义一个`chap_07_simple_rest_client_google_maps_streams.php`调用脚本，使用`Streams`客户端发出请求。

```php
<?php
define('DEFAULT_ORIGIN', 'New York City');
define('DEFAULT_DESTINATION', 'Redondo Beach');
define('DEFAULT_FORMAT', 'json');
$apiKey = include __DIR__ . '/google_api_key.php';
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Web\Request;
use Application\Web\Client\Curl;
```

你就可以得到出发地和目的地

```php
$start = $_GET['start'] ?? DEFAULT_ORIGIN;
$end   = $_GET['end'] ?? DEFAULT_DESTINATION;
$start = strip_tags($start);
$end   = strip_tags($end);
```

现在您可以填充Request对象，并使用它来生成请求。

```php
$request = new Request(
  'https://maps.googleapis.com/maps/api/directions/json',
  Request::METHOD_GET,
  NULL,
  ['origin' => $start, 'destination' => $end, 'key' => $apiKey],
  NULL
);

$received = Curl::send($request);
$routes   = $received->getData()->routes[0];
include __DIR__ . '/chap_07_simple_rest_client_google_maps_template.php';
```

为了说明问题，你也可以定义一个模板，代表视图逻辑来显示请求的结果。

```php
<?php foreach ($routes->legs as $item) : ?>
  <!-- Trip Info -->
  <br>Distance: <?= $item->distance->text; ?>
  <br>Duration: <?= $item->duration->text; ?>
  <!-- Driving Directions -->
  <table>
    <tr>
    <th>Distance</th><th>Duration</th><th>Directions</th>
    </tr>
    <?php foreach ($item->steps as $step) : ?>
    <?php $class = ($count++ & 01) ? 'color1' : 'color2'; ?>
    <tr>
    <td class="<?= $class ?>"><?= $step->distance->text ?></td>
    <td class="<?= $class ?>"><?= $step->duration->text ?></td>
    <td class="<?= $class ?>">
    <?= $step->html_instructions ?></td>
    </tr>
    <?php endforeach; ?>
  </table>
<?php endforeach; ?>
```

以下是浏览器中看到的请求结果。

![](../../.gitbook/assets/image%20%2895%29.png)

## 更多...

PHP 标准建议（PSR-7）精确地定义了在 PHP 应用程序之间进行请求时使用的请求和响应对象。这一点在附录 "定义PSR-7类 "中做了详细的介绍。

## 参考

关于流的更多信息，请看这个PHP文档页[http://php.net/manual/en/book.stream.php](http://php.net/manual/en/book.stream.php)。一个经常被问到的问题是 "HTTP PUT和POST之间有什么区别？"关于这个话题的精彩讨论请参考[http://stackoverflow.com/questions/107390/whats-the-difference-between-a-post-and-a-put-http-request](http://stackoverflow.com/questions/107390/whats-the-difference-between-a-post-and-a-put-http-request)。关于从Google获得API密钥的更多信息，请参考这些网页。

[https://developers.google.com/maps/documentation/directions/get-api-key](https://developers.google.com/maps/documentation/directions/get-api-key)

[https://developers.google.com/maps/documentation/directions/intro\#Introduction](https://developers.google.com/maps/documentation/directions/intro#Introduction)

