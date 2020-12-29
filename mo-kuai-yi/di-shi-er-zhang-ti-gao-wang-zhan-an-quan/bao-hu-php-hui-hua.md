# 保护PHP session

PHP的session机制非常简单。一旦会话通过`session_start()`或`php.ini`中的`session.autostart`设置被启动，PHP引擎就会生成一个唯一的标记，默认情况下，这个标记会通过cookie的方式传递给用户。在随后的请求中，当会话仍然被认为是活动的，用户的浏览器（或类似的）会提交会话标识符，通常也是通过cookie的方式，以供检查。然后PHP引擎使用这个标识符在服务器上找到合适的文件，用存储的信息填充`$_SESSION`。当会话标识符是识别一个返回网站的访问者的唯一手段时，就会产生巨大的安全问题。在这个事例中，我们将介绍几种技术，帮助你保护你的 session，反过来，这将大大提高网站的整体安全性。

## 如何做...

1.首先，重要的是要认识到使用session作为唯一的认证手段是多么危险。想象一下，当一个有效用户登录到你的网站时，你在`$_SESSION`中设置了一个`loggedIn`标志。

```php
session_start();
$loggedIn = $_SESSION['isLoggedIn'] ?? FALSE;
if (isset($_POST['login'])) {
  if ($_POST['username'] == // username lookup
      && $_POST['password'] == // password lookup) {
      $loggedIn = TRUE;
      $_SESSION['isLoggedIn'] = TRUE;
  }
}session_start();
$loggedIn = $_SESSION['isLoggedIn'] ?? FALSE;
if (isset($_POST['login'])) {
  if ($_POST['username'] == // username lookup
      && $_POST['password'] == // password lookup) {
      $loggedIn = TRUE;
      $_SESSION['isLoggedIn'] = TRUE;
  }
}
```

2. 在你的程序逻辑中，如果`$_SESSION['isLoggedIn']`被设置为 `TRUE`，将允许用户查看敏感信息。

```php
<br>Secret Info
<br><?php if ($loggedIn) echo // secret information; ?>
```

3. 如果攻击者通过成功地执行**跨站脚本（XSS）**攻击来获取会话标识符，他/她所需要做的就是将`PHPSESSID` cookie的值设置为非法获取的值，然后他们就会被你的应用程序视为一个有效的用户。

4. 缩小`PHPSESSID`有效的时间窗口的一个快速而简单的方法是使用`session_regenerate_id()`。这个非常简单的命令会生成一个新的会话标识符，使旧的标识符失效，保持会话数据的完整性，并且对性能的影响很小。这个命令只能在会话开始后执行。

```php
session_start();
session_regenerate_id();
```

5. 另一个经常被忽视的技术是确保网络访问者有一个注销选项。然而，重要的是，不仅要使用`session_destroy()`来销毁会话，还要取消设置`$_SESSION`数据，并过期会话cookie。

```php
session_unset();
session_destroy();
setcookie('PHPSESSID', 0, time() - 3600);
```

6. 另一种可用于防止会话劫持的简单技术是开发网站访问者的指纹或拇指指纹。实现这种技术的一种方法是收集网站访问者在会话标识符之上的独特信息。这些信息包括用户代理（即浏览器）、接受的语言和远程IP地址。你可以从这些信息中得出一个简单的哈希值，并将哈希值存储在服务器上的一个单独文件中。下次用户访问网站时，如果你已经根据会话信息确定他们已经登录，那么你可以通过匹配指纹进行二次验证。

```php
$remotePrint = md5($_SERVER['REMOTE_ADDR'] 
                   . $_SERVER['HTTP_USER_AGENT'] 
                   . $_SERVER['HTTP_ACCEPT_LANGUAGE']);
$printsMatch = file_exists(THUMB_PRINT_DIR . $remotePrint);
if ($loggedIn && !$printsMatch) {
    $info = 'SESSION INVALID!!!';
    error_log('Session Invalid: ' . date('Y-m-d H:i:s'), 0);
    // take appropriate action
}
```

{% hint style="info" %}
我们使用`md5()`，因为它是一种快速散列算法，非常适合内部使用。我们不建议将`md5()`用于任何外部用途，因为它会受到暴力攻击。
{% endhint %}

## 如何运行...

为了演示 session 是如何受到攻击的，编写一个简单的登录脚本，在成功登录时设置`$_SESSION['isLoggedIn']`标志。你可以调用文件`chap_12_session_hijack.php`。

```php
session_start();
$loggedUser = $_SESSION['loggedUser'] ?? '';
$loggedIn = $_SESSION['isLoggedIn'] ?? FALSE;
$username = 'test';
$password = 'password';
$info = 'You Can Now See Super Secret Information!!!';

if (isset($_POST['login'])) {
  if ($_POST['username'] == $username
      && $_POST['password'] == $password) {
        $loggedIn = TRUE;
        $_SESSION['isLoggedIn'] = TRUE;
        $_SESSION['loggedUser'] = $username;
        $loggedUser = $username;
  }
} elseif (isset($_POST['logout'])) {
  session_destroy();
}
```

然后你可以添加代码，显示一个简单的登录表格。要测试会话漏洞，请使用我们刚刚创建的`chap_12_session_hijack.php`文件按照这个步骤进行。

1. 改为包含文件的目录
2. 运行`php -S localhost:8080`命令
3. 用一个浏览器，打开网址`http://localhost:8080/<文件名>`
4. 以用户 `test` 身份登录，密码为`password`
5. 你现在可以看到超级秘密信息了！！
6. 刷新页面：每次，你应该看到一个新的会话标识符
7. 复制`PHPSESSID` cookie的值
8. 打开另一个浏览器到同一网页
9. 通过复制`PHPSESSID`的值来修改浏览器发送的cookie

为了说明问题，我们还显示了`$_COOKIE`和`$_SESSION`的值，如下图所示，使用Vivaldi浏览器的截图。

![](../../.gitbook/assets/image%20%28145%29.png)

然后，我们复制`PHPSESSID`的值，打开火狐浏览器，用一个叫Tamper Data的工具来修改cookie的值。

![](../../.gitbook/assets/image%20%28147%29.png)

在接下来的截图中可以看到，我们现在是一个经过认证的用户，无需输入用户名和密码。

![](../../.gitbook/assets/image%20%28150%29.png)

现在你可以实现前面步骤中讨论的变化。复制之前创建的文件到`chap_12_session_protected.php`。现在开始重新生成会话ID:

```php
<?php
define('THUMB_PRINT_DIR', __DIR__ . '/../data/');
session_start();
session_regenerate_id();
```

接下来，初始化变量并确定登录状态（如前）。

```php
$username = 'test';
$password = 'password';
$info = 'You Can Now See Super Secret Information!!!';
$loggedIn = $_SESSION['isLoggedIn'] ?? FALSE;
$loggedUser = $_SESSION['user'] ?? 'guest';
```

您可以使用远程地址、用户代理和语言设置添加会话指纹。

```php
$remotePrint = md5($_SERVER['REMOTE_ADDR']
  . $_SERVER['HTTP_USER_AGENT']
  . $_SERVER['HTTP_ACCEPT_LANGUAGE']);
$printsMatch = file_exists(THUMB_PRINT_DIR . $remotePrint);
```

如果登录成功，我们会在会话中存储拇指指纹信息和登录状态。

```php
if (isset($_POST['login'])) {
  if ($_POST['username'] == $username
      && $_POST['password'] == $password) {
        $loggedIn = TRUE;
        $_SESSION['user'] = strip_tags($username);
        $_SESSION['isLoggedIn'] = TRUE;
        file_put_contents(
          THUMB_PRINT_DIR . $remotePrint, $remotePrint);
  }
```

你也可以检查注销选项，并实现一个正确的注销程序：取消设置`$_SESSION`变量，使session无效，并使cookie过期。你也可以删除拇指印文件并实现重定向。

```php
} elseif (isset($_POST['logout'])) {
  session_unset();
  session_destroy();
  setcookie('PHPSESSID', 0, time() - 3600);
  if (file_exists(THUMB_PRINT_DIR . $remotePrint)) 
    unlink(THUMB_PRINT_DIR . $remotePrint);
    header('Location: ' . $_SERVER['REQUEST_URI'] );
  exit;
```

否则，如果不是登录或注销的操作，可以检查是否认为用户已经登录，如果拇指指纹不匹配，则认为会话无效，并采取相应的操作。

```php
} elseif ($loggedIn && !$printsMatch) {
    $info = 'SESSION INVALID!!!';
    error_log('Session Invalid: ' . date('Y-m-d H:i:s'), 0);
    // take appropriate action
}
```

现在你可以使用新的`chap_12_session_protected.php`文件运行与之前提到的相同的过程。你会注意到的第一件事是，会话现在被认为是无效的。输出结果看起来像这样:

![](../../.gitbook/assets/image%20%28162%29.png)

这是因为您现在使用的是不同的浏览器，所以拇指印不匹配。同样，如果您刷新第一个浏览器的页面，会话标识符将被重新生成，使任何先前复制的标识符过时。最后，注销按钮将完全清除会话信息。

## 更多...

关于网站漏洞的优秀概述，请参考https://www.owasp.org/index.php/Category:Vulnerability。

有关会话劫持的信息，请参考https://www.owasp.org/index.php/Session\_hijacking\_attack。

