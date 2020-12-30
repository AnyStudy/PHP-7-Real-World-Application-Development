# 用令牌保护表格的安全

这个事例介绍了另一种非常简单的技术，它可以保护你的表单免受跨站点请求伪造（CSRF）攻击。简单地说，当攻击者可能使用其他技术感染您网站上的一个网页时，CSRF攻击是可能的。在大多数情况下，受感染的网页会使用有效的登录用户的凭证开始发出请求（即使用JavaScript购买商品，或进行设置更改）。你的应用程序要检测到这种活动是非常困难的。一个可以轻松采取的措施是生成一个随机令牌，包含在每个要提交的表单中。由于受感染的页面将无法访问该令牌，也无法生成匹配的令牌，因此表单验证将失败。

## 如何做...

1.首先，为了演示这个问题，我们创建一个网页，模拟一个受感染的页面，生成一个请求，向数据库发布一个条目。在这个说明中，我们将调用文件`chap_12_form_csrf_test_unprotected.html`。

```php
<!DOCTYPE html>
  <body onload="load()">
  <form action="/chap_12_form_unprotected.php" 
    method="post" id="csrf_test" name="csrf_test">
    <input name="name" type="hidden" value="No Goodnick" />
    <input name="email" type="hidden" value="malicious@owasp.org" />
    <input name="comments" type="hidden" 
       value="Form is vulnerable to CSRF attacks!" />
    <input name="process" type="hidden" value="1" />
  </form>
  <script>
    function load() { document.forms['csrf_test'].submit(); }
  </script>
</body>
</html>
```

2. 接下来，我们创建一个名为`chap_12_form_unprotected.php`的脚本来响应表单发布。与本书中的其他调用程序一样，我们设置了自动加载，并使用第5章 "与数据库的交互 "中所涉及的`Application\Database\Connection`类。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
```

3. 然后，我们检查进程按钮是否被按下，甚至实现过滤机制，这在本章的过滤`$_POST`数据事例中有所涉及。这是为了证明CSRF攻击很容易绕过过滤器。

```php
if ($_POST['process']) {
    $filter = [
      'trim' => function ($item) { return trim($item); },
      'email' => function ($item) { 
        return filter_var($item, FILTER_SANITIZE_EMAIL); },
      'length' => function ($item, $length) { 
        return substr($item, 0, $length); },
      'stripTags' => function ($item) { 
      return strip_tags($item); },
  ];

  $assignments = [
    '*'         => ['trim' => NULL, 'stripTags' => NULL],
    'email'   => ['length' => 249, 'email' => NULL],
    'name'    => ['length' => 128],
    'comments'=> ['length' => 249],
  ];

  $data = $_POST;
  foreach ($data as $field => $item) {
    foreach ($assignments['*'] as $key => $option) {
      $item = $filter[$key]($item, $option);
    }
    if (isset($assignments[$field])) {
      foreach ($assignments[$field] as $key => $option) {
        $item = $filter[$key]($item, $option);
      }
      $filteredData[$field] = $item;
    }
  }
```

4. 最后，我们使用准备好的语句将过滤后的数据插入到数据库中。然后我们重定向到另一个脚本，叫做`chap_12_form_view_results.php`，它只是简单地转储`visitors`表的内容。

```php
try {
    $filteredData['visit_date'] = date('Y-m-d H:i:s');
    $sql = 'INSERT INTO visitors '
        . ' (email,name,comments,visit_date) '
        . 'VALUES (:email,:name,:comments,:visit_date)';
    $insertStmt = $conn->pdo->prepare($sql);
    $insertStmt->execute($filteredData);
} catch (PDOException $e) {
    echo $e->getMessage();
}
}
header('Location: /chap_12_form_view_results.php');
exit;
```

5. 结果当然是允许攻击，尽管有过滤和使用准备好的语句。

6. 实现表单保护令牌其实很简单！首先，你需要生成令牌并存储在会话中。首先，你需要生成令牌并将其存储在会话中。我们利用新的`random_bytes()` PHP 7 函数来生成一个真正的随机令牌，这个令牌很难，甚至不可能被攻击者匹配。

```php
session_start();
$token = urlencode(base64_encode((random_bytes(32))));
$_SESSION['token'] = $token;
```

{% hint style="info" %}
`random_bytes()`的输出是二进制的，我们使用`base64_encode()`将其转换为可用的字符串。我们使用`base64_encode()`将其转换为可用的字符串。然后我们使用`urlencode()`进一步处理它，使它正确地呈现在HTML表格中。
{% endhint %}

7. 当我们渲染表单时，我们将标记作为一个隐藏的字段显示出来。

```php
<input type="hidden" name="token" value="<?= $token ?>" />
```

8. 然后我们复制并修改之前提到的`chap_12_form_unprotected.php`脚本，添加逻辑来检查是否与存储在会话中的token匹配。请注意，我们取消设置当前的token，使其在将来使用时无效。我们调用新脚本 `chap_12_form_protected_with_token.php`。

```php
if ($_POST['process']) {
    $sessToken = $_SESSION['token'] ?? 1;
    $postToken = $_POST['token'] ?? 2;
    unset($_SESSION['token']);
    if ($sessToken != $postToken) {
        $_SESSION['message'] = 'ERROR: token mismatch';
    } else {
        $_SESSION['message'] = 'SUCCESS: form processed';
        // continue with form processing
    }
}
```

## 如何运行...

要测试受感染的网页如何发起CSRF攻击，请创建以下文件，如前面的事例所示。

* `chap_12_form_csrf_test_unprotected.html`
* `chap_12_form_unprotected.php`

然后你可以定义一个名为`chap_12_form_view_results.php`的文件，用来转储`visitors`表。

```php
<?php
session_start();
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
$message = $_SESSION['message'] ?? '';
unset($_SESSION['message']);
$stmt = $conn->pdo->query('SELECT * FROM visitors');
?>
<!DOCTYPE html>
<body>
<div class="container">
  <h1>CSRF Protection</h1>
  <h3>Visitors Table</h3>
  <?php while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) : ?>
  <pre><?php echo implode(':', $row); ?></pre>
  <?php endwhile; ?>
  <?php if ($message) : ?>
  <b><?= $message; ?></b>
  <?php endif; ?>
</div>
</body>
</html>
```

在浏览器中，启动`chap_12_form_csrf_test_unprotected.html`。以下是输出结果的显示方式。

![](../../.gitbook/assets/image%20%28161%29.png)

正如你所看到的，尽管有过滤和使用准备好的声明，攻击还是成功了!

接下来，将`chap_12_form_unprotected.php`文件复制到`chap_12_form_protected.php`中。进行事例中第8步的修改。还需要修改测试HTML文件，将`chap_12_form_csrf_test_unprotected.html`复制到`chap_12_form_csrf_test_protected.html`。将`FORM`标签中的action参数的值修改如下。

```php
<form action="/chap_12_form_protected_with_token.php" 
  method="post" id="csrf_test" name="csrf_test">
```

当你在浏览器中运行新的HTML文件时，它会调用`chap_12_form_protected.php`，寻找一个不存在的标记。这里是预期的输出。

![](../../.gitbook/assets/image%20%28196%29.png)

最后，继续定义一个名为`chap_12_form_protected.php`的文件，生成一个token，并将其显示为一个隐藏的元素。

```php
<?php
session_start();
$token = urlencode(base64_encode((random_bytes(32))));
$_SESSION['token'] = $token;
?>
<!DOCTYPE html>
<body onload="load()">
<div class="container">
<h1>CSRF Protected Form</h1>
<form action="/chap_12_form_protected_with_token.php" 
     method="post" id="csrf_test" name="csrf_test">
<table>
<tr><th>Name</th><td><input name="name" type="text" /></td></tr>
<tr><th>Email</th><td><input name="email" type="text" /></td></tr>
<tr><th>Comments</th><td>
<input name="comments" type="textarea" rows=4 cols=80 />
</td></tr>
<tr><th>&nbsp;</th><td>
<input name="process" type="submit" value="Process" />
</td></tr>
</table>
<input type="hidden" name="token" value="<?= $token ?>" />
</form>
<a href="/chap_12_form_view_results.php">
    CLICK HERE</a> to view results
</div>
</body>
</html>
```

当我们从表单中显示并提交数据时，令牌会被验证，并允许继续插入数据，如图所示。

![](../../.gitbook/assets/image%20%28183%29.png)

## 更多...

有关CSFR攻击的更多信息，请参考https://www.owasp.org/index.php/Cross-Site\_Request\_Forgery\_\(CSRF\)。

