# 实现对象关系映射

有两种主要技术可以实现对象之间的关系映射。第一种技术是将相关的子对象预先加载到父对象中。这种方法的优点是容易实现，所有的父子信息都可以立即获得。缺点是可能会消耗大量的内存，而且性能曲线是倾斜的。

第二种技术是将二次查找嵌入到父对象中。在后一种方法中，当你需要访问子对象时，你会运行一个执行二次查找的getter。这种方法的优点是，性能需求被分散在整个请求周期中，而且内存的使用（或可以）更容易管理。这种方法的缺点是，会产生更多的查询，这意味着数据库服务器的工作更多。

{% hint style="info" %}
不过，请注意，我们将说明如何利用准备好的报表来大大抵消这一不利因素。
{% endhint %}

## 如何做...

我们来看一下实现对象关系映射的两种技术。

### 技巧\#1——预加载所有子信息

首先，我们将讨论如何通过将所有子类信息预加载到父类中来实现对象关系映射。在这个例子中，我们将使用三个相关的数据库表，即`customer`、`purchases`和`product`。

1.我们将使用现有的`Application\Entity\Customer`类（在第5章与数据库的交互中，在定义实体类以匹配数据库表的示例中定义）作为模型来开发`Application\Entity\Purchase`类。与之前一样，我们将使用数据库定义作为实体类定义的基础。下面是 `purchases` 的数据库定义。

```sql
CREATE TABLE `purchases` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `transaction` varchar(8) NOT NULL,
  `date` datetime NOT NULL,
  `quantity` int(10) unsigned NOT NULL,
  `sale_price` decimal(8,2) NOT NULL,
  `customer_id` int(11) DEFAULT NULL,
  `product_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `IDX_C3F3` (`customer_id`),
  KEY `IDX_665A` (`product_id`),
  CONSTRAINT `FK_665A` FOREIGN KEY (`product_id`) REFERENCES `products` (`id`),
  CONSTRAINT `FK_C3F3` FOREIGN KEY (`customer_id`) REFERENCES `customer` (`id`)
);
```

2.基于`customer`实体类，以下是`Application\Entity\Purchase`的样子。请注意，并不是所有的`getter`和`setter`都被显示出来。

```php
namespace Application\Entity;

class Purchase extends Base
{

  const TABLE_NAME = 'purchases';
  protected $transaction = '';
  protected $date = NULL;
  protected $quantity = 0;
  protected $salePrice = 0.0;
  protected $customerId = 0;
  protected $productId = 0;

  protected $mapping = [
    'id'            => 'id',
    'transaction'   => 'transaction',
    'date'          => 'date',
    'quantity'      => 'quantity',
    'sale_price'    => 'salePrice',
    'customer_id'   => 'customerId',
    'product_id'    => 'productId',
  ];

  public function getTransaction() : string
  {
    return $this->transaction;
  }
  public function setTransaction($transaction)
  {
    $this->transaction = $transaction;
  }
  // NOTE: other getters / setters are not shown here
}
```

3.现在我们准备定义`Application\Entity\Product`。这里是`product`的数据库定义。

```sql
CREATE TABLE `products` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `sku` varchar(16) DEFAULT NULL,
  `title` varchar(255) NOT NULL,
  `description` varchar(4096) DEFAULT NULL,
  `price` decimal(10,2) NOT NULL,
  `special` int(11) NOT NULL,
  `link` varchar(128) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UNIQ_38C4` (`sku`)
);
```

4.基于`customer`实体类，下面是`Application\Entity\Product`的样子。

```sql
namespace Application\Entity;

class Product extends Base
{

  const TABLE_NAME = 'products';
  protected $sku = '';
  protected $title = '';
  protected $description = '';
  protected $price = 0.0;
  protected $special = 0;
  protected $link = '';

  protected $mapping = [
    'id'          => 'id',
    'sku'         => 'sku',
    'title'       => 'title',
    'description' => 'description',
    'price'       => 'price',
    'special'     => 'special',
    'link'        => 'link',
  ];

  public function getSku() : string
  {
    return $this->sku;
  }
  public function setSku($sku)
  {
    $this->sku = $sku;
  }
  // NOTE: other getters / setters are not shown here
}
```

5.接下来，我们需要实现一种嵌入相关对象的方法。我们将从`Application\Entity\Customer`父类开始。在本节中，我们将假设以下关系，如下图所示。

* 一个顾客，多次购买
* 一次购买，一个产品

![](../../.gitbook/assets/image%20%28140%29.png)

6. 相应地，我们定义了一个`getter`和`setter`，以对象数组的形式处理购买。

```php
protected $purchases = array();
public function addPurchase($purchase)
{
  $this->purchases[] = $purchase;
}
public function getPurchases()
{
  return $this->purchases;
}
```

7.现在我们把注意力转移到`Application\Entity\Purchase`上。在这种情况下，购买和产品之间是1:1的关系，所以不需要处理这个数组。

```php
protected $product = NULL;
public function getProduct()
{
  return $this->product;
}
public function setProduct(Product $product)
{
  $this->product = $product;
}
```

{% hint style="info" %}
注意，在这两个实体类中，我们并没有改变`$mapping`数组。这是因为实现对象关系映射对实体属性名和数据库列名之间的映射没有影响。
{% endhint %}

8. 由于仍然需要获取客户基本信息的核心功能，所以我们需要做的就是扩展第5章“与数据库的交互“中实体类与RDBMS查询配方中描述的`Application\Database\CustomerService`类。我们可以新建一个`Application\Database\CustomerOrmService_1`类，该类扩展`Application\Database\CustomerService`。

```php
namespace Application\Database;
use PDO;
use PDOException;
use Application\Entity\Customer;
use Application\Entity\Product;
use Application\Entity\Purchase;
class CustomerOrmService_1 extends CustomerService
{
  // add methods here
}
```

9. 然后，我们在新的服务类中添加一个方法，执行查找并将结果以产品和采购实体的形式嵌入到核心客户实体中。这个方法以`JOIN`的形式执行查找。之所以能够做到这一点，是因为购买和产品之间存在1:1的关系。由于id列在两个表中的名称相同，我们需要添加购买ID列作为别名。然后我们在结果中循环，创建`Product`和`Purchase`实体。覆盖ID后，我们就可以将`Product`实体嵌入到`Purchase`实体中，然后将`Purchase`实体添加到`Customer`实体的数组中。

```php
protected function fetchPurchasesForCustomer(Customer $cust)
{
  $sql = 'SELECT u.*,r.*,u.id AS purch_id '
    . 'FROM purchases AS u '
    . 'JOIN products AS r '
    . 'ON r.id = u.product_id '
    . 'WHERE u.customer_id = :id '
    . 'ORDER BY u.date';
  $stmt = $this->connection->pdo->prepare($sql);
  $stmt->execute(['id' => $cust->getId()]);
  while ($result = $stmt->fetch(PDO::FETCH_ASSOC)) {
    $product = Product::arrayToEntity($result, new Product());
    $product->setId($result['product_id']);
    $purch = Purchase::arrayToEntity($result, new Purchase());
    $purch->setId($result['purch_id']);
    $purch->setProduct($product);
    $cust->addPurchase($purch);
  }
  return $cust;
}
```

10. 接下来，我们为原来的`fetchById()`方法提供一个封装器。这块代码不仅需要获取原始的`Customer`实体，还需要查找并嵌入`Product`和`Purchases`实体。我们可以调用新的`fetchByIdAndEmbedPurchases()`方法，并接受一个客户ID作为参数。

```php
public function fetchByIdAndEmbedPurchases($id)
{
  return $this->fetchPurchasesForCustomer(
    $this->fetchById($id));
}
```

### 技术方法\#2——嵌入二级查找功能

现在我们将介绍将二次查找嵌入到相关实体类中。我们将继续使用与上面相同的图示，使用定义的实体类对应三个相关的数据库表，即`customer`、`purchases`和`product`。

1.这种方法的机制与上一节中描述的很相似。主要的区别是，我们不做数据库查询，也不立即生成实体类，而是嵌入一系列匿名函数，这些函数将做同样的事情，但从视图逻辑中调用。

2.  我们需要在`Application\Entity\Customer`类中添加一个新的方法，为购买属性添加一个条目。我们将提供一个匿名函数，而不是一个采购实体数组。

```php
public function setPurchases(Closure $purchaseLookup)
{
  $this->purchases = $purchaseLookup;
}
```

3. 接下来，我们将`Application\Database\CustomerOrmService_1`类复制一份，并调用`Application\Database\CustomerOrmService_2`。

```php
namespace Application\Database;
use PDO;
use PDOException;
use Application\Entity\Customer;
use Application\Entity\Product;
use Application\Entity\Purchase;
class CustomerOrmService_2 extends CustomerService
{
  // code
}
```

4. 然后，我们定义一个`fetchPurchaseById()`方法，该方法根据其ID查找单个购买并生成一个`Purchase`实体。 因为我们最终将以这种方式对单次购买提出一系列重复请求，所以我们可以通过处理相同的准备语句（在本例中为`$purchPreparedStmt`属性）来重新获得数据库效率。

```php
public function fetchPurchaseById($purchId)
{
  if (!$this->purchPreparedStmt) {
      $sql = 'SELECT * FROM purchases WHERE id = :id';
      $this->purchPreparedStmt = 
      $this->connection->pdo->prepare($sql);
  }
  $this->purchPreparedStmt->execute(['id' => $purchId]);
  $result = $this->purchPreparedStmt->fetch(PDO::FETCH_ASSOC);
  return Purchase::arrayToEntity($result, new Purchase());
}
```

5. 之后，我们需要一个`fetchProductById()`方法，该方法根据其ID查找单个产品并产生一个`Product`实体。 假设客户可能多次购买相同的产品，我们可以通过将获得的产品实体存储在`$products`数组中来提高效率。 另外，与购买一样，我们可以在相同的准备好的语句上执行查找。

```php
public function fetchProductById($prodId)
{
  if (!isset($this->products[$prodId])) {
      if (!$this->prodPreparedStmt) {
          $sql = 'SELECT * FROM products WHERE id = :id';
          $this->prodPreparedStmt = 
          $this->connection->pdo->prepare($sql);
      }
      $this->prodPreparedStmt->execute(['id' => $prodId]);
      $result = $this->prodPreparedStmt
      ->fetch(PDO::FETCH_ASSOC);
      $this->products[$prodId] = 
        Product::arrayToEntity($result, new Product());
  }
  return $this->products[$prodId];
}
```

6. 现在我们可以重构`fetchPurchasesForCustomer()`方法，让它嵌入一个匿名函数，同时调用`fetchPurchaseById()`和`fetchProductById()`，然后将得到的产品实体分配给新找到的购买实体。在这个例子中，我们做一个初始查找，只是返回这个客户的所有购买的ID。然后，我们在`Customer::$purchases`属性中嵌入一系列匿名函数，将购买ID作为数组键，匿名函数作为其值。

```php
public function fetchPurchasesForCustomer(Customer $cust)
{
  $sql = 'SELECT id '
    . 'FROM purchases AS u '
    . 'WHERE u.customer_id = :id '
    . 'ORDER BY u.date';
  $stmt = $this->connection->pdo->prepare($sql);
  $stmt->execute(['id' => $cust->getId()]);
  while ($result = $stmt->fetch(PDO::FETCH_ASSOC)) {
    $cust->addPurchaseLookup(
    $result['id'],
    function ($purchId, $service) { 
      $purchase = $service->fetchPurchaseById($purchId);
      $product  = $service->fetchProductById(
                  $purchase->getProductId());
      $purchase->setProduct($product);
      return $purchase; }
    );
  }
  return $cust;
}
```

## 如何运行...

根据本示例的步骤定义以下类，如下。

| Class | 技巧 \#1 步骤 |
| :--- | :--- |
| `Application\Entity\Purchase` | 1 - 2, 7 |
| `Application\Entity\Product` | 3 - 4 |
| `Application\Entity\Customer` | 6, 16, + 在第5章，与数据库的交互中 |
| `Application\Database\CustomerOrmService_1` | 8 - 10 |

第二种方法如下：

| Class | 技巧 \#2 步骤 |
| :--- | :--- |
| `Application\Entity\Customer` | 2 |
| `Application\Database\CustomerOrmService_2` | 3 - 6 |

为了实现方法\#1，在实体被嵌入的情况下，定义一个名为`chap_11_orm_embedded.php`的调用程序，它设置自动加载并使用适当的类。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
use Application\Database\CustomerOrmService_1;
```

接下来，创建一个服务的实例，用随机的ID查找一个客户。

```php
$service = new CustomerOrmService_1(new Connection(include __DIR__ . DB_CONFIG_FILE));
$id   = rand(1,79);
$cust = $service->fetchByIdAndEmbedPurchases($id);
```

在视图逻辑中，你将通过`fetchByIdAndEmbedPurchases()`方法获得一个完全填充的`Customer`实体。现在你需要做的就是调用正确的getter来显示信息。

```php
 <!-- Customer Info -->
  <h1><?= $cust->getname() ?></h1>
  <div class="row">
    <div class="left">Balance</div><div class="right">
      <?= $cust->getBalance(); ?></div>
  </div>
    <!-- etc. -->
```

显示购买信息所需的逻辑就会像下面的HTML一样。注意`Customer::getPurchases()`会返回一个`Purchase`实体的数组。要从Purchase实体中获取产品信息，在循环中调用`Purchase::getProduct()`，它将生成一个`Product`实体。然后你可以调用任何一个 `Product` 获取器，在这个例子中，`Product::getTitle()`。

```php
  <!-- Purchases Info -->
  <table>
  <?php foreach ($cust->getPurchases() as $purchase) : ?>
  <tr>
  <td><?= $purchase->getTransaction() ?></td>
  <td><?= $purchase->getDate() ?></td>
  <td><?= $purchase->getQuantity() ?></td>
  <td><?= $purchase->getSalePrice() ?></td>
  <td><?= $purchase->getProduct()->getTitle() ?></td>
  </tr>
  <?php endforeach; ?>
</table>
```

将目光转向第二种方法，即使用二次查找，定义一个名为`chap_11_orm_secondary_lookups.php`的调用程序，该程序设置自动加载并使用相应的类。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Connection;
use Application\Database\CustomerOrmService_2;
```

接下来，创建一个服务的实例，用随机的ID查找一个客户。

```php
$service = new CustomerOrmService_2(new Connection(include __DIR__ . DB_CONFIG_FILE));
$id   = rand(1,79);
```

现在你可以检索一个`Application\Entity\Customer`实例，并为这个客户调用`fetchPurchasesForCustomer()`，它嵌入了匿名函数序列。

```php
$cust = $service->fetchById($id);
$cust = $service->fetchPurchasesForCustomer($cust);
```

显示核心客户信息的视图逻辑与之前描述的相同。显示购买信息所需的逻辑就像下面的HTML代码片段一样。请注意，`Customer::getPurchases()`返回一个匿名函数数组。每个函数调用都会返回一个特定的购买和相关产品。

```php
<table>
  <?php foreach($cust->getPurchases() as $purchId => $function) : ?>
  <tr>
  <?php $purchase = $function($purchId, $service); ?>
  <td><?= $purchase->getTransaction() ?></td>
  <td><?= $purchase->getDate() ?></td>
  <td><?= $purchase->getQuantity() ?></td>
  <td><?= $purchase->getSalePrice() ?></td>
  <td><?= $purchase->getProduct()->getTitle() ?></td>
  </tr>
  <?php endforeach; ?>
</table>
```

下面是一个输出的例子。

![](../../.gitbook/assets/image%20%28141%29.png)

{% hint style="info" %}
**最佳实践**

虽然循环的每一次迭代都代表了两个独立的数据库查询（一个查询购买，一个查询产品），但由于使用了准备好的语句，效率得以保留。事先准备了两个语句：一个是查询特定的购买，另一个是查询特定的产品。这些准备好的语句就会被多次执行。同时，每一个产品的检索都独立地存储在一个数组中，从而达到更高的效率。
{% endhint %}

## 更多...

实现对象关系映射的库的最好例子可能是Doctrine。Doctrine使用了一种嵌入式的方法，它的文档将其称为代理。更多信息，请参考[http://www.doctrine-project.org/projects/orm.html](http://www.doctrine-project.org/projects/orm.html)。





