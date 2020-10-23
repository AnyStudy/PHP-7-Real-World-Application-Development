# 创建一个简单的SOAP服务器

和SOAP客户端一样，我们可以使用PHP SOAP扩展来实现一个SOAP服务器。实现中最困难的部分是从API类生成WSDL。我们不在这里介绍这个过程，因为有很多好的WSDL生成器可用。

## 如何做...

1.首先，你需要一个将由SOAP服务器处理的API。在这个例子中，我们定义了一个`Application\Web\Soap\ProspectsAp`i类，它允许我们创建、读取、更新和删除`prospects`表。

```php
namespace Application\Web\Soap;
use PDO;
class ProspectsApi
{
  protected $registerKeys;
  protected $pdo;
        
  public function __construct($pdo, $registeredKeys)
  {
    $this->pdo = $pdo;
    $this->registeredKeys = $registeredKeys;
  }
}
```

2. 然后我们定义方法，分别对应创建、读取、更新和删除。在这个例子中，这些方法被命名为`put()`、`get()`、`post()`和`delete()`。这些方法依次调用产生SQL请求的方法，这些请求将从PDO实例中执行。get\(\)的例子如下。

```php
public function get(array $request, array $response)
{
  if (!$this->authenticate($request)) return FALSE;
  $result = array();
  $id = $request[self::ID_FIELD] ?? 0;
  $email = $request[self::EMAIL_FIELD] ?? 0;
  if ($id > 0) {
      $result = $this->fetchById($id);  
      $response[self::ID_FIELD] = $id;
  } elseif ($email) {
      $result = $this->fetchByEmail($email);
      $response[self::ID_FIELD] = $result[self::ID_FIELD] ?? 0;
  } else {
      $limit = $request[self::LIMIT_FIELD] 
        ?? self::DEFAULT_LIMIT;
      $offset = $request[self::OFFSET_FIELD] 
        ?? self::DEFAULT_OFFSET;
      $result = [];
      foreach ($this->fetchAll($limit, $offset) as $row) {
        $result[] = $row;
      }
  }
  $response = $this->processResponse(
    $result, $response, self::SUCCESS, self::ERROR);
    return $response;
  }

  protected function processResponse($result, $response, 
                                     $success_code, $error_code)
  {
    if ($result) {
        $response['data'] = $result;
        $response['code'] = $success_code;
        $response['status'] = self::STATUS_200;
    } else {
        $response['data'] = FALSE;
        $response['code'] = self::ERROR_NOT_FOUND;
        $response['status'] = self::STATUS_500;
    }
    return $response;
  }
```

3.然后你可以从你的 API 中生成一个 WSDL。有很多基于 PHP 的 WSDL 生成器可以使用。大多数需要你在将要发布的方法之前添加 phpDocumentor 标签。在我们的例子中，两个参数都是数组。这里是前面讨论的 API 的完整 WSDL。

```markup
<?xml version="1.0" encoding="UTF-8"?>
  <wsdl:definitions xmlns:tns="php7cookbook" targetNamespace="php7cookbook" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:s="http://www.w3.org/2001/XMLSchema" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/">
  <wsdl:message name="getSoapIn">
    <wsdl:part name="request" type="tns:array" />
    <wsdl:part name="response" type="tns:array" />
  </wsdl:message>
  <wsdl:message name="getSoapOut">
    <wsdl:part name="return" type="tns:array" />
  </wsdl:message>
  <!—some nodes removed to conserve space -->
  <wsdl:portType name="CustomerApiSoap">
  <!—some nodes removed to conserve space -->
  <wsdl:binding name="CustomerApiSoap" type="tns:CustomerApiSoap">
  <soap:binding transport="http://schemas.xmlsoap.org/soap/http" style="rpc" />
    <wsdl:operation name="get">
      <soap:operation soapAction="php7cookbook#get" />
        <wsdl:input>
          <soap:body use="encoded" encodingStyle= "http://schemas.xmlsoap.org/soap/encoding/" namespace="php7cookbook" parts="request response" />
        </wsdl:input>
        <wsdl:output>
          <soap:body use="encoded" encodingStyle= "http://schemas.xmlsoap.org/soap/encoding/" namespace="php7cookbook" parts="return" />
        </wsdl:output>
    </wsdl:operation>
  <!—some nodes removed to conserve space -->
  </wsdl:binding>
  <wsdl:service name="CustomerApi">
    <wsdl:port name="CustomerApiSoap" binding="tns:CustomerApiSoap">
    <soap:address location="http://localhost:8080/" />
    </wsdl:port>
  </wsdl:service>
  </wsdl:definitions>
```

4.接下来，创建一个`chap_07_simple_soap_server.php`文件，它将执行SOAP服务器。首先定义WSDL和其他必要文件的位置\(在本例中，一个是数据库配置文件\)。如果设置了wsdl参数，那么就传递WSDL而不是尝试处理请求。在这个例子中，我们使用一个简单的API密钥来验证请求。然后我们创建一个SOAP服务器实例，分配一个API类的实例，并运行`handle()`。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
define('WSDL_FILENAME', __DIR__ . '/chap_07_wsdl.xml');
      
if (isset($_GET['wsdl'])) {
    readfile(WSDL_FILENAME);
    exit;
}
$apiKey = include __DIR__ . '/api_key.php';
require __DIR__ . '/../Application/Web/Soap/ProspectsApi.php';
require __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
use Application\Web\Soap\ProspectsApi;
$connection = new Application\Database\Connection(
  include __DIR__ . DB_CONFIG_FILE);
$api = new Application\Web\Soap\ProspectsApi(
  $connection->pdo, [$apiKey]);
$server = new SoapServer(WSDL_FILENAME);
$server->setObject($api);
echo $server->handle();
```

{% hint style="info" %}
根据你的php.ini文件的设置，你可能需要禁用WSDL缓存，如下所示。

```text
ini_set('soap.wsdl_cache_enabled', 0);
```

如果传入的POST数据有问题，可以按以下方式调整这个参数。

```text
ini_set('always_populate_raw_post_data', -1);
```
{% endhint %}

## 如何运行...

你可以首先创建你的目标API类，然后生成一个WSDL来轻松测试这个事例。然后，你可以使用内置的PHP webserver来交付SOAP服务，使用这个命令。

```bash
php -S localhost:8080 chap_07_simple_soap_server.php 
```

然后，你可以使用前面事例中讨论的SOAP客户端来调用测试SOAP服务。

```php
<?php
define('WSDL_URL', 'http://localhost:8080?wsdl=1');
$clientKey = include __DIR__ . '/api_key.php';
try {
  $client = new SoapClient(WSDL_URL);
  $response = [];
  $email = some_email_generated_by_test;
  $email = 'test5393@unlikelysource.com';
  echo "\nGet Prospect Info for Email: " . $email . "\n";
  $request = ['token' => $clientKey, 'email' => $email];
  $result = $client->get($request,$response);
  var_dump($result);
  
} catch (SoapFault $e) {
  echo 'ERROR' . PHP_EOL;
  echo $e->getMessage() . PHP_EOL;
} catch (Throwable $e) {
  echo 'ERROR' . PHP_EOL;
  echo $e->getMessage() . PHP_EOL;
} finally {
  echo $client->__getLastResponse() . PHP_EOL;
}
```

以下是电子邮件地址`test5393@unlikelysource.com` 的输出。

![](../../.gitbook/assets/image%20%28101%29.png)

## 更多...

简单的谷歌搜索PHP的WSDL生成器，很容易就得到了十几个结果。用来生成`ProspectsApi`类的WSDL的是基于[https://code.google.com/archive/p/php-wsdl-creator/](https://code.google.com/archive/p/php-wsdl-creator/)。关于`phpDocumentor`的更多信息，请参考[https://www.phpdoc.org/](https://www.phpdoc.org/)。

