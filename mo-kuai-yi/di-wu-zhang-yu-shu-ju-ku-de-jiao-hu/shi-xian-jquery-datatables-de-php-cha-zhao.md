# 实现jQuery DataTables的PHP查找

二次查找的另一种方法是由前端生成请求。在这个示例中，我们将对前一个示例 "将二次查找嵌入到查询结果 "中介绍的二次查找代码稍作修改。在前面的示例中，即使视图逻辑在执行查找，所有的处理仍然在服务器上完成。然而，当使用**jQuery DataTables**时，二次查询实际上是由客户端直接执行的，以浏览器发出的**异步JavaScript**和**XML（AJAX）**请求的形式。

## 如何做...

1.首先，我们需要将二次查找逻辑（在上面的示例中讨论过）分拆到一个单独的PHP文件中。这个新脚本的目的是执行二次查找并返回一个JSON数组。

2.新的脚本我们将调用`chap_05_jquery_datatables_php_lookups_ajax.php`。注意到`SELECT`语句非常明确的规定了哪些列会被传递。你也会注意到获取模式已经改为`PDO::FETCH_NUM`。你可能还注意到最后一行会将结果分配给一个JSON编码的数组中的数据键。

{% hint style="info" %}
在处理零配置的jQuery DataTables时，只返回与表头匹配的列数是极其重要的。
{% endhint %}

```php
$id  = $_GET['id'] ?? 0;
sql = 'SELECT u.transaction,u.date, u.quantity,u.sale_price,r.title '
   . 'FROM purchases AS u '
   . 'JOIN products AS r '
   . 'ON u.product_id = r.id '
   . 'WHERE u.customer_id = :id';
$stmt = $conn->pdo->prepare($sql);
$stmt->execute(['id' => (int) $id]);
$results = array();
while ($row = $stmt->fetch(PDO::FETCH_NUM)) {
  $results[] = $row;
}
echo json_encode(['data' => $results]); 

```

3. 接下来，我们需要修改按ID检索客户信息的函数，去掉之前示例中嵌入的二次查找。

```php
function findCustomerById($id, Connection $conn)
{
  $stmt = $conn->pdo->query(
    'SELECT * FROM customer WHERE id = ' . (int) $id);
  $results = $stmt->fetch(PDO::FETCH_ASSOC);
  return $results;
}
```

4. 之后，在视图逻辑中，我们导入最低限度的jQuery、DataTables和样式表来实现零配置。至少，你需要jQuery本身（在本例中`jquery-1.12.0.min.js`）和DataTables（`jquery.dataTables.js`）。我们还添加了一个与DataTables相关的方便的样式表，`jquery.dataTables.css`。

```php
<!DOCTYPE html>
<head>
  <script src="https://code.jquery.com/jquery-1.12.0.min.js">
  </script>
    <script type="text/javascript" 
      charset="utf8" 
      src="//cdn.datatables.net/1.10.11/js/jquery.dataTables.js">
    </script>
  <link rel="stylesheet" 
    type="text/css" 
    href="//cdn.datatables.net/1.10.11/css/jquery.dataTables.css">
</head>
```

5.然后我们定义一个jQuery document `ready`函数，将一个表与DataTables关联起来。在这种情况下，我们给表元素分配一个id属性`customerTable`，它将被分配给DataTables。你还会注意到，我们将AJAX数据源指定为步骤1中定义的脚本，`chap_05_jquery_datatables_php_lookups_ajax.php`。由于我们有`$id`可用，这将被附加到数据源URL中。

```php
<script>
$(document).ready(function() {
  $('#customerTable').DataTable(
    { "ajax": '/chap_05_jquery_datatables_php_lookups_ajax.php?id=<?= $id ?>' 
  });
} );
</script>
```

6. 在视图逻辑的主体中，我们定义了表，确保id属性与前面代码中指定的属性相匹配。我们还需要定义头部，以匹配响应AJAX请求时呈现的数据。

```php
<table id="customerTable" class="display" cellspacing="0" width="100%">
  <thead>
    <tr>
      <th>Transaction</th>
      <th>Date</th>
      <th>Qty</th>
      <th>Price</th>
      <th>Product</th>
    </tr>
  </thead>
</table>
```

7. 现在，剩下要做的就是加载页面，选择客户ID（本例中是随机的），让jQuery进行二次查找的请求。

## 如何运行...

创建一个`chap_05_jquery_datatables_php_lookups_ajax.php`脚本，它将响应AJAX请求。在里面放置初始化自动加载的代码，并创建一个`Connection`实例。然后，你可以附加前面示例第2步中的代码。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
include __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
```

接下来，创建一个`chap_05_jquery_datatables_php_lookups.php`调用程序，随机提取客户的信息。添加前面代码第3步中描述的函数。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
include __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
// 在这里添加 findCustomerById()
$id     = random_int(1,79);
$result = findCustomerById($id, $conn);
?>
```

调用程序还将包含导入最小JavaScript实现jQuery DataTables的视图逻辑。你可以添加前面代码第3步所示的代码。然后，添加步骤5和步骤6中所示的文档 `ready` 功能和显示逻辑。下面是输出结果。

![](../../.gitbook/assets/image%20%2870%29.png)

## 更多

有关jQuery的更多信息，请访问他们的网站[https://jquery.com/](https://jquery.com/)。要阅读关于jQuery的DataTables插件，请参考本文[https://www.datatables.net/](https://www.datatables.net/)。零配置数据表的讨论，请访问[https://datatables.net/examples/basic\_init/zero\_configuration.html](https://datatables.net/examples/basic_init/zero_configuration.html)。关于AJAX源数据的更多信息，请看[https://datatables.net/examples/data\_sources/ajax.html](https://datatables.net/examples/data_sources/ajax.html)。

