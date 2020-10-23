# 使用中间件进行认证

中间件的一个非常重要的用途是提供认证。大多数基于网络的应用程序都需要通过用户名和密码来验证访问者的能力。通过将PSR-7标准整合到一个认证类中，你将使它在全局范围内通用，可以说，它足够安全，可以在任何提供符合PSR-7的请求和响应对象的框架中使用。

## 如何做...

1.我们首先定义一个`Application\Acl\AuthenticateInterface`类。我们使用这个接口来支持Adapter软件设计模式，使我们的`Authenticate`类更加通用，允许各种适配器，每个适配器都可以从不同的源头（例如，从一个文件，使用OAuth2，等等）获取认证。注意使用 PHP 7 的能力来定义返回值数据类型。

```php
namespace Application\Acl;
use Psr\Http\Message\ { RequestInterface, ResponseInterface };
interface AuthenticateInterface
{
  public function login(RequestInterface $request) : 
    ResponseInterface;
}
```

{% hint style="info" %}
请注意，通过定义一个需要符合PSR-7标准的请求并产生符合PSR-7标准的响应的方法，我们已经使这个接口普遍适用。
{% endhint %}

2. 接下来，我们定义实现接口所需的`login()`方法的适配器。我们确保使用适当的类，并定义合适的常量和属性。构造函数使用了`Application\Database\Connection`，它在第5章，与数据库的交互中定义。

```php
namespace Application\Acl;
use PDO;
use Application\Database\Connection;
use Psr\Http\Message\ { RequestInterface, ResponseInterface };
use Application\MiddleWare\ { Response, TextStream };
class DbTable  implements AuthenticateInterface
{
  const ERROR_AUTH = 'ERROR: authentication error';
  protected $conn;
  protected $table;
  public function __construct(Connection $conn, $tableName)
  {
    $this->conn = $conn;
    $this->table = $tableName;
  }
```

3. 核心的`login()`方法从请求对象中提取用户名和密码。然后我们直接进行数据库查询。如果有匹配的信息，我们将用户信息存储在JSON编码的响应体中。

```php
public function login(RequestInterface $request) : 
  ResponseInterface
{
  $code = 401;
  $info = FALSE;
  $body = new TextStream(self::ERROR_AUTH);
  $params = json_decode($request->getBody()->getContents());
  $response = new Response();
  $username = $params->username ?? FALSE;
  if ($username) {
      $sql = 'SELECT * FROM ' . $this->table 
        . ' WHERE email = ?';
      $stmt = $this->conn->pdo->prepare($sql);
      $stmt->execute([$username]);
      $row = $stmt->fetch(PDO::FETCH_ASSOC);
      if ($row) {
          if (password_verify($params->password, 
              $row['password'])) {
                unset($row['password']);
                $body = 
                new TextStream(json_encode($row));
                $response->withBody($body);
                $code = 202;
                $info = $row;
              }
            }
          }
          return $response->withBody($body)->withStatus($code);
        }
      }
```

{% hint style="info" %}
**最佳实践**

永远不要用明文存储密码。当你需要进行密码匹配时，使用`password_verify()`，这样就可以否定重现密码哈希的必要性。
{% endhint %}

4. `Authenticate`类是一个实现`AuthenticationInterface`的适配器类的封装器。相应地，构造函数将一个适配器类作为参数，以及一个作为密钥的字符串，认证信息存储在`$_SESSION`中。

```php
namespace Application\Acl;
use Application\MiddleWare\ { Response, TextStream };
use Psr\Http\Message\ { RequestInterface, ResponseInterface };
class Authenticate
{
  const ERROR_AUTH = 'ERROR: invalid token';
  const DEFAULT_KEY = 'auth';
  protected $adapter;
  protected $token;
  public function __construct(
  AuthenticateInterface $adapter, $key)
  {
    $this->key = $key;
    $this->adapter = $adapter;
  }
```

5. 此外，我们还提供了一个带有安全令牌的登录表单，这有助于防止跨站点请求伪造（CSRF）攻击。

```php
public function getToken()
{
  $this->token = bin2hex(random_bytes(16));
  $_SESSION['token'] = $this->token;
  return $this->token;
}
public function matchToken($token)
{
  $sessToken = $_SESSION['token'] ?? date('Ymd');
  return ($token == $sessToken);
}
public function getLoginForm($action = NULL)
{
  $action = ($action) ? 'action="' . $action . '" ' : '';
  $output = '<form method="post" ' . $action . '>';
  $output .= '<table><tr><th>Username</th><td>';
  $output .= '<input type="text" name="username" /></td>';
  $output .= '</tr><tr><th>Password</th><td>';
  $output .= '<input type="password" name="password" />';
  $output .= '</td></tr><tr><th>&nbsp;</th>';
  $output .= '<td><input type="submit" /></td>';
  $output .= '</tr></table>';
  $output .= '<input type="hidden" name="token" value="';
  $output .= $this->getToken() . '" />';
  $output .= '</form>';
  return $output;
}
```

6. 最后，该类中的`login()`方法会检查`token`是否有效。如果无效，则返回一个400响应。否则，适配器的`login()`方法将被调用。

```php
public function login(
RequestInterface $request) : ResponseInterface
{
  $params = json_decode($request->getBody()->getContents());
  $token = $params->token ?? FALSE;
  if (!($token && $this->matchToken($token))) {
      $code = 400;
      $body = new TextStream(self::ERROR_AUTH);
      $response = new Response($code, $body);
  } else {
      $response = $this->adapter->login($request);
  }
  if ($response->getStatusCode() >= 200
      && $response->getStatusCode() < 300) {
      $_SESSION[$this->key] = 
        json_decode($response->getBody()->getContents());
  } else {
      $_SESSION[$this->key] = NULL;
  }
  return $response;
}

}
```

## 如何运行...

首先，一定要按照附录《定义PSR-7类》中定义的事例。接下来，继续定义本配方中所介绍的类，总结如下表。

| Class | 在这些步骤中 |
| :--- | :--- |
| `Application\Acl\AuthenticateInterface` | 1 |
| `Application\Acl\DbTable` | 2 - 3 |
| `Application\Acl\Authenticate` | 4 - 6 |

然后你可以定义一个`chap_09_middleware_authenticate.php`调用程序，设置自动加载并使用相应的类。

```php
<?php
session_start();
define('DB_CONFIG_FILE', __DIR__ . '/../config/db.config.php');
define('DB_TABLE', 'customer_09');
define('SESSION_KEY', 'auth');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');

use Application\Database\Connection;
use Application\Acl\ { DbTable, Authenticate };
use Application\MiddleWare\ { ServerRequest, Request, Constants, TextStream };
```

现在您可以设置认证适配器和核心类了。

```php
$conn   = new Connection(include DB_CONFIG_FILE);
$dbAuth = new DbTable($conn, DB_TABLE);
$auth   = new Authenticate($dbAuth, SESSION_KEY);
```

一定要对传入的请求进行初始化，并将请求设置为认证类。

```php
$incoming = new ServerRequest();
$incoming->initialize();
$outbound = new Request();
```

检查传入类方法是否为POST。如果是，则向认证类传递一个请求。

```php
if ($incoming->getMethod() == Constants::METHOD_POST) {
  $body = new TextStream(json_encode(
  $incoming->getParsedBody()));
  $response = $auth->login($outbound->withBody($body));
}
$action = $incoming->getServerParams()['PHP_SELF'];
?>
```

显示逻辑是这样的。

```php
<?= $auth->getLoginForm($action) ?>
```

以下是无效验证尝试的输出。注意右边的401状态码。在这个例子中，你可以添加一个响应对象的`var_dump()`。

![](../../.gitbook/assets/image%20%28100%29.png)

这里是一个成功的认证。

![](../../.gitbook/assets/image%20%2897%29.png)

## 更多...

有关如何避免CSRF和其他攻击的指导，请参见第12章《提高网站安全》。

