# 创建一个简单的REST服务器

在实现REST服务器时，有几个注意事项。这三个问题的答案将让你定义你的REST服务。

* 如何捕获原始请求？ 
* 你想发布什么应用编程接口（API）？ 
* 你打算如何将HTTP动词（例如，GET、PUT、POST和DELETE）映射到API方法？

## 如何做...

1.我们将通过在之前的 《创建一个简单的REST客户端 》中定义的请求和响应类的基础上实现我们的REST服务器。回顾之前事例中讨论的类，包括以下内容。

* `Application\Web\AbstractHttp`
* `Application\Web\Request`
* `Application\Web\Received`

2.我们还需要在AbstractHttp的基础上，定义一个正式的`Application\Web\Response`响应类。这个类和其他类的主要区别在于它接受`Application\Web\Request`的实例作为参数。主要工作在`__construct()`方法中完成。设置`Content-Type`头和状态也很重要。

```php
namespace Application\Web;
class Response extends AbstractHttp
{

  public function __construct(Request $request = NULL, 
                              $status = NULL, $contentType = NULL)
  {
    if ($request) {
      $this->uri = $request->getUri();
      $this->data = $request->getData();
      $this->method = $request->getMethod();
      $this->cookies = $request->getCookies();
      $this->setTransport();
    }
    $this->processHeaders($contentType);
    if ($status) {
      $this->setStatus($status);
    }
  }
  protected function processHeaders($contentType)
  {
    if (!$contentType) {
      $this->setHeaderByKey(self::HEADER_CONTENT_TYPE, 
        self::CONTENT_TYPE_JSON);
    } else {
      $this->setHeaderByKey(self::HEADER_CONTENT_TYPE, 
        $contentType);
    }
  }
  public function setStatus($status)
  {
    $this->status = $status;
  }
  public function getStatus()
  {
    return $this->status;
  }
}
```

3.我们现在可以定义`Application\WebRest\Server`类。你可能会惊讶于它的简单程度。真正的工作是在相关的API类中完成的。

{% hint style="info" %}
注意使用PHP 7的组使用语法。

```php
use Application\Web\ { Request,Response,Received }
```
{% endhint %}

```php
namespace Application\Web\Rest;
use Application\Web\ { Request, Response, Received };
class Server
{
  protected $api;
  public function __construct(ApiInterface $api)
  {
    $this->api = $api;
  }
```

4. 接下来，我们定义一个`listen()`方法，作为请求的目标。服务器实现的核心就是这行代码。

```php
$jsonData = json_decode(file_get_contents('php://input'),true);
```

5. 这将捕获原始输入，假定为JSON格式。

```php
public function listen()
{
  $request  = new Request();
  $response = new Response($request);
  $getPost  = $_REQUEST ?? array();
  $jsonData = json_decode(
    file_get_contents('php://input'),true);
  $jsonData = $jsonData ?? array();
  $request->setData(array_merge($getPost,$jsonData));
```

{% hint style="info" %}
我们还增加了一项认证条款。否则，任何人都可以提出请求并获得潜在的敏感数据。你会注意到，我们并没有让服务器类来执行认证，而是让API类来执行。

```php
if (!$this->api->authenticate($request)) {
    $response->setStatus(Request::STATUS_401);
    echo $this->api::ERROR;
    exit;
}
```
{% endhint %}

6. 然后，我们将API方法映射到主要的HTTP方法`GET`、`PUT`、`POST`和`DELETE`。

```php
$id = $request->getData()[$this->api::ID_FIELD] ?? NULL;
switch (strtoupper($request->getMethod())) {
  case Request::METHOD_POST :
    $this->api->post($request, $response);
    break;
  case Request::METHOD_PUT :
    $this->api->put($request, $response);
    break;
  case Request::METHOD_DELETE :
    $this->api->delete($request, $response);
    break;
  case Request::METHOD_GET :
  default :
    // return all if no params
  $this->api->get($request, $response);
}
```

7. 最后，我们将响应打包并发送出去

```php
  $this->processResponse($response);
  echo json_encode($response->getData());
}
```

8. `processResponse()`方法设置了头文件，并确保结果被打包为`Application\Web\Response`对象。

```php
protected function processResponse($response)
{
  if ($response->getHeaders()) {
    foreach ($response->getHeaders() as $key => $value) {
      header($key . ': ' . $value, TRUE, 
             $response->getStatus());
    }
  }        
  header(Request::HEADER_CONTENT_TYPE 
  . ': ' . Request::CONTENT_TYPE_JSON, TRUE);
  if ($response->getCookies()) {
    foreach ($response->getCookies() as $key => $value) {
      setcookie($key, $value);
    }
  }
}
```

9.如前所述，真正的工作是由API类完成的。我们首先定义一个抽象类，确保主要的方法`get()`、`put()`等被表示出来，并且所有这些方法都接受请求和响应对象作为参数。你可能会注意到，我们添加了一个 `generateToken()` 方法，使用 PHP 7 `random_bytes()` 函数来生成一个真正随机的 16 字节随机数。

```php
namespace Application\Web\Rest;
use Application\Web\ { Request, Response };
abstract class AbstractApi implements ApiInterface
{
  const TOKEN_BYTE_SIZE  = 16;
  protected $registeredKeys;
  abstract public function get(Request $request, 
                               Response $response);
  abstract public function put(Request $request, 
                               Response $response);
  abstract public function post(Request $request, 
                                Response $response);
  abstract public function delete(Request $request, 
                                  Response $response);
  abstract public function authenticate(Request $request);
  public function __construct($registeredKeys, $tokenField)
  {
    $this->registeredKeys = $registeredKeys;
  }
  public static function generateToken()
  {
    return bin2hex(random_bytes(self::TOKEN_BYTE_SIZE));    
  }
}
```

10. 我们还定义了一个相应的接口，可以用于架构和设计的目的，以及代码开发控制。

```php
namespace Application\Web\Rest;
use Application\Web\ { Request, Response };
interface ApiInterface
{
  public function get(Request $request, Response $response);
  public function put(Request $request, Response $response);
  public function post(Request $request, Response $response);
  public function delete(Request $request, Response $response);
  public function authenticate(Request $request);
}
```

11. 这里，我们介绍一个基于AbstractApi的示例API。这个类利用了第5章《与数据库的交互》中定义的数据库类。

```php
namespace Application\Web\Rest;
use Application\Web\ { Request, Response, Received };
use Application\Entity\Customer;
use Application\Database\ { Connection, CustomerService };

class CustomerApi extends AbstractApi
{
  const ERROR = 'ERROR';
  const ERROR_NOT_FOUND = 'ERROR: Not Found';
  const SUCCESS_UPDATE = 'SUCCESS: update succeeded';
  const SUCCESS_DELETE = 'SUCCESS: delete succeeded';
  const ID_FIELD = 'id';      // field name of primary key
  const TOKEN_FIELD = 'token';  // field used for authentication
  const LIMIT_FIELD = 'limit';
  const OFFSET_FIELD = 'offset';
  const DEFAULT_LIMIT = 20;
  const DEFAULT_OFFSET = 0;
      
  protected $service;
      
  public function __construct($registeredKeys, 
                              $dbparams, $tokenField = NULL)
  {
    parent::__construct($registeredKeys, $tokenField);
    $this->service = new CustomerService(
      new Connection($dbparams));
  }
```

12.所有方法都接收请求和响应作为参数。你会注意到使用`getDataByKey()`来检索数据项。实际的数据库交互是由服务类执行的。你可能还会注意到，在所有情况下，我们都设置了一个HTTP状态码来通知客户端成功或失败。在`get()`的情况下，我们寻找一个ID参数。如果收到，我们只传递单个客户的信息。否则，我们使用`limit`和`offset`来传递所有客户的列表。

```php
public function get(Request $request, Response $response)
{
  $result = array();
  $id = $request->getDataByKey(self::ID_FIELD) ?? 0;
  if ($id > 0) {
      $result = $this->service->
        fetchById($id)->entityToArray();  
  } else {
    $limit  = $request->getDataByKey(self::LIMIT_FIELD) 
      ?? self::DEFAULT_LIMIT;
    $offset = $request->getDataByKey(self::OFFSET_FIELD) 
      ?? self::DEFAULT_OFFSET;
    $result = [];
    $fetch = $this->service->fetchAll($limit, $offset);
    foreach ($fetch as $row) {
      $result[] = $row;
    }
  }
  if ($result) {
      $response->setData($result);
      $response->setStatus(Request::STATUS_200);
  } else {
      $response->setData([self::ERROR_NOT_FOUND]);
      $response->setStatus(Request::STATUS_500);
  }
}
```

13.`put()`方法用于插入客户数据。

```php
public function put(Request $request, Response $response)
{
  $cust = Customer::arrayToEntity($request->getData(), 
                                  new Customer());
  if ($newCust = $this->service->save($cust)) {
      $response->setData(['success' => self::SUCCESS_UPDATE, 
                          'id' => $newCust->getId()]);
      $response->setStatus(Request::STATUS_200);
  } else {
      $response->setData([self::ERROR]);
      $response->setStatus(Request::STATUS_500);
  }      
}
```

14.`post()`方法用于更新现有的客户条目。

```php
public function post(Request $request, Response $response)
{
  $id = $request->getDataByKey(self::ID_FIELD) ?? 0;
  $reqData = $request->getData();
  $custData = $this->service->
    fetchById($id)->entityToArray();
  $updateData = array_merge($custData, $reqData);
  $updateCust = Customer::arrayToEntity($updateData, 
  new Customer());
  if ($this->service->save($updateCust)) {
      $response->setData(['success' => self::SUCCESS_UPDATE, 
                          'id' => $updateCust->getId()]);
      $response->setStatus(Request::STATUS_200);
  } else {
      $response->setData([self::ERROR]);
      $response->setStatus(Request::STATUS_500);
  }      
}
```

15.顾名思义，`delete()`删除客户条目。

```php
public function delete(Request $request, Response $response)
{
  $id = $request->getDataByKey(self::ID_FIELD) ?? 0;
  $cust = $this->service->fetchById($id);
  if ($cust && $this->service->remove($cust)) {
      $response->setData(['success' => self::SUCCESS_DELETE, 
                          'id' => $id]);
      $response->setStatus(Request::STATUS_200);
  } else {
      $response->setData([self::ERROR_NOT_FOUND]);
      $response->setStatus(Request::STATUS_500);
  }
}
```

16.最后，我们定义了`authenticate()`，在这个例子中，该方法作为底层机制来保护API的使用。

```php
public function authenticate(Request $request)
{
  $authToken = $request->getDataByKey(self::TOKEN_FIELD) 
    ?? FALSE;
  if (in_array($authToken, $this->registeredKeys, TRUE)) {
      return TRUE;
  } else {
      return FALSE;
  }
}
}
```

## 如何运行...

定义以下类，这在前面的事例中已经讨论过。

* `Application\Web\AbstractHttp`
* `Application\Web\Request`
* `Application\Web\Received`

然后，你可以定义以下类，在本事例中描述，总结在这个表中。

| Class Application\Web\\* | 在这些步骤中讨论 |
| :--- | :--- |
| `Response` | 2 |
| `Rest\Server` | 3 - 8 |
| `Rest\AbstractApi` | 9 |
| `Rest\ApiInterface` | 10 |
| `Rest\CustomerApi` | 11 - 16 |

现在你可以自由地开发你自己的API类了。然而，如果你选择按照`Application\Web\Rest\CustomerApi`的说明，你还需要确保实现这些类，在第5章 《与数据库的交互》中有所涉及。

* `Application\Entity\Customer`
* `Application\Database\Connection`
* `Application\Database\CustomerService`

现在你可以定义一个`chap_07_simple_rest_server.php`脚本来调用REST服务器。

```php
<?php
$dbParams = include __DIR__ .  '/../../config/db.config.php';
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Web\Rest\Server;
use Application\Web\Rest\CustomerApi;
$apiKey = include __DIR__ . '/api_key.php';
$server = new Server(new CustomerApi([$apiKey], $dbParams, 'id'));
$server->listen();
```

然后你可以使用内置的PHP 7开发服务器来监听`8080`端口的REST请求。

```bash
php -S localhost:8080 chap_07_simple_rest_server.php
```

要测试你的API，使用`Application\Web\Rest\AbstractApi::generateToken()`方法来生成一个认证令牌，你可以把它放在`api_key.php`文件中，就像这样。

```php
<?php return '79e9b5211bbf2458a4085707ea378129';
```

然后，你可以使用一个通用的API客户端（如前面的事例中描述的客户端），或者一个浏览器插件，如Chao Zhou的RESTClient（更多信息见http://restclient.net/）来生成示例请求。确保你的请求包含了令牌，否则定义的API会拒绝该请求。

下面是一个ID 为  1的POST请求的例子，它将余额字段设置为888888。

![](../../.gitbook/assets/image%20%2892%29.png)

## 更多...

有很多库可以帮助你实现一个REST服务器。我最喜欢的一个例子是在一个文件中实现REST服务器：[https://www.leaseweb.com/labs/2015/10/creating-a-simple-rest-api-in-php/](https://www.leaseweb.com/labs/2015/10/creating-a-simple-rest-api-in-php/)

各种框架，如CodeIgniter和Zend Framework，也有REST服务器的实现。

