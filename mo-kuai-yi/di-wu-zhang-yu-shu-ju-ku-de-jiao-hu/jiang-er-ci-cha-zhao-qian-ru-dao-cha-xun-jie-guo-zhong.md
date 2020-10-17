# 将二次查找嵌入到查询结果中

在实现实体类之间关系的道路上，我们先来看看如何嵌入执行二次查找所需的代码。这样的一个例子是，在显示客户信息时，让视图逻辑执行二次查找，得到该客户的购买清单。

{% hint style="info" %}
这种方法的优点是，处理被推迟到实际的视图逻辑执行。这将最终平滑性能曲线，在最初的客户信息查询和后来的购买信息查询之间，工作量分配得更加均匀。另一个好处是，通过其固有的冗余数据避免了大规模的`JOIN`。
{% endhint %}

## 如何做...

1.首先，定义一个根据客户ID找到客户的函数。为了这个说明的目的，我们将简单地使用获取模式`PDO::FETCH_ASSOC`来获取一个数组。我们还将继续使用第一章建立基础中讨论的`Application\Database\Connection`类。

```php
function findCustomerById($id, Connection $conn)
{
  $stmt = $conn->pdo->query(
    'SELECT * FROM customer WHERE id = ' . (int) $id);
  $results = $stmt->fetch(PDO::FETCH_ASSOC);
  return $results;
}
```

2.接下来，我们分析一下购买表，看看客户表和产品表是如何连接的。从这个表的`CREATE`语句可以看出，`customer_id`和`product_id`外键构成了关系。

```php
CREATE TABLE 'purchases' (
  'id' int(11) NOT NULL AUTO_INCREMENT,
  'transaction' varchar(8) NOT NULL,
  'date' datetime NOT NULL,
  'quantity' int(10) unsigned NOT NULL,
  'sale_price' decimal(8,2) NOT NULL,
  'customer_id' int(11) DEFAULT NULL,
  'product_id' int(11) DEFAULT NULL,
  PRIMARY KEY ('id'),
  KEY 'IDX_C3F3' ('customer_id'),
  KEY 'IDX_665A' ('product_id'),
  CONSTRAINT 'FK_665A' FOREIGN KEY ('product_id') 
  REFERENCES 'products' ('id'),
  CONSTRAINT 'FK_C3F3' FOREIGN KEY ('customer_id') 
  REFERENCES 'customer' ('id')
);
```

3.现在我们扩展原来的`findCustomerById()`函数，以匿名函数的形式定义二次查找，然后可以在视图脚本中执行。匿名函数被分配到`$results['purchases'`\]元素中。

```php
function findCustomerById($id, Connection $conn)
{
  $stmt = $conn->pdo->query(
       'SELECT * FROM customer WHERE id = ' . (int) $id);
  $results = $stmt->fetch(PDO::FETCH_ASSOC);
  if ($results) {
    $results['purchases'] = 
      // 定义二级查询
      function ($id, $conn) {
        $sql = 'SELECT * FROM purchases AS u '
           . 'JOIN products AS r '
           . 'ON u.product_id = r.id '
           . 'WHERE u.customer_id = :id '
           . 'ORDER BY u.date';
          $stmt = $conn->pdo->prepare($sql);
          $stmt->execute(['id' => $id]);
          while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            yield $row;
          }
      };
  }
  return $results;
}
```

4.假设我们已经成功地将客户信息检索到`$results`数组中，在视图逻辑中，我们需要做的就是循环执行匿名函数的返回值。在本例中，我们随机检索客户信息。

```php
$result = findCustomerById(rand(1,79), $conn);
```

5. 在视图逻辑中，我们循环查看二次查找返回的结果。在下面的代码中突出显示了对嵌入式匿名函数的调用。

```php
<table>
  <tr>
<th>Transaction</th><th>Date</th><th>Qty</th>
<th>Price</th><th>Product</th>
  </tr>
<?php 
foreach ($result['purchases']($result['id'], $conn) as $purchase) : ?>
  <tr>
    <td><?= $purchase['transaction'] ?></td>
    <td><?= $purchase['date'] ?></td>
    <td><?= $purchase['quantity'] ?></td>
    <td><?= $purchase['sale_price'] ?></td>
    <td><?= $purchase['title'] ?></td>
  </tr>
<?php endforeach; ?>
</table>
```

## 如何运行...

创建一个`chap_05_secondary_lookups.php`调用程序，并插入需要创建`Application\Database\Connection`实例的代码。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
include __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
```

接下来，添加步骤3中所示的`findCustomerById()`函数。然后就可以随机抽取一个客户的信息，结束调用程序的PHP部分。

```php
function findCustomerById($id, Connection $conn)
{
  // 上文第3点所示代码
}
$result = findCustomerById(rand(1,79), $conn);
?>
```

对于视图逻辑，可以显示核心客户信息，如前面几个示例所示。

```php
<h1><?= $result['name'] ?></h1>
<div class="row">
<div class="left">Balance</div>
<div class="right"><?= $result['balance']; ?></div>
</div>
<!-- etc.l -->
```

你可以这样显示购买的信息。

```php
<table>
<tr><th>Transaction</th><th>Date</th><th>Qty</th>
<th>Price</th><th>Product</th></tr>
  <?php 
  foreach ($result['purchases']($result['id'], $conn) as $purchase) : ?>
  <tr>
    <td><?= $purchase['transaction'] ?></td>
    <td><?= $purchase['date'] ?></td>
    <td><?= $purchase['quantity'] ?></td>
    <td><?= $purchase['sale_price'] ?></td>
    <td><?= $purchase['title'] ?></td>
  </tr>
<?php endforeach; ?>
</table>
```

关键的部分是，通过调用内嵌的匿名函数`$result['purchases']($result['id'], $conn)`，将二次查找作为视图逻辑的一部分来执行。下面是输出结果。

![](../../.gitbook/assets/image%20%2875%29.png)

