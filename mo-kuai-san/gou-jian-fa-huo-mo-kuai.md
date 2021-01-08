# 构建发货模块

发货模块与支付模块一起，为我们网店的进一步销售功能提供了基础。当我们到达即将到来的销售模块的结账流程时，它将使我们能够选择发货方式。与支付类似，发货方式也可以分为静态和动态两种。静态可能意味着一个固定的价格值，甚至是通过一些简单的条件计算出来的，而动态通常意味着与外部API服务的连接。

在本章中，我们将对这两种类型进行接触，并看看如何为实现发货模块设置一个基本结构。

在本章中，我们将涉及到发货模块的以下主题:

* 要求
* 依赖
* 实施
* 单元测试
* 功能测试

## 要求

应用需求在前面的模块化网店App需求规范中定义的，并没有给我们任何具体的说明，我们需要实现什么样的快递方式。因此，在本章中，我们将开发两种发货方式：动态费率发货和定额发货。动态费率装运是作为一种将发货方法与真正的快递商（如UPS、FedEx等）连接起来的方式。然而，它不会实际连接到任何外部API。

理想情况下，我们希望通过类似于下面的接口来实现。

```php
namespace Foggyline\SalesBundle\Interface;

interface Shipment
{
  function getInfo($street, $city, $country, $postcode, $amount, $qty);
  function process($street, $city, $country, $postcode, $amount, $qty);
}
```

然后，`getInfo`方法可以用来获取给定订单信息的可用送货选项，而 `process`方法则会处理选定的送货选项。例如，我们可以让 API 返回 "当日送达\($9.99\)"和 "标准送达\($4.99\) "作为动态费率装运方法下的送货选项。

有了这样的发货接口，就会强加一个`SalesBundle`模块的要求，而我们还没有开发这个模块。因此，我们将继续我们的发货方法，使用 Symfony 控制器来处理流程方法，使用服务来处理`getInfo`方法。

与上一章的支付方法类似，我们将通过标记的 Symfony 服务来公开 `getInfo` 方法。我们将在发货方法中使用的标签是shipment\_method。稍后，在结账过程中，SalesBundle模块将获取所有被标记为shipment\_method的服务，并在内部使用它们作为可用的发货方法列表。

## 依赖

我们正在以相反的方式构建模块。也就是说，我们在对`SalesBundle`模块有任何了解之前就开始构建该模块，因为`SalesBundle`是唯一会使用该模块的模块。考虑到这一点，发货模块与其他模块没有牢固的依赖关系。然而，如果先构建`SalesBundle`模块，然后再暴露一些`shipment`模块可能使用的接口，可能会更方便。

## 实施

我们将从创建一个新的模块`Foggyline\ShipmentBundle`开始。我们将在控制台的帮助下运行以下命令：

```php
php bin/console generate:bundle --namespace=Foggyline/ShipmentBundle
```

该命令会触发一个交互过程，在这个过程中会问我们几个问题，如下图所示:

![](../.gitbook/assets/image%20%28238%29.png)

完成后，文件`app/AppKernel.php`和`app/config/routing.yml`会自动修改。`AppKernel`类的`registerBundles`方法被添加到`$bundles`数组下的下面一行:

```php
new Foggyline\PaymentBundle\FoggylineShipmentBundle(),
```

`routing.yml`文件已经更新，加入了以下条目:

```yaml
foggyline_payment:
  resource: "@FoggylineShipmentBundle/Resources/config/routing.xml"
  prefix:   /
```

为了避免与核心应用代码发生冲突，我们需要更改 `prefix: /`改成`prefix: /shipment/`。

### 创建统一费率的运输服务

统一费率发货服务要提供固定的发货方式，我们的销售模块要在其结账过程中使用。它的作用是提供装运方式标签、代码、交付选项和处理 URL。

我们先在 `src/Foggyline/ShipmentBundle/Resources/config/services.xml` 文件的 `services` 元素下定义以下服务:

```markup
<service id="foggyline_shipment.dynamicrate_shipment"class="Foggyline\ShipmentBundle\Service\DynamicRateShipment">
  <argument type="service" id="router"/>
  <tag name="shipment_method"/>
</service>
```

这个服务只接受一个参数：`router`。`tagname`的值被设置为`shipment_method`，因为我们的`SalesBundle`模块将根据分配给服务的`shipment_method`标签来寻找发货方式。

现在我们将在 `src/Foggyline/ShipmentBundle/Service/FlatRateShipment.php` 文件中创建实际的服务类，如下所示:

```php
namespace Foggyline\ShipmentBundle\Service;
class FlatRateShipment
{
  private $router;

  public function __construct(
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->router = $router;
  }

  public function getInfo($street, $city, $country, $postcode, $amount, $qty)
  {
    return array(
      'shipment' => array(
        'title' =>'Foggyline FlatRate Shipment',
        'code' =>'flat_rate',
        'delivery_options' => array(
        'title' =>'Fixed',
        'code' =>'fixed',
        'price' => 9.99
      ),
      'url_process' => $this->router->generate('foggyline_shipment_flat_rate_process'),
    )
  ;
  }
}
```

`getInfo`方法将为我们未来的`SalesBundle`模块提供必要的信息，以便它能构建结账过程中的发货步骤。它接受一系列参数：`$street、$city、$country、$postcode、$amount和$qty`。`url_process`是我们将插入所选发货方式的URL。然后，我们未来的`SalesBundle`模块将仅仅是对这个URL做一个AJAX POST，期望得到一个成功或错误的JSON响应，这与我们想象中对支付方式做的事情很相似。

### 创建统一费率装运控制器和路由

我们编辑 `src/Foggyline/ShipmentBundle/Resources/config/routing.xml` 文件，在其中添加以下路由定义:

```markup
<route id="foggyline_shipment_flat_rate_process"path="/flat_rate/process">
  <default key="_controller">FoggylineShipmentBundle:FlatRate:process
  </default>
</route>
```

然后我们创建一个`src/Foggyline/ShipmentBundle/Controller/FlatRateController.php`文件，内容如下:

```php
namespace Foggyline\ShipmentBundle\Controller;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class FlatRateController extends Controller
{
  public function processAction(Request $request)
  {
    // Simulating some transaction id, if any
    $transaction = md5(time() . uniqid());

    return new JsonResponse(array(
      'success' => $transaction
    ));
  }
}
```

现在我们应该能够访问一个URL，比如`/app_dev.php/shipment/flat_rate/process`，并看到`processAction`的输出。这里给出的实现是虚拟的。对我们来说，重要的是销售模块在结账过程中，将通过`shipment_method`标签服务的`getInfo`方法推送任何可能的交付选项。意思是，结账过程应该显示平价运费作为一个选项。结账的行为将被编码为，如果没有选择发货方式，它将阻止结账过程的进一步发展。当我们进入到`SalesBundle`模块时，我们将进一步触及这个问题。

### 创建动态费率支付服务

除了统一费率的发货方式，我们再来定义一个动态的发货方式，叫做动态费率。

我们先在 `src/Foggyline/ShipmentBundle/Resources/config/services.xml` 文件的 `services` 元素下定义以下服务:

```markup
<service id="foggyline_shipment.dynamicrate_shipment"class="Foggyline\ShipmentBundle\Service\DynamicRateShipment">
  <argument type="service" id="router"/>
  <tag name="shipment_method"/>
</service>
```

这里定义的服务只接受一个路由器参数。`tag name`与统一费率装运服务相同，我们将创建src/Foggyline/ShipmentBundle/Service/DynamicRateShipment.php文件。

然后我们将创建`src/Foggyline/ShipmentBundle/Service/DynamicRateShipment.php`文件，内容如下:

```php
namespace Foggyline\ShipmentBundle\Service;

class DynamicRateShipment
{
  private $router;

  public function __construct(
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->router = $router;
  }

  public function getInfo($street, $city, $country, $postcode, $amount, $qty)
  {
    return array(
      'shipment' => array(
        'title' =>'Foggyline DynamicRate Shipment',
        'code' =>'dynamic_rate_shipment',
        'delivery_options' => $this->getDeliveryOptions($street, $city, $country, $postcode, $amount, $qty),
        'url_process' => $this->router->generate('foggyline_shipment_dynamic_rate_process'),
      )
    );
  }

  public function getDeliveryOptions($street, $city, $country, $postcode, $amount, $qty)
  {
    // Imagine we are hitting the API with: $street, $city, $country, $postcode, $amount, $qty
    return array(
      array(
        'title' =>'Same day delivery',
        'code' =>'dynamic_rate_sdd',
        'price' => 9.99
      ),
      array(
        'title' =>'Standard delivery',
        'code' =>'dynamic_rate_sd',
        'price' => 4.99
      ),
    );
  }
}
```

与统一费率发货不同，这里`getInfo`方法的`delivery_options`键是用`getDeliveryOptions`方法的响应来构造的。该方法是服务的内部方法，并没有被想象成是暴露的，也没有被看成是接口的一部分。我们可以很容易地想象在它内部做一些API调用，以便为我们的动态发货方法获取计算的费率。

### 创建动态费率发货控制器和路由

一旦动态费率发货服务到位，我们就可以继续为它创建必要的路由。我们将首先在 `src/Foggyline/ShipmentBundle/Resources/config/routing.xml` 文件中添加以下路径定义:

```markup
<route id="foggyline_shipment_dynamic_rate_process" path="/dynamic_rate/process">
  <default key="_controller">FoggylineShipmentBundle:DynamicRate:process
  </default>
</route>
```

然后我们将创建`src/Foggyline/ShipmentBundle/Controller/DynamicRateController.php`文件，内容如下所示:

```php
namespace Foggyline\ShipmentBundle\Controller;

use Foggyline\ShipmentBundle\Entity\DynamicRate;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\Form\Extension\Core\Type\ChoiceType;

class DynamicRateController extends Controller
{
  public function processAction(Request $request)
  {
    // Just a dummy string, simulating some transaction id
    $transaction = md5(time() . uniqid());

    if ($transaction) {
      return new JsonResponse(array(
'success' => $transaction
      ));
    }

    return new JsonResponse(array(
      'error' =>'Error occurred while processing DynamicRate shipment.'
    ));
  }
}
```

与统一费率发货类似，这里我们添加了一个简单的虚体实现过程和方法。传入的`$request`应该包含与服务`getInfo`方法相同的信息，也就是说，它应该有以下参数可用：`$street、$city、$country、$postcode、$amount和$qty`。方法的响应将在后面反馈到`SalesBundle`模块中。我们可以很容易地从这些方法中实现更强大的功能，但这不在本章的范围内。

## 单元测试

`FoggylineShipmentBundle`模块非常简单。通过只提供两个简单的服务和两个简单的控制器，它很容易测试。

我们首先在`phpunit.xml.dist`文件的`testuites`元素下添加以下一行:

```markup
<directory>src/Foggyline/ShipmentBundle/Tests</directory>
```

有了这些，从商店的根目录下运行`phpunit`命令，就可以在`src/Foggyline/ShipmentBundle/Tests/`目录下找到我们定义的测试。

现在，让我们继续为`FlatRateShipment`服务创建一个测试。我们将创建一个 `src/Foggyline/ShipmentBundle/Tests/Service/FlatRateShipmentTest.php` 文件，内容如下:

```php
namespace Foggyline\ShipmentBundle\Tests\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class FlatRateShipmentTest extends KernelTestCase
{
  private $container;
  private $router;

  private $street = 'Masonic Hill Road';
  private $city = 'Little Rock';
  private $country = 'US';
  private $postcode = 'AR 72201';
  private $amount = 199.99;
  private $qty = 7;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
    $this->router = $this->container->get('router');
  }

  public function testGetInfoViaService()
  {
    $shipment = $this->container->get('foggyline_shipment.flat_rate');

    $info = $shipment->getInfo(
      $this->street, $this->city, $this->country, $this->postcode, $this->amount, $this->qty
    );

    $this->validateGetInfoResponse($info);
  }

  public function testGetInfoViaClass()
  {
    $shipment = new \Foggyline\ShipmentBundle\Service\FlatRateShipment($this->router);

    $info = $shipment->getInfo(
      $this->street, $this->city, $this->country, $this->postcode, $this->amount, $this->qty
    );

    $this->validateGetInfoResponse($info);
  }

  public function validateGetInfoResponse($info)
  {
    $this->assertNotEmpty($info);
    $this->assertNotEmpty($info['shipment']['title']);
    $this->assertNotEmpty($info['shipment']['code']);
    $this->assertNotEmpty($info['shipment']['delivery_options']);
    $this->assertNotEmpty($info['shipment']['url_process']);
  }
}
```

这里正在运行两个简单的测试。一个检查我们是否可以通过容器实例化一个服务，另一个检查我们是否可以直接实例化。一旦被实例化，我们只需调用服务的`getInfo`方法，向它传递一个虚拟地址和订单信息。虽然我们实际上并没有在`getInfo`方法中使用这些数据，但我们需要传递一些东西，否则测试会失败。该方法预计将返回一个响应，其中包含`shipment`键下的几个键，最主要的是 `title`, `code`, `delivery_options`和`url_process`。

现在，让我们继续为我们的`DynamicRateShipment`服务创建一个测试。我们将创建一个 `src/Foggyline/ShipmentBundle/Tests/Service/DynamicRateShipmentTest.php` 文件，内容如下:

```php
namespace Foggyline\ShipmentBundle\Tests\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
class DynamicRateShipmentTest extends KernelTestCase
{
  private $container;
  private $router;

  private $street = 'Masonic Hill Road';
  private $city = 'Little Rock';
  private $country = 'US';
  private $postcode = 'AR 72201';
  private $amount = 199.99;
  private $qty = 7;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
    $this->router = $this->container->get('router');
  }

  public function testGetInfoViaService()
  {
    $shipment = $this->container->get('foggyline_shipment.dynamicrate_shipment');
    $info = $shipment->getInfo(
      $this->street, $this->city, $this->country, $this->postcode, $this->amount, $this->qty
    );
    $this->validateGetInfoResponse($info);
  }

  public function testGetInfoViaClass()
  {
    $shipment = new \Foggyline\ShipmentBundle\Service\DynamicRateShipment($this->router);
    $info = $shipment->getInfo(
      $this->street, $this->city, $this->country, $this->postcode, $this->amount, $this->qty
    );

    $this->validateGetInfoResponse($info);
  }

  public function validateGetInfoResponse($info)
  {
    $this->assertNotEmpty($info);
    $this->assertNotEmpty($info['shipment']['title']);
    $this->assertNotEmpty($info['shipment']['code']);

    // Could happen that dynamic rate has none?!
    //$this->assertNotEmpty($info['shipment']['delivery_options']);

    $this->assertNotEmpty($info['shipment']['url_process']);
  }
}
```

这个测试与`FlatRateShipment`服务的测试几乎相同。在这里，我们也有两个简单的测试：一个是通过容器获取支付方法，另一个是直接通过类获取。不同的是，我们不再断言`delivery_options`不为空的存在。这是因为根据给定的地址和订单信息，真正的API请求可能不会返回任何交付选项。

## 功能测试

我们整个模块有两个控制器类，我们要测试响应。首先要确保`FlatRateController`和`DynamicRateController`类的`process`方法可以访问和工作，我们将首先创建`src/Foggyline/ShipmentBundle/Tests/Controller/FlatRateControllerTest.php`文件。

我们先创建一个`src/Foggyline/ShipmentBundle/Tests/Controller/FlatRateControllerTest.php`文件，内容如下:

```php
namespace Foggyline\ShipmentBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
class FlatRateControllerTest extends WebTestCase
{
  private $client;
  private $router;

  public function setUp()
  {
    $this->client = static::createClient();
    $this->router = $this->client->getContainer()->get('router');
  }

  public function testProcessAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_shipment_flat_rate_process'));
    $this->assertSame(200, $this->client->getResponse()->getStatusCode());
    $this->assertSame('application/json', $this->client->getResponse()->headers->get('Content-Type'));
    $this->assertContains('success', $this->client->getResponse()->getContent());
    $this->assertNotEmpty($this->client->getResponse()->getContent());
  }
}
```

然后我们将创建一个`src/Foggyline/ShipmentBundle/Tests/Controller/DynamicRateControllerTest.php`文件，内容如下:

```php
namespace Foggyline\ShipmentBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class DynamicRateControllerTest extends WebTestCase
{
  private $client;
  private $router;

  public function setUp()
  {
    $this->client = static::createClient();
    $this->router = $this->client->getContainer()->get('router');
  }

  public function testProcessAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_shipment_dynamic_rate_process'));
    $this->assertSame(200, $this->client->getResponse()->getStatusCode());
    $this->assertSame('application/json', $this->client->getResponse()->headers->get('Content-Type'));
    $this->assertContains('success', $this->client->getResponse()->getContent());
    $this->assertNotEmpty($this->client->getResponse()->getContent());
  }
}
```

这两个测试几乎完全相同。它们包含了一个单一的过程动作方法的测试。按照现在的编码，控制器过程动作只是返回一个固定的成功JSON响应。我们可以很容易地扩展它，使其返回的不仅仅是一个固定的响应，并且可以用一个更强大的功能测试来配合这种变化。

## 小结

在本章中，我们建立了一个有两种发货方式的发货模块。每个发货方法都提供了可用的交付选项。统一费率的发货方法在其交付选项下只有一个固定的值，而动态费率方法则从`getDeliveryOptions`方法中获取其值。我们可以很容易地嵌入一个真正的运费API作为`getDeliveryOptions`的一部分，以便提供真正的动态运费选项。

显然，我们在这里缺乏官方的接口，就像我们对支付方法所做的那样。然而，当我们最终确定最终模块时，我们可以随时回来重构我们的应用程序。

与支付方式类似，这里的想法是创建一个最小的结构，展示如何开发一个简单的运输模块，以便进一步定制。使用`shipment_methodservice`标签，我们有效地暴露了未来销售模块的发货方法。

在下一章中，我们将建立一个销售模块，它将最终利用我们的支付和发货模块。

