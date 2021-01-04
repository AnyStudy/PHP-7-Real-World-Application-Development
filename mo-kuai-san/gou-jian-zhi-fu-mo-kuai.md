# 构建支付模块

支付模块为我们网店的进一步销售功能提供了基础。它可以让我们在到达即将到来的销售模块的结账过程中，实际选择支付方式。支付方式一般可以有多种类型。有的可以是静态的，比如支票汇款和货到付款，有的可以是常规的信用卡，比如Visa、MasterCard、American Express、Discover和Switch/Solo。在本章中，我们将对这两种类型进行阐述。

在本章中，我们将涉及以下主题:

* 要求
* 依赖
* 实施
* 单元测试
* 功能测试

## 要求

我们的应用需求模块化网店应用需求规范中定义，并没有真正说明我们需要实现的支付方式类型。因此，在本章中，我们将开发两种支付方式：信用卡支付和支票支付。关于信用卡支付，我们不会连接到一个真正的支付处理器，但其他的一切都会像我们使用信用卡一样进行。

理想情况下，我们希望通过一个类似于下面的接口来完成:

```php
namespace Foggyline\SalesBundle\Interface;

interface Payment
{
  function authorize();
  function capture();
  function cancel();
}
```

这样一来，我们就需要有`SalesBundle`模块，而我们还没有开发这个模块。因此，我们将使用一个简单的 Symfony 控制器类来处理我们的支付方法，该类提供了自己的方法来解决以下功能：

* `authorize();`
* `capture();`
* `cancel();`

`authorize`方法用于我们只想授权交易，而不实际执行交易的情况。其结果是一个交易ID，我们未来的`SalesBundle`模块可以存储并重用这个ID，用于进一步的 `capture` 和 `cancel` 操作。 `capture` 方法让我们更进一步，首先执行授权动作，然后捕获资金。 `cancel` 方法则根据之前存储的授权令牌执行取消操作。

我们将通过标记的 Symfony 服务来暴露我们的支付方式。服务的标签是一个很好的功能，它可以让我们查看容器和所有被同一标签标记的服务，我们可以用它来获取所有的`paymentmethod`服务。标签的命名必须遵循一定的模式，这是我们作为应用程序创建者强加给自己的。考虑到这一点，我们将给每个支付服务标记 `name`,`payment_method`。

稍后，`SalesBundle`模块将获取并使用所有用`payment_method`标记的服务，然后在内部使用它们来生成一个可用的支付方式列表，你可以使用它们。

## 依赖

该模块对任何其他模块都没有确定的依赖性。但是，如果先建立`SalesBundle`模块，然后再暴露一些支付模块可能使用的接口，会更方便。

## 实施

我们先创建一个新的模块`Foggyline\PaymentBundle`。我们在控制台的帮助下运行以下命令：

```bash
php bin/console generate:bundle --namespace=Foggyline/PaymentBundle
```

该命令会触发一个互动过程，在这个过程中会问我们几个问题，如下图所示:

![](../.gitbook/assets/image%20%28232%29.png)

完成后，文件`app/AppKernel.php和app/config/routing.yml`会自动修改。`AppKernel`类的`registerBundles`方法被添加到`$bundles`数组下的下面一行：

```php
new Foggyline\PaymentBundle\FoggylinePaymentBundle(),
```

`routing.yml`更新了以下内容:

```yaml
foggyline_payment:
  resource: "@FoggylinePaymentBundle/Resources/config/routing.xml"
  prefix:   /
```

为了避免与核心应用代码发生冲突，我们需要将`prefix: /`改成`prefix: /payment/`。

### 创建卡实体

尽管作为本章的一部分，我们不会在数据库中存储任何信用卡，但我们希望重用 Symfony 自动生成 CRUD 功能，以便它为我们提供信用卡模型和表单。让我们继续创建一个`Card`实体。我们将通过使用控制台来实现，如下所示：

```bash
php bin/console generate:doctrine:entity
```

该命令触发了交互式生成器，为其提供了`FoggylinePaymentBundle:Card`的实体快捷方式，在这里我们还需要提供实体属性。我们要用以下字段来模拟我们的`Card`实体。

* `card_type`: string
* `card_number`: string
* `expiry_date`: date
* `security_code`: string

完成后，生成器在 `src/Foggyline/PaymentBundle/` 目录下创建 `Entity/Card.php` 和 `Repository/CardRepository.php`。现在我们可以更新数据库，让它拉入`Card`实体，如下图所示:

```bash
php bin/console doctrine:schema:update --force
```

有了实体，我们就可以生成它的CRUD了。我们将通过使用以下命令来实现:

```bash
php bin/console generate:doctrine:crud
```

这将导致一个 `src/Foggyline/PaymentBundle/Controller/CardController.php` 文件被创建。它还在我们的`app/config/routing.yml`文件中添加了一个条目，如下所示：

```yaml
foggyline_payment_card:
  resource: "@FoggylinePaymentBundle/Controller/CardController.php"
  type:    annotation
```

同样，视图文件是在`app/Resources/views/card/`目录下创建的。由于我们实际上不会围绕卡片做任何CRUD相关的操作，我们可以继续删除所有生成的视图文件，以及`CardController`类的整个主体。此时，我们应该有我们的`Card`实体、`CardType`表单和空的`CardController`类。

#### 创建信用卡支付服务

卡片支付服务要提供我们未来销售模块在结账过程中所需要的相关信息。它的作用是提供支付方式标签、代码和订单的处理URL，如 `authorize`、 `capture`和`cancel`。

我们先在 `src/Foggyline/PaymentBundle/Resources/config/services.xml` 文件的 `services` 元素下定义以下服务：

```markup
<service id="foggyline_payment.card_payment"class="Foggyline\PaymentBundle\Service\CardPayment">
  <argument type="service" id="form.factory"/>
  <argument type="service" id="router"/>
  <tag name="payment_method"/>
</service>
```

这个服务接受两个参数：一个是`form.factory`，另一个是`router.form.factory`，将在服务中用于创建`CardType`表单的表单视图。标签在这里是一个至关重要的元素，因为我们的`SalesBundle`模块将根据分配给服务的`payment_method`标签来寻找支付方式。

现在我们需要在`src/Foggyline/PaymentBundle/Service/CardPayment.php`文件中创建实际的服务类，如下所示:

```php
namespace Foggyline\PaymentBundle\Service;

use Foggyline\PaymentBundle\Entity\Card;

class CardPayment
{
  private $formFactory;
  private $router;

  public function __construct(
    $formFactory,
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->formFactory = $formFactory;
    $this->router = $router;
  }

  public function getInfo()
  {
    $card = new Card();
    $form = $this->formFactory->create('Foggyline\PaymentBundle\Form\CardType', $card);

    return array(
      'payment' => array(
      'title' =>'Foggyline Card Payment',
      'code' =>'card_payment',
      'url_authorize' => $this->router->generate('foggyline_payment_card_authorize'),
      'url_capture' => $this->router->generate('foggyline_payment_card_capture'),
      'url_cancel' => $this->router->generate('foggyline_payment_card_cancel'),
      'form' => $form->createView()
      )
    );
  }
}
```

`getInfo`方法将为我们未来的`SalesBundle`模块提供必要的信息，以便它构建结账过程中的支付步骤。我们在这里传递了三种不同类型的URL： `authorize`, `capture`和`cancel`。这些路径现在还不存在，因为我们将很快创建它们。我们的想法是，我们将把支付动作和流程转移到实际的支付方式上。我们未来的`SalesBundle`模块将仅仅对这些支付URL做一个**AJAX POST**，并期望得到一个成功或错误的JSON响应。一个成功的响应应该产生某种交易ID，而一个错误的响应应该产生一个标签信息来显示给用户。

### 创建卡支付控制器和路由

我们将编辑 `src/Foggyline/PaymentBundle/Resources/config/routing.xml` 文件，在其中添加以下路由定义：

```markup
<route id="foggyline_payment_card_authorize" path="/card/authorize">
  <default key="_controller">FoggylinePaymentBundle:Card:authorize</default>
</route>

<route id="foggyline_payment_card_capture" path="/card/capture">
  <default key="_controller">FoggylinePaymentBundle:Card:capture</default>
</route>

<route id="foggyline_payment_card_cancel" path="/card/cancel">
  <default key="_controller">FoggylinePaymentBundle:Card:cancel</default>
</route>
```

然后，我们将编辑`CardController`类的主体，在其中添加以下内容：

```php
public function authorizeAction(Request $request)
{
  $transaction = md5(time() . uniqid()); // Just a dummy string, simulating some transaction id, if any

  if ($transaction) {
    return new JsonResponse(array(
      'success' => $transaction
    ));
  }

  return new JsonResponse(array(
    'error' =>'Error occurred while processing Card payment.'
  ));
}

public function captureAction(Request $request)
{
  $transaction = md5(time() . uniqid()); // Just a dummy string, simulating some transaction id, if any

  if ($transaction) {
    return new JsonResponse(array(
      'success' => $transaction
    ));
  }

  return new JsonResponse(array(
    'error' =>'Error occurred while processing Card payment.'
  ));
}

public function cancelAction(Request $request)
{
  $transaction = md5(time() . uniqid()); // Just a dummy string, simulating some transaction id, if any

  if ($transaction) {
    return new JsonResponse(array(
      'success' => $transaction
    ));
  }

  return new JsonResponse(array(
    'error' =>'Error occurred while processing Card payment.'
  ));
}
```

现在我们应该能够访问`/app_dev.php/payment/card/authorize`这样的URL，并看到`authorizeAction`的输出。这里给出的实现是虚拟的。在本章中，我们不会连接到一个真正的支付处理API。对我们来说，重要的是销售模块将在结账过程中，通过`payment_method`标签服务的`getInfo`方法的`['payment']['form']`键来呈现任何可能的表单视图。意思是说，结账过程应该在信用卡支付下显示一个信用卡表单。结账的行为将被编码为：如果选择了带有表单的支付，并且点击了下单按钮，该支付表单将阻止结账过程继续进行，直到提交支付表单以授权或捕获支付本身中定义的URL。当我们进入`SalesBundle`模块时，我们将进一步触及这一点。

### 创建支票支付服务

除了信用卡支付方式，我们再去定义一个静态支付，叫做`Check Money`。

我们先在 `src/Foggyline/PaymentBundle/Resources/config/services.xml` 文件的 `services` 元素下定义以下服务:

```markup
<service id="foggyline_payment.check_money"class="Foggyline\PaymentBundle\Service\CheckMoneyPayment">
  <argument type="service" id="router"/>
  <tag name="payment_method"/>
</service>
```

这里定义的服务只接受一个路由器参数。标签名称与卡支付服务相同。

然后我们将创建`src/Foggyline/PaymentBundle/Service/CheckMoneyPayment.php`文件，内容如下:

```php
namespace Foggyline\PaymentBundle\Service;

class CheckMoneyPayment
{
  private $router;

  public function __construct(
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->router = $router;
  }

  public function getInfo()
  {
    return array(
      'payment' => array(
        'title' =>'Foggyline Check Money Payment',
        'code' =>'check_money',
        'url_authorize' => $this->router->generate('foggyline_payment_check_money_authorize'),
        'url_capture' => $this->router->generate('foggyline_payment_check_money_capture'),
        'url_cancel' => $this->router->generate('foggyline_payment_check_money_cancel'),
        //'form' =>''
      )
    );
  }
}
```

与卡支付不同的是，支票货币支付在`getInfo`方法下没有定义表单键。这是因为没有信用卡条目供它定义。它只是要成为一个静态的支付方法。然而，我们仍然需要定义 `authorize`, `capture`和 `cancel` URL，尽管它们的实现可能只是一个简单的带有成功或错误键的JSON响应。

### 创建支票支付控制器和路由

一旦支票支付服务到位，我们就可以继续为它创建必要的路由。我们将首先在 `src/Foggyline/PaymentBundle/Resources/config/routing.xml` 文件中添加以下路由定义:

```markup
<route id="foggyline_payment_check_money_authorize"path="/check_money/authorize">
  <default key="_controller">FoggylinePaymentBundle:CheckMoney:authorize</default>
</route>

<route id="foggyline_payment_check_money_capture"path="/check_money/capture">
  <default key="_controller">FoggylinePaymentBundle:CheckMoney:capture</default>
</route>

<route id="foggyline_payment_check_money_cancel"path="/check_money/cancel">
  <default key="_controller">FoggylinePaymentBundle:CheckMoney:cancel</default>
</route>
```

然后我们将创建`src/Foggyline/PaymentBundle/Controller/CheckMoneyController.php`文件，内容如下:

```php
namespace Foggyline\PaymentBundle\Controller;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class CheckMoneyController extends Controller
{
  public function authorizeAction(Request $request)
  {
    $transaction = md5(time() . uniqid());
    return new JsonResponse(array(
      'success' => $transaction
    ));
  }

  public function captureAction(Request $request)
  {
    $transaction = md5(time() . uniqid());
    return new JsonResponse(array(
      'success' => $transaction
    ));
  }

  public function cancelAction(Request $request)
  {
    $transaction = md5(time() . uniqid());
    return new JsonResponse(array(
      'success' => $transaction
    ));
  }
}
```

类似于银行卡支付，这里我们添加了一个简单的 `authorize`, `capture`和 `cancel` 方法的虚拟实现。这些方法的响应将在后面反馈到`SalesBundle`模块中。我们可以很容易地从这些方法中实现更强大的功能，但这不在本章的范围内。

## 单元测试

我们的`FoggylinePaymentBundle`模块非常简单。它只提供两种支付方式：卡和支票支付。它通过两个简单的服务类来实现。由于我们不打算进行完整的代码覆盖测试，我们将只覆盖`CardPayment`和`CheckMoneyPayment`服务类作为单元测试的一部分。

我们首先在`phpunit.xml.dist`文件的`testuites`元素下添加以下一行。

```markup
<directory>src/Foggyline/PaymentBundle/Tests</directory>
```

有了这些，从我们的商店根目录下运行`phpunit`命令，就可以在`src/Foggyline/PaymentBundle/Tests/`目录下找到我们定义的测试。

现在，让我们继续为`CardPayment`服务创建一个测试。我们将创建一个 `src/Foggyline/PaymentBundle/Tests/Service/CardPaymentTest.php` 文件，内容如下:

```php
namespace Foggyline\PaymentBundle\Tests\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class CardPaymentTest extends KernelTestCase
{
  private $container;
  private $formFactory;
  private $router;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
    $this->formFactory = $this->container->get('form.factory');
    $this->router = $this->container->get('router');
  }

  public function testGetInfoViaService()
  {
    $payment = $this->container->get('foggyline_payment.card_payment');
    $info = $payment->getInfo();
    $this->assertNotEmpty($info);
    $this->assertNotEmpty($info['payment']['form']);
  }

  public function testGetInfoViaClass()
  {
    $payment = new \Foggyline\PaymentBundle\Service\CardPayment(
       $this->formFactory,
       $this->router
    );

    $info = $payment->getInfo();
    $this->assertNotEmpty($info);
    $this->assertNotEmpty($info['payment']['form']);
  }
}
```

这里，我们正在运行两个简单的测试，看看我们是否可以通过容器或直接实例化一个服务，并简单地调用它的`getInfo`方法。该方法将返回一个包含`['payment']['form']`键的响应。

现在，让我们继续为`CheckMoneyPayment`服务创建一个测试。我们将创建一个 `src/Foggyline/PaymentBundle/Tests/Service/CheckMoneyPaymentTest.php` 文件，内容如下:

```php
namespace Foggyline\PaymentBundle\Tests\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class CheckMoneyPaymentTest extends KernelTestCase
{
  private $container;
  private $router;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
    $this->router = $this->container->get('router');
  }

  public function testGetInfoViaService()
  {
    $payment = $this->container->get('foggyline_payment.check_money');
    $info = $payment->getInfo();
    $this->assertNotEmpty($info);
  }

  public function testGetInfoViaClass()
  {
    $payment = new \Foggyline\PaymentBundle\Service\CheckMoneyPayment(
        $this->router
      );

    $info = $payment->getInfo();
    $this->assertNotEmpty($info);
  }
}
```

同样，这里我们也有两个简单的测试：一个是通过容器获取支付方法，另一个是直接通过类获取。不同的是，我们并没有在`getInfo`方法响应下检查是否存在表单键。

## 功能测试

我们的模块有两个控制器类，我们想测试它们的响应。我们要确保`CardController`和`CheckMoneyController`类的 `authorize`, `capture`和 `cancel` 方法能够正常工作。

我们首先创建`asrc/Foggyline/PaymentBundle/Tests/Controller/CardControllerTest.php`文件，内容如下:

```php
namespace Foggyline\PaymentBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;


class CardControllerTest extends WebTestCase
{
  private $client;
  private $router;

  public function setUp()
  {
    $this->client = static::createClient();
    $this->router = $this->client->getContainer()->get('router');
  }

  public function testAuthorizeAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_payment_card_authorize'));
    $this->assertTests();
  }

  public function testCaptureAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_payment_card_capture'));
    $this->assertTests();
  }

  public function testCancelAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_payment_card_cancel'));
    $this->assertTests();
  }

  private function assertTests()
  {
    $this->assertSame(200, $this->client->getResponse()->getStatusCode());
    $this->assertSame('application/json', $this->client->getResponse()->headers->get('Content-Type'));
    $this->assertContains('success', $this->client->getResponse()->getContent());
    $this->assertNotEmpty($this->client->getResponse()->getContent());
  }
}
```

然后我们创建`src/Foggyline/PaymentBundle/Tests/Controller/CheckMoneyControllerTest.php`，内容如下:

```php
namespace Foggyline\PaymentBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class CheckMoneyControllerTest extends WebTestCase
{
  private $client;
  private $router;

  public function setUp()
  {
    $this->client = static::createClient();
    $this->router = $this->client->getContainer()->get('router');
  }

  public function testAuthorizeAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_payment_check_money_authorize'));
    $this->assertTests();
  }

  public function testCaptureAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_payment_check_money_capture'));
    $this->assertTests();
  }

  public function testCancelAction()
  {
    $this->client->request('GET', $this->router->generate('foggyline_payment_check_money_cancel'));
    $this->assertTests();
  }

  private function assertTests()
  {
    $this->assertSame(200, $this->client->getResponse()->getStatusCode());
    $this->assertSame('application/json', $this->client->getResponse()->headers->get('Content-Type'));
    $this->assertContains('success', $this->client->getResponse()->getContent());
    $this->assertNotEmpty($this->client->getResponse()->getContent());
  }
}
```

这两个测试几乎是相同的。它们包含对 `authorize`, `capture`和 `cancel` 方法的测试。由于我们的方法是通过一个固定的成功JSON响应来实现的，所以这里没有任何意外。然而，我们可以很容易地通过扩展我们的支付方法来玩转它，使其变得更加强大。

## 小结

在本章中，我们建立了一个支付模块，有两种支付方式。卡片支付方式是这样制作的，它是在模拟涉及信用卡的支付。为此，它包括一个表单作为其`getInfo`方法的一部分。另一方面，支票支付是模拟静态支付方式--不包含任何形式的信用卡。这两个方法都是作为虚方法实现的，这意味着它们实际上并没有与任何外部支付处理器进行通信。

我们的想法是创建一个最小的结构，展示如何开发一个简单的支付模块，以便进一步定制。我们通过标记服务来暴露每个支付方法。使用`payment_method`标签是一个共识问题，因为我们是构建完整应用的人，所以我们可以选择如何在销售模块中实现这一点.通过为每个支付方法使用相同的标签名称，我们有效地创造了条件，使未来的销售模块能够选择所有的支付方法，并在其结账流程下呈现它们。

在下一章中，我们将构建发货模块。



