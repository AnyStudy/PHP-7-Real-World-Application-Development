# 显示多维数组并累计总数

如何正确显示多维数组中的数据，一直是任何一个Web开发人员的经典问题。为了说明问题，假设你希望显示一个客户和他们的购买清单。对于每个客户，你希望显示他们的姓名、电话号码、账户余额等。这已经代表了一个二维数组，其中x轴代表客户，y轴代表该客户的数据。现在加入购买的数据，你就有了第三个轴! 如何在二维屏幕上表示一个三维模型？一个可能的解决方案是将 "隐藏 "的分割标签与一个简单的JavaScript可见性切换结合起来。

## 如何做...

1.首先，我们需要从一个使用了许多`JOIN`子句的SQL语句中生成一个3D数组。我们将使用第1章 《建立基础》中介绍的 `Application/Database/Connection` 来制定一个合适的SQL查询。为了支持分页，我们开放了两个参数：`min`和`max`。不幸的是，在这种情况下，我们不能使用简单的`LIMIT`和`OFFSET`，因为行数会根据任何给定客户的购买数量而变化。相应地，我们可以通过对客户ID的限制来限制行数，这个限制大概（希望）是递增的。为了使这个工作正常进行，我们还需要将主`order` 设置为customer ID。

```php
define('ITEMS_PER_PAGE', 6);
define('SUBROWS_PER_PAGE', 6);
define('DB_CONFIG_FILE', '/../config/db.config.php');
include __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
$sql  = 'SELECT c.id,c.name,c.balance,c.email,f.phone, '
  . 'u.transaction,u.date,u.quantity,u.sale_price,r.title '
  . 'FROM customer AS c '
  . 'JOIN profile AS f '
  . 'ON f.id = c.id '
  . 'JOIN purchases AS u '
  . 'ON u.customer_id = c.id '
  . 'JOIN products AS r '
  . 'ON u.product_id = r.id '
  . 'WHERE c.id >= :min AND c.id < :max '
  . 'ORDER BY c.id ASC, u.date DESC ';
```

2. 接下来我们可以根据对客户ID的限制，使用简单的`$_GET`参数实现一种分页形式。请注意，我们添加了一个额外的检查，以确保`$prev`的值不会低于零。你可以考虑添加另一个控件，确保`$next`的值不超过最后一个客户ID。在本例中，我们只允许它递增。

```php
$page = $_GET['page'] ?? 1;
$page = (int) $page;
$next = $page + 1;
$prev = $page - 1;
$prev = ($prev >= 0) ? $prev : 0;
```

3. 然后我们计算`$min`和`$max`的值，并准备和执行SQL语句。

```php
$min  = $prev * ITEMS_PER_PAGE;
$max  = $page * ITEMS_PER_PAGE;
$stmt = $conn->pdo->prepare($sql);
$stmt->execute(['min' => $min, 'max' => $max]);
```

4. 可以使用`while()`循环来获取结果。在这个例子中，我们使用`PDO::FETCH_ASSOC`的简单获取模式。使用客户ID作为键，我们将客户的基本信息存储为数组参数。然后我们在一个子数组中存储一个购买信息的数组，`$results[$key]['purchases'][]`。当客户ID发生变化时，就是一个信号，为下一个客户存储同样的信息。需要注意的是，我们将每个客户的总数累积在数组key total中。

```php
$custId = 0;
$result = array();
$grandTotal = 0.0;
while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
  if ($row['id'] != $custId) {
    $custId = $row['id'];
    $result[$custId] = [
      'name'    => $row['name'],
      'balance' => $row['balance'],
      'email'   => $row['email'],
      'phone'   => $row['phone'],
    ];
    $result[$custId]['total'] = 0;
  }
  $result[$custId]['purchases'][] = [
    'transaction' => $row['transaction'],
    'date'        => $row['date'],
    'quantity'    => $row['quantity'],
    'sale_price'  => $row['sale_price'],
    'title'       => $row['title'],
  ];
  $result[$custId]['total'] += $row['sale_price'];
  $grandTotal += $row['sale_price'];
}
?>
```

5. 接下来我们实现视图逻辑。首先，我们从一个显示主要客户信息的块开始。

```php
<div class="container">
<?php foreach ($result as $key => $data) : ?>
<div class="mainLeft color0">
    <?= $data['name'] ?> [<?= $key ?>]
</div>
<div class="mainRight">
  <div class="row">
    <div class="left">Balance</div>
           <div class="right"><?= $data['balance']; ?></div>
  </div>
  <div class="row">
    <div class="left color2">Email</div>
           <div class="right"><?= $data['email']; ?></div>
  </div>
  <div class="row">
    <div class="left">Phone</div>
           <div class="right"><?= $data['phone']; ?></div>
    </div>
  <div class="row">
        <div class="left color2">Total Purchases</div>
    <div class="right">
<?= number_format($data['total'],2); ?>
</div>
  </div>
```

6. 接下来是显示这个客户的购买清单的逻辑。

```php
<!-- Purchases Info -->
<table>
  <tr>
  <th>Transaction</th><th>Date</th><th>Qty</th>
   <th>Price</th><th>Product</th>
  </tr>
  <?php $count  = 0; ?>
  <?php foreach ($data['purchases'] as $purchase) : ?>
  <?php $class = ($count++ & 01) ? 'color1' : 'color2'; ?>
  <tr>
  <td class="<?= $class ?>"><?= $purchase['transaction'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['date'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['quantity'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['sale_price'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['title'] ?></td>
  </tr>
  <?php endforeach; ?>
</table>
```

7. 为了分页的目的，我们添加按钮来代表上一个和下一个。

```php
<?php endforeach; ?>
<div class="container">
  <a href="?page=<?= $prev ?>">
        <input type="button" value="Previous"></a>
  <a href="?page=<?= $next ?>">
        <input type="button" value="Next" class="buttonRight"></a>
</div>
<div class="clearRow"></div>
</div>
```

8. 遗憾的是，到目前为止，结果还远没有达到整洁的程度。因此，我们添加了一个简单的JavaScript函数来根据`id`属性来切换`<div>`标签的可见性。

```php
<script type="text/javascript">
function showOrHide(id) {
  var div = document.getElementById(id);
  div.style.display = div.style.display == "none" ? "block" : "none";
}
</script>
```

9. 接下来，我们将Purchases表包装在最初不可见的 `<div>` 标签内。 然后，我们可以限制最初可见的子行数，并添加一个链接来显示剩余的购买数据。

```php
<div class="row" id="<?= 'purchase' . $key ?>" style="display:none;">
  <table>
    <tr>
      <th>Transaction</th><th>Date</th><th>Qty</th>
                 <th>Price</th><th>Product</th>
    </tr>
  <?php $count  = 0; ?>
  <?php $first  = TRUE; ?>
  <?php foreach ($data['purchases'] as $purchase) : ?>
    <?php if ($count > SUBROWS_PER_PAGE && $first) : ?>
    <?php     $first = FALSE; ?>
    <?php     $subId = 'subrow' . $key; ?>
    </table>
    <a href="#" onClick="showOrHide('<?= $subId ?>')">More</a>
    <div id="<?= $subId ?>" style="display:none;">
    <table>
    <?php endif; ?>
  <?php $class = ($count++ & 01) ? 'color1' : 'color2'; ?>
  <tr>
  <td class="<?= $class ?>"><?= $purchase['transaction'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['date'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['quantity'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['sale_price'] ?></td>
  <td class="<?= $class ?>"><?= $purchase['title'] ?></td>
  </tr>
  <?php endforeach; ?>
  </table>
  <?php if (!$first) : ?></div><?php endif; ?>
</div>
```

10. 然后我们添加一个按钮，当点击时，就会显示出隐藏的`<div>`标签。

```php
<input type="button" value="Purchases" class="buttonRight" 
    onClick="showOrHide('<?= 'purchase' . $key ?>')">
```

## 如何运行...

将步骤1至步骤5中描述的代码放入 `chap_10_html_table_multi_array_hidden.php` 文件中。

在`while()`循环中添加以下内容。

```php
printf('%6s : %20s : %8s : %20s' . PHP_EOL, 
    $row['id'], $row['name'], $row['transaction'], $row['title']);
```

就在 `while()` 循环之后，添加一条退出命令。下面是输出结果。

![](../../.gitbook/assets/image%20%28129%29.png)

你会注意到，基本的客户信息，如ID和姓名，在每条结果行中都会重复，但购买信息，如交易和产品名称，则有所不同。继续删除`printf()`语句。

将退出命令改为：

```php
echo '<pre>', var_dump($result), '</pre>'; exit;
```

下面是新组成的3D阵列的样子。

![](../../.gitbook/assets/image%20%28118%29.png)

现在可以添加步骤5至7中所示的显示逻辑。如前所述，虽然你现在显示的是所有数据，但可视化显示并没有什么帮助。现在继续添加剩余步骤中提到的细化内容。下面是初始输出可能出现的情况。

![](../../.gitbook/assets/image%20%28132%29.png)

当点击 "购买 "按钮时，会出现初始购买信息。如果点击 "更多 "链接，则显示剩余的购买信息。

![](../../.gitbook/assets/image%20%28126%29.png)

