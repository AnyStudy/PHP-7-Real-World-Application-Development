# 构建销售模块

销售模块是我们将建立的一系列模块中的最后一个模块，以提供一个简单而实用的网店应用程序。我们将通过在目录上添加购物车和结账功能来实现。结账本身将最终使用前面章节中定义的发货和支付服务。这里的总体重点将是绝对的基础知识，因为真正的购物车应用程序将采用更强大的方法。然而，了解如何以简单的方式将所有的东西联系在一起，是为以后更强大的网店应用实现打开大门的第一步。

在本章中，我们将涉及销售模块的以下主题：

* 要求
* 依赖
* 实施
* 单元测试
* 功能测试

## 要求

在前面的模块化网店App需求规范中给我们提供了一些与购物车和结账相关的线框。根据这些线框，我们可以推测我们需要创建什么样的实体才能实现功能。

以下是所需模块实体的清单：

* Cart
* Cart Item
* Order
* Order Item

购物车实体包括以下属性和它们的数据类型：

* `id`: integer, auto-increment
* `customer_id`: string
* `created_at`: datetime
* `modified_at`: datetime

购物车物品实体包括以下属性：

* `id`: integer, auto-increment
* `cart_id`: integer, 引用类别表id列的外键
* `product_id`: integer, 引用产品表id列的外键
* `qty`: string
* `unit_price`: decimal
* `created_at`: datetime
* `modified_at`: datetime

订单实体包括以下属性：

* `id`: integer, auto-increment
* `customer_id`: integer,引用客户表id列的外键
* `items_price`: decimal
* `shipment_price`: decimal
* `total_price`: decimal
* `status`: string
* `customer_email`: string
* `customer_first_name`: string
* `customer_last_name`: string
* `address_first_name`: string
* `address_last_name`: string
* `address_country`: string
* `address_state`: string
* `address_city`: string
* `address_postcode`: string
* `address_street`: string
* `address_telephone`: string
* `payment_method`: string
* `shipment_method`: string
* `created_at`: datetime
* `modified_at`: datetime

订单物品实体包括以下属性：

* `id`: integer, auto-increment
* `sales_order_id`: integer, 引用订单表id列的外键
* `product_id`: integer, 引用产品表id列的外键
* `title`: string
* `qty`: int
* `unit_price`: decimal
* `total_price`: decimal
* `created_at`: datetime
* `modified_at`: datetime

除了只是添加这些实体和它们的CRUD页面，我们还需要覆盖一个核心模块服务，负责构建分类菜单和发售商品。

## 依赖

销售模块在整个代码中会有几个依赖关系。这些依赖关系是针对客户和目录模块的。

## 实施

我们先创建一个新的模块`Foggyline\SalesBundle`。我们在控制台的帮助下，运行以下命令:

```bash
php bin/console generate:bundle --namespace=Foggyline/SalesBundle
```

该命令会触发一个交互式的过程，沿途向我们提出几个问题，如图所示:

![](../.gitbook/assets/image%20%28236%29.png)

完成后，`app/AppKernel.php`和`app/config/routing.yml`文件会自动修改。`AppKernel`类的`registerBundles`方法被添加到`$bundles`数组下的下面一行:

```php
new Foggyline\PaymentBundle\FoggylineSalesBundle(),
```

`routing.yml`文件已经更新，加入了以下条目:

```yaml
foggyline_payment:
  resource: "@FoggylineSalesBundle/Resources/config/routing.xml"
  prefix:   /
```

为了避免与核心应用代码发生冲突，我们需要将 `prefix: /` 改成 `prefix: /sales/`。

### 创建购物车实体

让我们继续创建购物车实体。我们通过使用控制台来创建，如图所示:

```bash
php bin/console generate:doctrine:entity
```

这将触发交互式生成器，如下图所示:

![](../.gitbook/assets/image%20%28237%29.png)

这将在 `src/Foggyline/SalesBundle/` 目录下创建 `Entity/Cart.php` 和 `Repository/CartRepository.php` 文件。在这之后，我们需要更新数据库，通过运行下面的命令，使其拉入购物车实体:

```bash
php bin/console doctrine:schema:update --force
```

有了`Cart`实体，我们可以继续生成`CartItem`实体。

### 创建购物车物品实体

让我们继续创建一个`CartItem`实体。我们使用现在著名的控制台命令来创建:

```bash
php bin/console generate:doctrine:entity
```

这将触发交互式生成器，如下截图所示:

![](../.gitbook/assets/image%20%28242%29.png)

这将在 `src/Foggyline/SalesBundle/` 目录下创建 `Entity/CartItem.php` 和 `Repository/CartItemRepository.php`。当自动生成完成后，我们需要返回并编辑`CartItem`实体，更新购物车字段关系如下:

```php
/**
 * @ORM\ManyToOne(targetEntity="Cart", inversedBy="items")
 * @ORM\JoinColumn(name="cart_id", referencedColumnName="id")
 */
private $cart;
```

在这里，我们定义了所谓的双向的一对多关联。一对多关联中的外键被定义在多端，在本例中是`CartItem`实体。双向映射需要在`OneToMany`关联上使用`mappedBy`属性，在`ManyToOne`关联上使用`inverseBy`属性。本例中的`OneToMany`是`Cart`实体，所以我们回到`src/Foggyline/SalesBundle/Entity/Cart.php`文件，并添加以下内容:

```php
/**
 * @ORM\OneToMany(targetEntity="CartItem", mappedBy="cart")
 */
private $items;

public function __construct() {
  $this->items = new \Doctrine\Common\Collections\ArrayCollection();
}
```

然后，我们需要更新数据库，通过运行以下命令，使其拉入`CartItem`实体:

```bash
php bin/console doctrine:schema:update --force
```

有了`CartItem`实体，我们可以继续生成订单实体。

### 创建体订单实

让我们继续创建一个订单实体。我们通过使用控制台来创建，如图所示：

```bash
php bin/console generate:doctrine:entity
```

如果我们试图提供`FoggylineSalesBundle:Order`作为实体的快捷方式名称，生成的输出将抛出一个错误，如下截图所示:

![](../.gitbook/assets/image%20%28244%29.png)

相反，我们将使用`SensioGeneratorBundle:SalesOrder`作为实体的快捷方式名称，并按照这里所示的生成器完成:

![](../.gitbook/assets/image%20%28240%29.png)

接下来是其余的客户信息相关字段。为了更好地了解情况，请看下面的截图:

![](../.gitbook/assets/image%20%28241%29.png)

之后是与订单地址相关的其他字段，如图所示:

![](../.gitbook/assets/image%20%28243%29.png)

值得注意的是，通常我们希望在它自己的表中提取地址信息，也就是让它成为自己的实体。然而，为了保持简单，我们将把它作为`SalesOrder`实体的一部分。

一旦完成，这将在 `src/Foggyline/SalesBundle/` 目录下创建 `Entity/SalesOrder.php` 和 `Repository/SalesOrderRepository.php` 文件。在这之后，我们需要更新数据库，通过运行下面的命令，使其拉入`SalesOrder`实体:

```bash
php bin/console doctrine:schema:update --force
```

有了`SalesOrder`实体，我们可以继续生成`SalesOrderItem`实体。

### 创建SalesOrderItem实体

让我们继续创建`SalesOrderItem`实体。我们通过使用下面的控制台命令来启动代码生成器:

```bash
php bin/console generate:doctrine:entity
```

当被要求提供实体快捷方式名称时，我们提供`FoggylineSalesBundle:SalesOrderItem`，然后按照生成器字段定义，如下截图所示:

![](../.gitbook/assets/image%20%28239%29.png)

这将在 `src/Foggyline/SalesBundle/`目录下创建 `Entity/SalesOrderItem.php` 和 `Repository/SalesOrderItemRepository.php` 文件。当自动生成完成后，我们需要返回编辑`SalesOrderItem`实体来更新`SalesOrder`字段关系，如下所示:

```php
/**
 * @ORM\ManyToOne(targetEntity="SalesOrder", inversedBy="items")
 * @ORM\JoinColumn(name="sales_order_id", referencedColumnName="id")
 */
private $salesOrder;

/**
 * @ORM\OneToOne(targetEntity="Foggyline\CatalogBundle\Entity\Product")
 * @ORM\JoinColumn(name="product_id", referencedColumnName="id")
 */
private $product;
```

在这里，我们定义了两种类型的关系。第一种，与`$salesOrder`相关，是一对多的双向关联，我们在`Cart`和`CartItem`实体中看到了这种关联。第二种，与`$product`有关，是单向的一对一的关联。之所以说是单向引用，是因为`CartItem`引用了`Product`，而`Product`不会引用`CartItem`，因为我们不想改变属于其他模块的东西。

我们仍然需要回到 `src/Foggyline/SalesBundle/Entity/SalesOrder.php` 文件，并添加以下内容:

```php
/**
 * @ORM\OneToMany(targetEntity="SalesOrderItem", mappedBy="salesOrder")
 */
private $items;

public function __construct() {
  $this->items = new \Doctrine\Common\Collections\ArrayCollection();
}
```

然后，我们需要更新数据库，使其拉入`SalesOrderItem`实体，运行以下命令:

```bash
php bin/console doctrine:schema:update --force
```

有了`SalesOrderItem`实体，我们就可以开始构建购物车和结账页面了。

### 覆盖add\_to\_cart\_url服务

`add_to_cart_url`服务最初是在`FoggylineCustomerBundle`中用j假数据声明的。这是因为在销售功能可用之前，我们需要一种在产品上建立Add to Cart URL的方法。虽然肯定不是很理想，但这是一种可能的方式。

现在，我们将用销售模块中声明的服务覆盖该服务，以提供正确的添加到购物车的URL。我们首先在 `src/Foggyline/SalesBundle/Resources/config/services.xml` 中定义服务，在服务下添加以下服务元素，如下所示:

```markup
<service id="foggyline_sales.add_to_cart_url" class="Foggyline\SalesBundle\Service\AddToCartUrl">
  <argument type="service" id="doctrine.orm.entity_manager"/>
  <argument type="service" id="router"/>
</service>
```

然后我们创建`src/Foggyline/SalesBundle/Service/AddToCartUrl.php`，内容如下:

```php
namespace Foggyline\SalesBundle\Service;

class AddToCartUrl
{
  private $em;
  private $router;

  public function __construct(
    \Doctrine\ORM\EntityManager $entityManager,
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->em = $entityManager;
    $this->router = $router;
  }

  public function getAddToCartUrl($productId)
  {
    return $this->router->generate('foggyline_sales_cart_add', array('id' => $productId));
  }
}
```

路由器服务在这里期待名为`foggyline_sales_cart_add`的路由，但它仍然不存在。我们通过在 `src/Foggyline/SalesBundle/Resources/config/routing.xml` 文件的 `routes` 元素下添加以下条目来创建路由，如下所示:

```markup
<route id="foggyline_sales_cart_add" path="/cart/add/{id}">
  <default key="_controller">FoggylineSalesBundle:Cart:add</default>
</route>
```

路由定义期望在`src/Foggyline/SalesBundle/Controller/CarController.php`文件中找到购物车控制器中的`addAction`函数，我们定义如下:

```php
namespace Foggyline\SalesBundle\Controller;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;


class CartController extends Controller
{
  public function addAction($id)
  {
    if ($customer = $this->getUser()) {
      $em = $this->getDoctrine()->getManager();
      $now = new \DateTime();

      $product = $em->getRepository('FoggylineCatalogBundle:Product')->find($id);

      // Grab the cart for current user
      $cart = $em->getRepository('FoggylineSalesBundle:Cart')->findOneBy(array('customer' => $customer));

      // If there is no cart, create one
      if (!$cart) {
        $cart = new \Foggyline\SalesBundle\Entity\Cart();
        $cart->setCustomer($customer);
        $cart->setCreatedAt($now);
        $cart->setModifiedAt($now);
      } else {
        $cart->setModifiedAt($now);
      }

      $em->persist($cart);
      $em->flush();

      // Grab the possibly existing cart item
      // But, lets find it directly
      $cartItem = $em->getRepository('FoggylineSalesBundle:CartItem')->findOneBy(array('cart' => $cart, 'product' => $product));

      if ($cartItem) {
        // Cart item exists, update it
        $cartItem->setQty($cartItem->getQty() + 1);
        $cartItem->setModifiedAt($now);
      } else {
        // Cart item does not exist, add new one
        $cartItem = new \Foggyline\SalesBundle\Entity\CartItem();
        $cartItem->setCart($cart);
        $cartItem->setProduct($product);
        $cartItem->setQty(1);
        $cartItem->setUnitPrice($product->getPrice());
        $cartItem->setCreatedAt($now);
        $cartItem->setModifiedAt($now);
      }

      $em->persist($cartItem);
      $em->flush();

      $this->addFlash('success', sprintf('%s successfully added to cart', $product->getTitle()));

      return $this->redirectToRoute('foggyline_sales_cart');
    } else {
      $this->addFlash('warning', 'Only logged in users can add to cart.');
      return $this->redirect('/');
    }
  }
}
```

在`addAction`方法中，有相当多的逻辑在进行。我们首先检查当前用户在数据库中是否已经有一个购物车条目；如果没有，我们就创建一个新的条目。然后我们添加或更新现有的购物车项目。

为了让我们新的`add_to_cart`服务实际覆盖`Customermodule`的服务，我们仍然需要添加一个编译器。我们通过定义`src/Foggyline/SalesBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php`文件来实现，内容如下:

```php
namespace Foggyline\SalesBundle\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Definition;

class OverrideServiceCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container)
    {
        // Override 'add_to_cart_url' service
        $container->removeDefinition('add_to_cart_url');
        $container->setDefinition('add_to_cart_url', $container->getDefinition('foggyline_sales.add_to_cart_url'));

        // Override 'checkout_menu' service
        // Override 'foggyline_customer.customer_orders' service
        // Override 'bestsellers' service
        // Pickup/parse 'shipment_method' services
        // Pickup/parse 'payment_method' services
    }
}
```

稍后，我们将在这个文件中添加其余的覆盖。为了使`add_to_cart`服务覆盖生效，我们需要在`src/Foggyline/SalesBundle/FoggylineSalesBundle.php`文件的`build`方法中注册编译器传递，如下所示:

```php
public function build(ContainerBuilder $container)
{
    parent::build($container);;
    $container->addCompilerPass(new OverrideServiceCompilerPass());
}
```

覆盖现在应该是有效的，我们的销售模块现在应该提供有效的添加到购物车链接。

### 覆盖checkout\_menu服务

客户模块中定义的结账菜单服务的目的很简单，就是提供一个链接到购物车和结账过程的第一步。由于当时不知道销售模块，`Customer`模块提供了一个假链接，我们现在将覆盖这个链接。

我们先在`src/Foggyline/SalesBundle/Resources/config/services.xml`文件的`services`元素下添加以下服务条目，具体如下：

```markup
<service id="foggyline_sales.checkout_menu" class="Foggyline\SalesBundle\Service\CheckoutMenu">
<argument type="service" id="doctrine.orm.entity_manager"/>
<argument type="service" id="security.token_storage"/>
<argument type="service" id="router"/>
</service>
```

然后我们添加`src/Foggyline/SalesBundle/Service/CheckoutMenu.php`文件，内容如下:

```php
namespace Foggyline\SalesBundle\Service;

class CheckoutMenu
{
  private $em;
  private $token;
  private $router;

  public function __construct(
    \Doctrine\ORM\EntityManager $entityManager,
    $tokenStorage,
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->em = $entityManager;
    $this->token = $tokenStorage->getToken();
    $this->router = $router;
  }

  public function getItems()
  {
    if ($this->token
      && $this->token->getUser() instanceof \Foggyline\CustomerBundle\Entity\Customer
    ) {
      $customer = $this->token->getUser();

      $cart = $this->em->getRepository('FoggylineSalesBundle:Cart')->findOneBy(array('customer' => $customer));

      if ($cart) {
        return array(
          array('path' => $this->router->generate('foggyline_sales_cart'), 'label' =>sprintf('Cart (%s)', count($cart->getItems()))),
          array('path' => $this->router->generate('foggyline_sales_checkout'), 'label' =>'Checkout'),
        );
      }
    }

    return array();
  }
}
```

服务需要两个路由，`foggyline_sales_cart`和`foggyline_sales_checkout`，所以我们需要修改`src/Foggyline/SalesBundle/Resources/config/routing.xml`，在其中添加以下路由定义:

```markup
<route id="foggyline_sales_cart" path="/cart/">
  <default key="_controller">FoggylineSalesBundle:Cart:index</default>
</route>

<route id="foggyline_sales_checkout" path="/checkout/">
  <default key="_controller">FoggylineSalesBundle:Checkout:index</default>
</route>
```

新添加的路由期待购物车和结账控制器。购物车控制器已经有了，所以我们只需要给它添加`indexAction`即可。此时，我们只需添加一个空的，如下所示:

```php
public function indexAction(Request $request)
{
}
```

同样，让我们创建一个`src/Foggyline/SalesBundle/Controller/CheckoutController.php`文件，内容如下:

```php
namespace Foggyline\SalesBundle\Controller;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\Extension\Core\Type\CountryType;

class CheckoutController extends Controller
{
  public function indexAction()
  {
  }
}
```

稍后，我们将重新回到这两个`indexAction`方法，并添加适当的方法体实现。

为了结束服务覆盖，我们现在修改之前创建的`src/Foggyline/SalesBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php`文件，将`// Override 'checkout_menu'` 服务注释替换为以下内容:

```php
$container->removeDefinition('checkout_menu');
$container->setDefinition('checkout_menu', $container->getDefinition('foggyline_sales.checkout_menu'));
```

我们新定义的服务现在应该覆盖在客户模块中定义的服务，从而提供正确的结账和购物车（购物车计数中的项目）URL。

### 覆盖客户订单服务

`foggyline_customer.customer_orders`服务是为当前登录的客户提供以前创建的订单集合。客户模块为此定义了一个虚拟服务，这样我们就可以继续在我的账户页面下建立My Orders部分。现在我们需要覆盖这个服务，使其返回正确的订单。

我们首先在 `src/Foggyline/SalesBundle/Resources/config/services.xml` 文件的服务下添加以下服务元素，如下所示:

```markup
<service id="foggyline_sales.customer_orders" class="Foggyline\SalesBundle\Service\CustomerOrders">
  <argument type="service" id="doctrine.orm.entity_manager"/>
  <argument type="service" id="security.token_storage"/>
  <argument type="service" id="router"/>
</service>
```

然后我们添加`src/Foggyline/SalesBundle/Service/CustomerOrders.php`文件，内容如下:

```php
namespace Foggyline\SalesBundle\Service;

class CustomerOrders
{
  private $em;
  private $token;
  private $router;

  public function __construct(
    \Doctrine\ORM\EntityManager $entityManager,
    $tokenStorage,
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->em = $entityManager;
    $this->token = $tokenStorage->getToken();
    $this->router = $router;
  }

  public function getOrders()
  {
    $orders = array();

    if ($this->token
    && $this->token->getUser() instanceof \Foggyline\CustomerBundle\Entity\Customer
    ) {
      $salesOrders = $this->em->getRepository('FoggylineSalesBundle:SalesOrder')
      ->findBy(array('customer' => $this->token->getUser()));

      foreach ($salesOrders as $salesOrder) {
        $orders[] = array(
          'id' => $salesOrder->getId(),
          'date' => $salesOrder->getCreatedAt()->format('d/m/Y H:i:s'),
          'ship_to' => $salesOrder->getAddressFirstName() . '' . $salesOrder->getAddressLastName(),
'         'order_total' => $salesOrder->getTotalPrice(),
          'status' => $salesOrder->getStatus(),
          'actions' => array(
            array(
              'label' =>'Cancel',
              'path' => $this->router->generate('foggyline_sales_order_cancel', array('id' => $salesOrder->getId()))
            ),
            array(
              'label' =>'Print',
              'path' => $this->router->generate('foggyline_sales_order_print', array('id' => $salesOrder->getId()))
            )
          )
        );
      }
    }
    return $orders;
  }
}
```

路由生成方法期望找到两个路由，`foggyline_sales_order_cancel`和`foggyline_sales_order_print`，这两个路由还没有被创建。

让我们继续创建它们，在 `src/Foggyline/SalesBundle/Resources/config/routing.xml` 文件的路由元素下添加以下内容:

```php
<route id="foggyline_sales_order_cancel"path="/order/cancel/{id}">
  <default key="_controller">FoggylineSalesBundle:SalesOrder:cancel</default>
</route>

<route id="foggyline_sales_order_print" path="/order/print/{id}">
  <default key="_controller">FoggylineSalesBundle:SalesOrder:print</default>
</route>
```

路由定义则期望定义`SalesOrderController`。由于我们的应用程序需要一个管理员用户才能列出和编辑订单，我们将使用下面的Symfony命令来自动生成销售订单的CRUD:

```bash
php bin/console generate:doctrine:crud
```

当被要求提供实体快捷方式名称时，我们只需提供`FoggylineSalesBundle:SalesOrder`，然后继续，允许创建写操作。此时，我们已经为我们创建了几个文件，以及`Sales bundle`之外的一些条目。其中一个条目是`app/config/routing.yml`文件中的路由定义，如下所示:

```yaml
foggyline_sales_sales_order:
  resource: "@FoggylineSalesBundle/Controller/SalesOrderController.php"
  type:     annotation
```

我们应该已经有一个`foggyline_sales`条目。不同的是 `foggyline_sales` 指向我们的 `router.xml` 文件，而新创建的 `foggyline_sales_sales_order` 指向新创建的 `SalesOrderController`。为了简单起见，我们可以同时保留它们。

自动生成器还在`app/Resources/views/`目录下创建了一个`salesorder`目录，我们需要把它移到我们的`bundle`中，作为`src/Foggyline/SalesBundle/Resources/views/Default/salesorder/`目录。

现在我们可以在 `src/Foggyline/SalesBundle/Controller/SalesOrderController.php` 文件中添加以下内容来处理打印和取消操作，具体如下:

```php
public function cancelAction($id)
{
  if ($customer = $this->getUser()) {
    $em = $this->getDoctrine()->getManager();
    $salesOrder = $em->getRepository('FoggylineSalesBundle:SalesOrder')
    ->findOneBy(array('customer' => $customer, 'id' => $id));

    if ($salesOrder->getStatus() != \Foggyline\SalesBundle\Entity\SalesOrder::STATUS_COMPLETE) {
      $salesOrder->setStatus(\Foggyline\SalesBundle\Entity\SalesOrder::STATUS_CANCELED);
      $em->persist($salesOrder);
      $em->flush();
    }
  }

  return $this->redirectToRoute('customer_account');
}

public function printAction($id)
{
  if ($customer = $this->getUser()) {
    $em = $this->getDoctrine()->getManager();
    $salesOrder = $em->getRepository('FoggylineSalesBundle:SalesOrder')
    ->findOneBy(array('customer' => $customer, 'id' =>$id));

    return $this->render('FoggylineSalesBundle:default:salesorder/print.html.twig', array(
      'salesOrder' => $salesOrder,
      'customer' => $customer
    ));
  }

  return $this->redirectToRoute('customer_account');
}
```

`cancelAction`方法只是检查该订单是否属于当前登录的客户，如果属于，则允许更改订单状态。`printAction`方法只是加载订单，如果它属于当前登录的客户，并将其传递给`print.html.twig`模板。

然后我们创建`src/Foggyline/SalesBundle/Resources/views/default/salesorder/print.html.twig`模板，内容如下:

```markup
{% block body %}
<h1>Printing Order #{{ salesOrder.id }}</h1>
  {#<p>Just a dummy Twig dump of entire variable</p>#}
  {{ dump(salesOrder) }}
{% endblock %}
```

显然，这只是一个简化的输出，我们可以根据自己的需要进一步定制。重要的是，我们已经把订单对象传给了我们的模板，现在可以从中提取任何需要的信息。

最后，我们用以下代码替换 `src/Foggyline/SalesBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php` 文件中的 `// Override 'foggyline_customer.customer_orders'` 服务注释:

```php
$container->removeDefinition('foggyline_customer.customer_orders');
$container->setDefinition('foggyline_customer.customer_orders', $container->getDefinition('foggyline_sales.customer_orders'));
```

这将使服务覆盖启动，并拉动我们刚才的所有变化。

### 覆盖最畅销服务

客户模块中定义的畅销品服务应该是为首页显示的畅销品功能提供虚拟数据。我们的想法是在商店中展示五种最畅销的产品。现在销售模块需要覆盖这个服务以提供正确的实现方式，实际销售的产品数量会影响到所展示的畅销产品的内容。

我们首先在 `src/Foggyline/SalesBundle/Resources/config/services.xml` 文件的 service 元素下添加以下定义：

```markup
<service id="foggyline_sales.bestsellers" class="Foggyline\SalesBundle\Service\BestSellers">
  <argument type="service" id="doctrine.orm.entity_manager"/>
  <argument type="service" id="router"/>
</service>
```

然后我们定义`src/Foggyline/SalesBundle/Service/BestSellers.php`文件，内容如下:

```php
namespace Foggyline\SalesBundle\Service;

class BestSellers
{
  private $em;
  private $router;

  public function __construct(
    \Doctrine\ORM\EntityManager $entityManager,
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->em = $entityManager;
    $this->router = $router;
  }

  public function getItems()
  {
    $products = array();
    $salesOrderItem = $this->em->getRepository('FoggylineSalesBundle:SalesOrderItem');
    $_products = $salesOrderItem->getBestsellers();

    foreach ($_products as $_product) {
      $products[] = array(
        'path' => $this->router->generate('product_show', array('id' => $_product->getId())),
        'name' => $_product->getTitle(),
        'img' => $_product->getImage(),
        'price' => $_product->getPrice(),
        'id' => $_product->getId(),
      );
    }
    return $products;
  }
}
```

在这里，我们要获取`SalesOrderItemRepository`类的实例，并对其调用`getBestsellers`方法。这个方法仍然没有被定义。我们将其添加到文件 `src/Foggyline/SalesBundle/Repository/SalesOrderItemRepository.php` 文件中，如下所示:

```php
public function getBestsellers()
{
  $products = array();

  $query = $this->_em->createQuery('SELECT IDENTITY(t.product), SUM(t.qty) AS HIDDEN q
  FROM Foggyline\SalesBundle\Entity\SalesOrderItem t
  GROUP BY t.product ORDER BY q DESC')
  ->setMaxResults(5);

  $_products = $query->getResult();

  foreach ($_products as $_product) {
    $products[] = $this->_em->getRepository('FoggylineCatalogBundle:Product')
    ->find(current($_product));
  }

  return $products;
}
```

在这里，我们使用Doctrine Query Language \(DQL\)来建立一个五个畅销产品的列表。最后，我们需要将`src/Foggyline/SalesBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php`文件中的`// Override 'bestsellers'`服务注释替换为如下代码:

```php
$container->removeDefinition('bestsellers');
$container->setDefinition('bestsellers', $container->getDefinition('foggyline_sales.bestsellers'));
```

通过覆盖畅销产品服务，我们将基于实际销量的畅销产品列表暴露出来，供其他模块取用。

### 创建购物车页面

购物车页面是客户通过 "添加到购物车 "按钮，从首页、分类页或产品页看到添加到购物车的产品列表的地方。我们之前创建了`CartController`和一个空的`indexAction`函数。现在我们继续编辑`indexAction`函数如下:

```php
public function indexAction()
{
  if ($customer = $this->getUser()) {
    $em = $this->getDoctrine()->getManager();

    $cart = $em->getRepository('FoggylineSalesBundle:Cart')->findOneBy(array('customer' => $customer));
    $items = $cart->getItems();
    $total = null;

    foreach ($items as $item) {
      $total += floatval($item->getQty() * $item->getUnitPrice());
    }

    return $this->render('FoggylineSalesBundle:default:cart/index.html.twig', array(
        'customer' => $customer,
        'items' => $items,
        'total' => $total,
      ));
  } else {
    $this->addFlash('warning', 'Only logged in customers can access cart page.');
    return $this->redirectToRoute('foggyline_customer_login');
  }
}
```

在这里，我们检查用户是否已登录；如果用户已登录，我们将向他们展示购物车中的所有商品。未登录的用户会被重定向到客户登录的URL。`indexAction`函数期望得到`src/Foggyline/SalesBundle/Resources/views/Default/art/index.html.twig`文件，其内容我们定义如下:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<h1>Shopping Cart</h1>
<div class="row">
  <div class="large-8 columns">
    <form action="{{ path('foggyline_sales_cart_update') }}"method="post">
    <table>
      <thead>
        <tr>
          <th>Item</th>
          <th>Price</th>
          <th>Qty</th>
          <th>Subtotal</th>
        </tr>
      </thead>
      <tbody>
        {% for item in items %}
        <tr>
          <td>{{ item.product.title }}</td>
          <td>{{ item.unitPrice }}</td>
          <td><input name="item[{{ item.id }}]" value="{{ item.qty }}"/></td>
          <td>{{ item.qty * item.unitPrice }}</td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
    <button type="submit" class="button">Update Cart</button>
  </form>
</div>
<div class="large-4 columns">
  <div>Order Total: {{ total }}</div>
  <div><a href="{{ path('foggyline_sales_checkout') }}"class="button">Go to Checkout</a></div>
  </div>
</div>
{% endblock %}
```

渲染后，模板将在每个添加的产品下显示数量输入元素，同时显示更新购物车按钮。更新购物车按钮提交表单，其操作指向`foggyline_sales_cart_update`路径。

让我们继续创建`foggyline_sales_cart_update`，在`src/Foggyline/SalesBundle/Resources/config/routing.xml`文件的路由元素下添加以下条目，如下所示:

```markup
<route id="foggyline_sales_cart_update" path="/cart/update">
  <default key="_controller">FoggylineSalesBundle:Cart:update</default>
</route>
```

新定义的路由期望在 `src/Foggyline/SalesBundle/Controller/CartController.php` 文件下找到一个 `updateAction` 函数，我们添加如下:

```php
public function updateAction(Request $request)
{
  $items = $request->get('item');

  $em = $this->getDoctrine()->getManager();
  foreach ($items as $_id => $_qty) {
    $cartItem = $em->getRepository('FoggylineSalesBundle:CartItem')->find($_id);
    if (intval($_qty) > 0) {
      $cartItem->setQty($_qty);
      $em->persist($cartItem);
    } else {
      $em->remove($cartItem);
    }
  }
  // Persist to database
  $em->flush();

  $this->addFlash('success', 'Cart updated.');

  return $this->redirectToRoute('foggyline_sales_cart');
}
```

要从购物车中删除产品，我们只需插入0作为数量值，然后点击更新购物车按钮。这就完成了我们简单的购物车页面。

### 创建支付服务

为了从购物车转移到结账，我们需要整理出支付和发货服务。之前的 `Payment` 和 `Shipment` 模块暴露了他们的一些 `Payment` 和 `Shipment` 服务，现在我们需要将这些服务聚合成一个单一的 `Payment` 和 `Shipment` 服务，以便我们的结账流程使用。

我们首先将之前添加的  `// Pickup/parse 'payment_method'` 服务注释替换到 `src/Foggyline/SalesBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php` 文件下，代码如下:

```php
$container->getDefinition('foggyline_sales.payment')
  ->addArgument(
  array_keys($container->findTaggedServiceIds('payment_method'))
);
```

`findTaggedServiceIds`方法返回一个`key-value`列表，其中包含了所有带有`payment_method`标签的服务，然后我们将其作为参数传递给`foggyline_sales.payment`服务。这是Symfony在编译过程中获取服务列表的唯一方法。

然后我们编辑`src/Foggyline/SalesBundle/Resources/config/services.xml`文件，在服务元素下添加以下内容:

```markup
<service id="foggyline_sales.payment" class="Foggyline\SalesBundle\Service\Payment">
  <argument type="service" id="service_container"/>
</service>
```

最后，我们在 `src/Foggyline/SalesBundle/Service/Payment.php` 文件中创建 `Payment` 类，如下所示:

```php
namespace Foggyline\SalesBundle\Service;

class Payment
{
  private $container;
  private $methods;

  public function __construct($container, $methods)
  {
    $this->container = $container;
    $this->methods = $methods;
  }

  public function getAvailableMethods()
  {
    $methods = array();

    foreach ($this->methods as $_method) {
      $methods[] = $this->container->get($_method);
    }

    return $methods;
  }
}
```

按照`services.xml`文件中的服务定义，我们的服务接受两个参数，一个是`$container`，另一个是`$methods`。`$methods`参数是在编译的时候传递的，在这里我们能够获取所有`payment_method`标签服务的列表。这意味着我们的`getAvailableMethods`现在能够从任何模块中返回所有`payment_method`标记的服务。

### 创建发货服务

发货服务的实现方式与支付服务大同小异。整体的想法是相似的，只是在方式上有一些不同。我们首先将之前在 `src/Foggyline/SalesBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php` 文件下添加的   `// Pickup/parse shipment_method'` 服务注释替换为如下代码:

```php
$container->getDefinition('foggyline_sales.shipment')
  ->addArgument(
  array_keys($container->findTaggedServiceIds('shipment_method'))
);
```

然后我们编辑 `src/Foggyline/SalesBundle/Resources/config/services.xml` 文件，在服务元素下添加以下内容:

```markup
<service id="foggyline_sales.shipment"class="Foggyline\SalesBundle\Service\Payment">
  <argument type="service" id="service_container"/>
</service>
```

最后，我们在 `src/Foggyline/SalesBundle/Service/Shipment.php` 文件中创建 `Shipment` 类，如下所示:

```php
namespace Foggyline\SalesBundle\Service;

class Shipment
{
  private $container;
  private $methods;

  public function __construct($container, $methods)
  {
    $this->container = $container;
    $this->methods = $methods;
  }

  public function getAvailableMethods()
  {
    $methods = array();
    foreach ($this->methods as $_method) {
      $methods[] = $this->container->get($_method);
    }

    return $methods;
  }
}
```

我们现在能够通过统一的支付和发货服务获取所有的支付和发货服务，从而使结账过程变得简单。

### 创建结账页面

结账页面将由两个结账步骤构成，第一个步骤是发货信息收集，第二个步骤是支付信息收集。

我们先从发货步骤开始，将我们的`src/Foggyline/SalesBundle/Controller/CheckoutController.php`文件及其`indexAction`修改如下:

```php
public function indexAction()
{
  if ($customer = $this->getUser()) {

    $form = $this->getAddressForm();

    $em = $this->getDoctrine()->getManager();
    $cart = $em->getRepository('FoggylineSalesBundle:Cart')->findOneBy(array('customer' => $customer));
    $items = $cart->getItems();
    $total = null;

    foreach ($items as $item) {
      $total += floatval($item->getQty() * $item->getUnitPrice());
    }

    return $this->render('FoggylineSalesBundle:default:checkout/index.html.twig', array(
      'customer' => $customer,
      'items' => $items,
      'cart_subtotal' => $total,
      'shipping_address_form' => $form->createView(),
      'shipping_methods' => $this->get('foggyline_sales.shipment')->getAvailableMethods()
    ));
  } else {
    $this->addFlash('warning', 'Only logged in customers can access checkout page.');
    return $this->redirectToRoute('foggyline_customer_login');
  }
}
private function getAddressForm()
{
  return $this->createFormBuilder()
  ->add('address_first_name', TextType::class)
  ->add('address_last_name', TextType::class)
  ->add('company', TextType::class)
  ->add('address_telephone', TextType::class)
  ->add('address_country', CountryType::class)
  ->add('address_state', TextType::class)
  ->add('address_city', TextType::class)
  ->add('address_postcode', TextType::class)
  ->add('address_street', TextType::class)
  ->getForm();
}
```

在这里，我们要获取当前登录的客户购物车，并将其传递到一个`checkout/index.html.twig`模板上，同时还有发货步骤所需的其他几个变量。`getAddressForm`方法只是为我们构建了一个地址表单。还有一个向我们新创建的`foggyline_sales.shipment`服务的调用，它使我们能够获取所有可用的发货方式的列表。

然后我们创建`src/Foggyline/SalesBundle/Resources/views/default/checkout/index.html.twig`，内容如下:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<h1>Checkout</h1>

<div class="row">
  <div class="large-8 columns">
    <form action="{{ path('foggyline_sales_checkout_payment') }}" method="post" id="shipping_form">
      <fieldset>
        <legend>Shipping Address</legend>
        {{ form_widget(shipping_address_form) }}
      </fieldset>

      <fieldset>
        <legend>Shipping Methods</legend>
        <ul>
          {% for method in shipping_methods %}
          {% set shipment = method.getInfo('street', 'city', 'country', 'postcode', 'amount', 'qty')['shipment'] %}
          <li>
            <label>{{ shipment.title }}</label>
            <ul>
              {% for delivery_option in shipment.delivery_options %}
              <li>
                <input type="radio" name="shipment_method"
                  value="{{ shipment.code }}____{{ delivery_option.code }}____{{ delivery_option.price }}"> {{ delivery_option.title }}
                  ({{ delivery_option.price }})
                <br>
              </li>
              {% endfor %}
            </ul>
          </li>
          {% endfor %}
        </ul>
      </fieldset>
    </form>
  </div>
  <div class="large-4 columns">
    {% include 'FoggylineSalesBundle:default:checkout/order_sumarry.html.twig' 
    %}
    <div>Cart Subtotal: {{ cart_subtotal }}</div>
    <div><a id="shipping_form_submit" href="#" class="button">Next</a>
    </div>
  </div>
</div>

<script type="text/javascript">
  var form = document.getElementById('shipping_form');
  document.getElementById('shipping_form_submit').addEventListener('click', function () {
    form.submit();
  });
</script>
{% endblock %}
```

该模板列出了所有与地址相关的表单字段，以及可用的发货方式。JavaScript部分负责处理Next按钮的点击，这基本上是将表单提交到`foggyline_sales_checkout_payment`路由。

然后，我们通过在 `src/Foggyline/SalesBundle/Resources/config/routing.xml` 文件的 `routes` 元素下添加以下条目来定义 `foggyline_sales_checkout_payment` 路由:

```markup
<route id="foggyline_sales_checkout_payment" path="/checkout/payment">
  <default key="_controller">FoggylineSalesBundle:Checkout:payment</default>
</route>
```

路由条目期望在`CheckoutController`中找到一个`paymentAction`，我们定义如下:

```php
public function paymentAction(Request $request)
{
  $addressForm = $this->getAddressForm();
  $addressForm->handleRequest($request);

  if ($addressForm->isSubmitted() && $addressForm->isValid() && $customer = $this->getUser()) {

    $em = $this->getDoctrine()->getManager();
    $cart = $em->getRepository('FoggylineSalesBundle:Cart')->findOneBy(array('customer' => $customer));
    $items = $cart->getItems();
    $cartSubtotal = null;

    foreach ($items as $item) {
      $cartSubtotal += floatval($item->getQty() * $item->getUnitPrice());
    }

    $shipmentMethod = $_POST['shipment_method'];
    $shipmentMethod = explode('____', $shipmentMethod);
    $shipmentMethodCode = $shipmentMethod[0];
    $shipmentMethodDeliveryCode = $shipmentMethod[1];
    $shipmentMethodDeliveryPrice = $shipmentMethod[2];

    // Store relevant info into session
    $checkoutInfo = $addressForm->getData();
    $checkoutInfo['shipment_method'] = $shipmentMethodCode . '____' . $shipmentMethodDeliveryCode;
    $checkoutInfo['shipment_price'] = $shipmentMethodDeliveryPrice;
    $checkoutInfo['items_price'] = $cartSubtotal;
    $checkoutInfo['total_price'] = $cartSubtotal + $shipmentMethodDeliveryPrice;
    $this->get('session')->set('checkoutInfo', $checkoutInfo);

    return $this->render('FoggylineSalesBundle:default:checkout/payment.html.twig', array(
      'customer' => $customer,
      'items' => $items,
      'cart_subtotal' => $cartSubtotal,
      'delivery_subtotal' => $shipmentMethodDeliveryPrice,
      'delivery_label' =>'Delivery Label Here',
      'order_total' => $cartSubtotal + $shipmentMethodDeliveryPrice,
      'payment_methods' => $this->get('foggyline_sales.payment')->getAvailableMethods()
    ));
  } else {
    $this->addFlash('warning', 'Only logged in customers can access checkout page.');
    return $this->redirectToRoute('foggyline_customer_login');
  }
}
```

前面的代码从结账过程的发货步骤中获取提交的内容，将相关的值存储到会话中，获取支付步骤所需的变量，并渲染回`checkout/payment.html.twig`模板。

我们定义`src/Foggyline/SalesBundle/Resources/views/default/checkout/payment.html.twig`文件，内容如下:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<h1>Checkout</h1>
<div class="row">
  <div class="large-8 columns">
    <form action="{{ path('foggyline_sales_checkout_process') }}"method="post" id="payment_form">
      <fieldset>
        <legend>Payment Methods</legend>
        <ul>
          {% for method in payment_methods %}
          {% set payment = method.getInfo()['payment'] %}
          <li>
            <input type="radio" name="payment_method"
              value="{{ payment.code }}"> {{ payment.title }}
            {% if payment['form'] is defined %}
            <div id="{{ payment.code }}_form">
              {{ form_widget(payment['form']) }}
            </div>
            {% endif %}
          </li>
          {% endfor %}
        </ul>
      </fieldset>
    </form>
  </div>
  <div class="large-4 columns">
    {% include 'FoggylineSalesBundle:default:checkout/order_sumarry.html.twig' %}
    <div>Cart Subtotal: {{ cart_subtotal }}</div>
    <div>{{ delivery_label }}: {{ delivery_subtotal }}</div>
    <div>Order Total: {{ order_total }}</div>
    <div><a id="payment_form_submit" href="#" class="button">Place Order</a>
    </div>
  </div>
</div>
<script type="text/javascript">
  var form = document.getElementById('payment_form');
  document.getElementById('payment_form_submit').addEventListener('click', function () {
    form.submit();
  });
</script>
{% endblock %}
```

与发货步骤类似，我们在这里有一个可用的支付方式的渲染，同时还有一个 "下订单 "按钮，该按钮位于提交表单之外，由JavaScript处理。一旦订单被下达，POST提交就会进入`foggyline_sales_checkout_process`路由，我们在`src/Foggyline/SalesBundle/Resources/config/routing.xml`文件的路由元素中定义了如下路由:

```markup
<route id="foggyline_sales_checkout_process"path="/checkout/process">
  <default key="_controller">FoggylineSalesBundle:Checkout:process</default>
</route>
```

路径指向`CheckoutController`中的`processAction`函数，我们定义如下:

```php
public function processAction()
{
  if ($customer = $this->getUser()) {

    $em = $this->getDoctrine()->getManager();
    // Merge all the checkout info, for SalesOrder
    $checkoutInfo = $this->get('session')->get('checkoutInfo');
    $now = new \DateTime();

    // Create Sales Order
    $salesOrder = new \Foggyline\SalesBundle\Entity\SalesOrder();
    $salesOrder->setCustomer($customer);
    $salesOrder->setItemsPrice($checkoutInfo['items_price']);
    $salesOrder->setShipmentPrice
      ($checkoutInfo['shipment_price']);
    $salesOrder->setTotalPrice($checkoutInfo['total_price']);
    $salesOrder->setPaymentMethod($_POST['payment_method']);
    $salesOrder->setShipmentMethod($checkoutInfo['shipment_method']);
    $salesOrder->setCreatedAt($now);
    $salesOrder->setModifiedAt($now);
    $salesOrder->setCustomerEmail($customer->getEmail());
    $salesOrder->setCustomerFirstName($customer->getFirstName());
    $salesOrder->setCustomerLastName($customer->getLastName());
    $salesOrder->setAddressFirstName($checkoutInfo['address_first_name']);
    $salesOrder->setAddressLastName($checkoutInfo['address_last_name']);
    $salesOrder->setAddressCountry($checkoutInfo['address_country']);
    $salesOrder->setAddressState($checkoutInfo['address_state']);
    $salesOrder->setAddressCity($checkoutInfo['address_city']);
    $salesOrder->setAddressPostcode($checkoutInfo['address_postcode']);
    $salesOrder->setAddressStreet($checkoutInfo['address_street']);
    $salesOrder->setAddressTelephone($checkoutInfo['address_telephone']);
    $salesOrder->setStatus(\Foggyline\SalesBundle\Entity\SalesOrder::STATUS_PROCESSING);

    $em->persist($salesOrder);
    $em->flush();

    // Foreach cart item, create order item, and delete cart item
    $cart = $em->getRepository('FoggylineSalesBundle:Cart')->findOneBy(array('customer' => $customer));
    $items = $cart->getItems();

    foreach ($items as $item) {
      $orderItem = new \Foggyline\SalesBundle\Entity\SalesOrderItem();

      $orderItem->setSalesOrder($salesOrder);
      $orderItem->setTitle($item->getProduct()->getTitle());
      $orderItem->setQty($item->getQty());
      $orderItem->setUnitPrice($item->getUnitPrice());
      $orderItem->setTotalPrice($item->getQty() * $item->getUnitPrice());
      $orderItem->setModifiedAt($now);
      $orderItem->setCreatedAt($now);
      $orderItem->setProduct($item->getProduct());

      $em->persist($orderItem);
      $em->remove($item);
    }

    $em->remove($cart);
    $em->flush();

    $this->get('session')->set('last_order', $salesOrder->getId());
    return $this->redirectToRoute('foggyline_sales_checkout_success');
  } else {
    $this->addFlash('warning', 'Only logged in customers can access checkout page.');
    return $this->redirectToRoute('foggyline_customer_login');
  }
}
```

一旦POST提交点击控制器，一个包含所有相关项目的新订单就会被创建。同时，购物车和购物车中的物品也会被清空。最后，客户会被重定向到订单成功页面。

### 创建订单成功页面

订单成功页在完整的网络商店应用中具有重要作用。在这里，我们要感谢客户的购买，并可能提出一些更多相关或交叉相关的购物选项，以及一些可选的折扣。虽然我们的应用很简单，但值得构建一个简单的订单成功页面。

我们首先在 `src/Foggyline/SalesBundle/Resources/config/routing.xml` 文件的 `routes` 元素下添加以下路由定义:

```markup
<route id="foggyline_sales_checkout_success" path="/checkout/success">
  <default key="_controller">FoggylineSalesBundle:Checkout:success</default>
</route>
```

路径指向`CheckoutController`中的一个`successAction`函数，我们定义如下:

```php
public function successAction()
{

  return $this->render('FoggylineSalesBundle:default:checkout/success.html.twig', array(
    'last_order' => $this->get('session')->get('last_order')
  ));
}
```

在这里，我们只是简单地获取当前登录客户最后创建的订单 ID，并将完整的订单对象传递给 `src/Foggyline/SalesBundle/Resources/views/Default/checkout/success.html.twig` 模板，如下所示:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<h1>Checkout Success</h1>
<div class="row">
  <p>Thank you for placing your order #{{ last_order }}.</p>
  <p>You can see order details <a href="{{ path('customer_account') }}">here</a>.</p>
</div>
{% endblock %}
```

有了这个，我们就完成了网店的整个结账流程。虽然这是一个绝对简单的，但它为更强大的实现奠定了基础。

### 创建店长仪表盘

现在，我们已经敲定了结账销售模块，让我们快速回复到我们的核心模块--`AppBundle`。根据我们的应用需求，让我们继续创建一个简单的店长仪表盘。

我们先添加`src/AppBundle/Controller/StoreManagerController.php`文件，内容如下:

```php
namespace AppBundle\Controller;

use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class StoreManagerController extends Controller
{
  /**
  * @Route("/store_manager", name="store_manager")
  */
  public function indexAction()
  {
    return $this->render('AppBundle:default:store_manager.html.twig');
  }
}
```

`indexAction`函数只是返回`src/AppBundle/Resources/views/default/store_manager.html.twig`文件，其内容我们定义如下:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<h1>Store Manager</h1>
<div class="row">
  <div class="large-6 columns">
    <div class="stacked button-group">
      <a href="{{ path('category_new') }}" class="button">Add new Category</a>
      <a href="{{ path('product_new') }}" class="button">Add new Product</a>
      <a href="{{ path('customer_new') }}" class="button">Add new Customer</a>
    </div>
  </div>
  <div class="large-6 columns">
    <div class="stacked button-group">
      <a href="{{ path('category_index') }}" class="button">List & Manage Categories</a>
      <a href="{{ path('product_index') }}" class="button">List & Manage Products</a>
      <a href="{{ path('customer_index') }}" class="button">List & Manage Customers</a>
      <a href="{{ path('salesorder_index') }}" class="button">List & Manage Orders</a>
    </div>
  </div>
</div>
{% endblock %}
```

模板只是渲染了类别、产品、客户和订单管理的链接。这些链接的实际访问是由防火墙控制的，这在前面的章节中已经解释过。

## 单元测试

销售模块比之前的任何一个模块都要强大得多。有几件事我们可以进行单元测试。然而，我们不会把完整的单元测试作为本章的一部分。我们将简单地把注意力转向一个单一的单元测试，即`CustomerOrders`服务的单元测试。

我们首先在`phpunit.xml.dist`文件的`testsuites`元素下添加以下一行:

```markup
<directory>src/Foggyline/SalesBundle/Tests</directory>
```

有了这些，从我们的商店根目录下运行`phpunit`命令，就可以在`src/Foggyline/SalesBundle/Tests/`目录下找到我们定义的测试。

现在，让我们继续为`CustomerOrders`服务创建一个测试。我们通过定义`src/Foggyline/SalesBundle/Tests/Service/CustomerOrdersTest.php`文件来进行测试，内容如下:

```php
namespace Foggyline\SalesBundle\Test\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\Security\Core\Authentication\Token\UsernamePasswordToken;

class CustomerOrdersTest extends KernelTestCase
{
  private $container;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
  }

  public function testGetOrders()
  {
    $firewall = 'foggyline_customer';

    $em = $this->container->get('doctrine.orm.entity_manager');

    $user = $em->getRepository('FoggylineCustomerBundle:Customer')->findOneByUsername
      ('ajzele@gmail.com');
    $token = new UsernamePasswordToken($user, null, $firewall, array('ROLE_USER'));

    $tokenStorage = $this->container->get('security.token_storage');
    $tokenStorage->setToken($token);

    $orders = new \Foggyline\SalesBundle\Service\CustomerOrders(
      $em,
      $tokenStorage,
      $this->container->get('router')
    );

    $this->assertNotEmpty($orders->getOrders());
  }
}
```

这里，我们使用`UsernamePasswordToken`函数来模拟客户登录。然后，密码令牌被传递给`CustomerOrders`服务。然后`CustomerOrders`服务在内部检查令牌存储是否有分配的令牌，将其标记为已登录的用户并返回其订单列表。能够模拟客户登录对于我们可能为销售模块编写的任何其他测试都是至关重要的。

## 功能测试

与单元测试类似，我们将只关注单一功能测试，因为做任何更强大的测试都不在本章的范围内。我们将编写一段简单的代码，将商品添加到购物车中，并进入结账页面。为了将商品添加到购物车中，这里我们还需要模拟用户登录。

我们编写`src/Foggyline/SalesBundle/Tests/Controller/CartControllerTest.php`测试如下:

```php
namespace Foggyline\SalesBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\BrowserKit\Cookie;
use Symfony\Component\Security\Core\Authentication\Token\UsernamePasswordToken;

class CartControllerTest extends WebTestCase
{
  private $client = null;

  public function setUp()
  {
    $this->client = static::createClient();
  }

  public function testAddToCartAndAccessCheckout()
  {
    $this->logIn();

    $crawler = $this->client->request('GET', '/');
    $crawler = $this->client->click($crawler->selectLink('Add to Cart')->link());
    $crawler = $this->client->followRedirect();

    $this->assertTrue($this->client->getResponse()->isSuccessful());
    $this->assertGreaterThan(0, $crawler->filter('html:contains("added to cart")')->count());

    $crawler = $this->client->request('GET', '/sales/cart/');
    $crawler = $this->client->click($crawler->selectLink('Go to Checkout')->link());


    $this->assertTrue($this->client->getResponse()->isSuccessful());
    $this->assertGreaterThan(0, $crawler->filter('html:contains("Checkout")')->count());
  }

  private function logIn()
  {
    $session = $this->client->getContainer()->get('session');
    $firewall = 'foggyline_customer'; // firewall name
    $em = $this->client->getContainer()->get('doctrine')->getManager();
    $user = $em->getRepository('FoggylineCustomerBundle:Customer')->findOneByUsername('ajzele@gmail.com');

    $token = new UsernamePasswordToken($user, null, $firewall, array('ROLE_USER'));
    $session->set('_security_' . $firewall, serialize($token));
    $session->save();

    $cookie = new Cookie($session->getName(), $session->getId());
    $this->client->getCookieJar()->set($cookie);
  }
}
```

运行后，测试将模拟客户登录，向购物车中添加商品，并尝试进入结账页面。根据数据库中的实际客户情况，我们可能需要更改前面测试中提供的客户邮箱。

现在运行`phpunit`命令应该可以成功执行我们的测试。

## 小结

在本章中，我们构建了一个简单而实用的销售模块。只用四个简单的实体（`Cart、CartItem、SalesOrder和SalesOrderItem`），我们就实现了简单的购物车和结账功能。通过这样做，我们让客户实际进行购买，而不仅仅是浏览产品目录。销售模块使用了前面章节中定义的支付和发货服务。虽然支付和发货服务是作为虚拟的服务来实现的，但它们确实提供了一个基本的骨架，我们可以用来实现真正的支付和装运API。

此外，在这一章中，我们解决了管理员仪表板的问题，做了一个简单的接口，只是聚合了一些现有的CRUD接口。通过`app/config/security.yml`中的条目来保护对仪表盘和管理链接的访问，并且只允许`ROLE_ADMIN`访问。

到目前为止，所写的模块共同组成了一个简化的应用程序。编写健壮的网店应用程序通常会包括现代电子商务平台（如Magento）中发现的数十种其他功能。这些功能包括多种语言、货币和网站支持；强大的类别、产品和产品库存管理；购物车和目录销售规则；以及其他许多功能。模块化我们的应用程序使开发和维护过程变得更加容易。

