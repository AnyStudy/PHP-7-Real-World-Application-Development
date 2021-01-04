# 构建客户模块

客户模块为我们网店的进一步销售功能提供了基础。在最基本的层面上，它负责注册、登录、管理和显示相关客户信息。它是后面销售模块的要求，它为我们的网店应用增加了实际的销售功能。

在本章中，我们将涉及以下主题:

* 要求
* 依赖性
* 实施
* 单元测试
* 功能测试

## 要求

按照前面的模块化网店应用需求规范中定义的高级应用需求，我们的模块将有一个单一的客户实体定义。

客户实体包括以下属性:

* `id`: integer, auto-increment
* `email`: string, unique
* `username`: string, unique, 登录系统所需
* `password`: string
* `first_name`: string
* `last_name`: string
* `company`: string
* `phone_number`: string
* `country`: string
* `state`: string
* `city`: string
* `postcode`: string
* `street`: string

在本章中，除了添加 Custome r实体及其 CRUD 页面外，我们还需要解决登录、注册、忘记密码页面的创建，以及覆盖一个核心模块服务，负责构建客户菜单。

## 依赖

该模块对其他模块没有明确的依赖。虽然它确实覆盖了核心模块中定义的服务，但模块本身并不依赖于它。此外，一些安全配置需要作为核心应用程序的一部分来提供，我们将在后面看到。

## 实施

我们首先创建一个新的模块`Foggyline\CustomerBundle`。我们在控制台的帮助下，通过运行以下命令来实现：

```bash
php bin/console generate:bundle --namespace=Foggyline/CustomerBundle
```

该命令会触发一个交互式过程，沿途向我们询问几个问题，如下截图所示:

![](../.gitbook/assets/image%20%28230%29.png)

做完之后，我们就会生成以下结构:

![](../.gitbook/assets/image%20%28228%29.png)

如果我们现在看一下 `app/AppKernel.php` 文件，我们会在 `registerBundles` 方法下看到以下一行：

```php
new Foggyline\CustomerBundle\FoggylineCustomerBundle()
```

同样，`app/config/routing.yml` 目录中也添加了以下路由定义：

```yaml
foggyline_customer:
  resource: "@FoggylineCustomerBundle/Resources/config/routing.xml"
  prefix:   /
```

这里我们需要把 `prefix: /` 改成 `prefix: /customer/`，这样我们就不会和核心模块的路由发生冲突。如果将其保留为 `prefix: /`会简单地覆盖我们的核心 AppBundle，并从`src/Foggyline/CustomerBundle/Resources/views/Default/index.html.twig`模板中输出`Hello World！`这时就会输出给浏览器。我们希望保持良好的分离。这意味着模块不为自己定义根路由。

### 创建客户实体

让我们继续创建一个 `Customer` 实体。我们通过使用控制台来创建，如图所示：

```bash
php bin/console generate:doctrine:entity
```

这个命令会触发交互式生成器，我们需要提供实体属性。一旦完成，生成器会在 `src/Foggyline/CustomerBundle/` 目录下创建 `Entity/Customer.php` 和 `Repository/CustomerRepository.php` 文件。在这之后，我们需要更新数据库，这样它就会拉入`Customer` 实体，运行以下命令：

```bash
php bin/console doctrine:schema:update --force
```

这样就会出现如下截图所示的画面:

![](../.gitbook/assets/image%20%28234%29.png)

实体到位后，我们就可以生成它的 CRUD了。我们通过使用以下命令来实现：

```bash
php bin/console generate:doctrine:crud
```

这样就会有一个交互式的输出，如图所示:

![](../.gitbook/assets/image%20%28233%29.png)

这将导致`src/Foggyline/CustomerBundle/Controller/CustomerController.php`目录被创建。它还在我们的`app/config/routing.yml`文件中添加了一个条目，如下所示:

```yaml
foggyline_customer_customer:
  resource: "@FoggylineCustomerBundle/Controller/CustomerController.php"
  type:     annotation
```

同样，视图文件是在`app/Resources/views/customer/`目录下创建的，这不是我们所期望的。我们希望它们在我们的模块`src/Foggyline/CustomerBundle/Resources/views/Default/customer/`目录下，所以我们需要把它们复制过来。此外，我们需要修改`CustomerController`内的所有`$this->render`调用，将`FoggylineCustomerBundle:default:string` 追加到每个模板路径中。

### 修改安全配置

在我们进一步进行模块内的实际更改之前，让我们想象一下，我们的模块需求强制要求一定的安全配置，以使其工作。这些需求规定我们需要对`app/config/security.yml`文件进行一些修改。首先，我们编辑 `providers` 元素，在其中添加以下内容：

```yaml
foggyline_customer:
  entity:
    class: FoggylineCustomerBundle:Customer
  property: username
```

这就有效地将我们的 `Customer` 类定义为安全提供者，而 `username` 元素则是存储用户身份的属性。

然后我们在 `encoders` 元素下定义编码器类型，如下所示：

```yaml
Foggyline\CustomerBundle\Entity\Customer:
  algorithm: bcrypt
  cost: 12
```

这告诉 Symfony 在加密密码时使用 `bcrypt` 算法，算法成本值为12。这样我们的密码在保存到数据库中的时候就不会以明文的形式出现了。

然后我们继续在firewalls元素下定义一个新的防火墙条目，如下所示:

```yaml
foggyline_customer:
  anonymous: ~
  provider: foggyline_customer
  form_login:
    login_path: foggyline_customer_login
    check_path: foggyline_customer_login
    default_target_path: customer_account
  logout:
    path:   /customer/logout
    target: /
```

这里有不少的事情。我们的防火墙使用 `anonymous: ~`定义来表示它并不真正需要用户登录才能看到某些页面。默认情况下，所有的 Symfony 用户都是匿名认证的，如下截图所示，在开发者工具栏上:

![](../.gitbook/assets/image%20%28231%29.png)

`form_login` 的定义有三个属性，其中`login_path`和`check_path`指向我们的自定义路由`foggyline_customer_login`。当安全系统启动认证过程时，它会将用户重定向到`foggyline_customer_login`路由，在那里我们将很快实现所需的控制器逻辑和视图模板，以处理登录表单。登录后，`default_target_path`决定用户将被重定向到哪里。

最后，我们重用 Symfony 的匿名用户功能，以排除某些页面被禁止。我们希望我们的非认证用户能够访问登录、注册和忘记密码页面。为了实现这一点，我们在 `access_control` 元素下添加了以下内容：

```yaml
- { path: customer/login, roles: IS_AUTHENTICATED_ANONYMOUSLY }
- { path: customer/register, roles: IS_AUTHENTICATED_ANONYMOUSLY }
- { path: customer/forgotten_password, roles: IS_AUTHENTICATED_ANONYMOUSLY }
- { path: customer/account, roles: ROLE_USER }
- { path: customer/logout, roles: ROLE_USER }
- { path: customer/, roles: ROLE_ADMIN }
```

值得注意的是，这种处理模块和基础应用之间的安全问题的方法是迄今为止最理想的方法。这只是一个可能的例子，说明我们如何实现这个模块所需要的功能。

### 扩展客户实体

有了前面的 `security.yml` 的添加，我们现在就可以开始实际执行注册过程了。首先，我们编辑`src/Foggyline/CustomerBundle/Entity/`目录下的 `Customer` 实体，让它实现`Symfony/Component/Security/Core/User/UserInterface, \Serializable`。这意味着要实现以下方法：

```php
public function getSalt()
{
  return null;
}

public function getRoles()
{
  return array('ROLE_USER');
}

public function eraseCredentials()
{
}

public function serialize()
{
  return serialize(array(
    $this->id,
    $this->username,
    $this->password
  ));
}

public function unserialize($serialized)
{
  list (
    $this->id,
    $this->username,
    $this->password,
  ) = unserialize($serialized);
}
```

尽管所有的密码都需要用盐来进行哈希，但在这种情况下，`getSalt` 函数是无关紧要的，因为 `bcrypt` 内部已经完成了这项工作。`getRoles`函数是重要的一点。我们可以返回单个客户将拥有的一个或多个角色。为了使事情简单化，我们将只为每个客户分配一个 `ROLE_USER` 角色。但这可以很容易地变得更加强大，这样角色也会存储在数据库中。`eraseCredentials`函数只是一个清理方法，我们将其留空。

由于用户对象每次请求都要先进行未序列化、序列化，并保存到会话中，所以我们实现了 `\Serializable` 接口。实际实现序列化和`unserialize`可以只包括一部分客户属性，因为我们不需要把所有的东西都保存在session中。

在我们开始实现注册、登录、忘记密码等位之前，我们先来定义一下我们后面要使用的需要的服务。

### 创建订单服务

我们将创建一个订单服务，用于填写 "我的账户 "页面下的可用数据。稍后，其他模块可以覆盖这个服务并注入真实的客户订单。要定义一个订单服务，我们编辑 `src/Foggyline/CustomerBundle/Resources/config/services.xml` 文件，在 `services` 元素下添加以下内容：

```markup
<service id="foggyline_customer.customer_orders" class="Foggyline\CustomerBundle\Service\CustomerOrders">
</service>
```

然后，我们继续创建 `src/Foggyline/CustomerBundle/Service/CustomerOrders.php`目录，内容如下:

```php
namespace Foggyline\CustomerBundle\Service;

class CustomerOrders
{
  public function getOrders()
  {
    return array(
      array(
        'id' => '0000000001',
        'date' => '23/06/2016 18:45',
        'ship_to' => 'John Doe',
        'order_total' => 49.99,
        'status' => 'Processing',
        'actions' => array(
          array(
            'label' => 'Cancel',
            'path' => '#'
          ),
          array(
            'label' => 'Print',
            'path' => '#'
          )
        )
      ),
    );
  }
}
```

`getOrders` 方法在这里只是返回一些虚数据。我们可以很容易地让它返回一个空数组。理想情况下，我们希望它返回一个符合某些特定接口的特定类型元素的集合。

### 创建客户菜单服务

在上一个模块中，我们定义了一个客户服务，在客户菜单中填充一些虚拟数据。现在我们将创建一个覆盖服务，根据客户登录状态，用实际的客户数据填充菜单。要定义一个客户菜单服务，我们编辑`src/Foggyline/CustomerBundle/Resources/config/services.xml`文件，在`services`元素下添加以下内容：

```markup
<service id="foggyline_customer.customer_menu" class="Foggyline\CustomerBundle\Service\Menu\CustomerMenu">
  <argument type="service" id="security.token_storage"/>
  <argument type="service" id="router"/>
</service>
```

这里我们将`token_storage`和 `router` 对象注入到我们的服务中，因为我们需要它们根据客户的登录状态来构建菜单。

然后我们继续创建`src/Foggyline/CustomerBundle/Service/Menu/CustomerMenu.php`，内容如下：

```php
namespace Foggyline\CustomerBundle\Service\Menu;

class CustomerMenu
{
  private $token;
  private $router;

  public function __construct(
    $tokenStorage,
    \Symfony\Bundle\FrameworkBundle\Routing\Router $router
  )
  {
    $this->token = $tokenStorage->getToken();
    $this->router = $router;
  }

  public function getItems()
  {
    $items = array();
    $user = $this->token->getUser();

    if ($user instanceof \Foggyline\CustomerBundle\Entity\Customer) {
      // customer authentication
      $items[] = array(
        'path' => $this->router->generate('customer_account'),
        'label' => $user->getFirstName() . ' ' . $user->getLastName(),
      );
      $items[] = array(
        'path' => $this->router->generate('customer_logout'),
        'label' => 'Logout',
      );
    } else {
      $items[] = array(
        'path' => $this->router->generate('foggyline_customer_login'),
        'label' => 'Login',
      );
      $items[] = array(
        'path' => $this->router->generate('foggyline_customer_register'),
        'label' => 'Register',
      );
    }

    return $items;
  }
}
```

在这里，我们看到一个基于用户登录状态的菜单被构建。这样一来，客户在登录时可以看到**Logout**链接，在未登录时可以看到**Login**。

然后我们添加`src/Foggyline/CustomerBundle/DependencyInjection/Compiler/OverrideServiceCompilerPass.php`，内容如下：

```php
namespace Foggyline\CustomerBundle\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class OverrideServiceCompilerPass implements CompilerPassInterface
{
  public function process(ContainerBuilder $container)
  {
    // Override the core module 'onsale' service
    $container->removeDefinition('customer_menu');
    $container->setDefinition('customer_menu', $container->getDefinition('foggyline_customer.customer_menu'));
  }
}
```

这里我们做的是实际的`customer_menu`服务覆盖。然而，这将不会启动，直到我们编辑`src/Foggyline/CustomerBundle/FoggylineCustomerBundle.php`，添加构建方法如下：

```php
namespace Foggyline\CustomerBundle;

use Symfony\Component\HttpKernel\Bundle\Bundle;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Foggyline\CustomerBundle\DependencyInjection\Compiler\OverrideServiceCompilerPass;

class FoggylineCustomerBundle extends Bundle
{
  public function build(ContainerBuilder $container)
  {
    parent::build($container);;
    $container->addCompilerPass(new OverrideServiceCompilerPass());
  }
}
```

`addCompilerPass`方法调用接受我们的`OverrideServiceCompilerPass`的实例，确保我们的服务覆盖会启动。

### 实施注册程序

为了实现注册页面，我们首先修改`src/Foggyline/CustomerBundle/Controller/CustomerController.php`文件如下：

```php
/**
 * @Route("/register", name="foggyline_customer_register")
 */
public function registerAction(Request $request)
{
  // 1) build the form
  $user = new Customer();
  $form = $this->createForm(CustomerType::class, $user);

  // 2) handle the submit (will only happen on POST)
  $form->handleRequest($request);
  if ($form->isSubmitted() && $form->isValid()) {

    // 3) Encode the password (you could also do this via Doctrine listener)
    $password = $this->get('security.password_encoder')
    ->encodePassword($user, $user->getPlainPassword());
    $user->setPassword($password);

    // 4) save the User!
    $em = $this->getDoctrine()->getManager();
    $em->persist($user);
    $em->flush();

    // ... do any other work - like sending them an email, etc
    // maybe set a "flash" success message for the user

    return $this->redirectToRoute('customer_account');
  }

  return $this->render(
    'FoggylineCustomerBundle:default:customer/register.html.twig',
    array('form' => $form->createView())
  );
}
```

注册页面使用标准的自动生成的Customer CRUD 表单，只需将其指向`src/Foggyline/CustomerBundle/Resources/views/Default/customer/register.html.twig`模板文件，内容如下：

```markup
{% extends 'base.html.twig' %}
{% block body %}
  {{ form_start(form) }}
  {{ form_widget(form) }}
  <button type="submit">Register!</button>
  {{ form_end(form) }}
{% endblock %}
```

一旦这两个文件到位，我们的注册功能就应该可以使用了。

### 实施登录流程

我们将在自己的`/customer/login` URL上实现登录页面，因此我们编辑`CustomerController.php`文件，添加`loginAction`函数如下:

```php
/**
 * Creates a new Customer entity.
 *
 * @Route("/login", name="foggyline_customer_login")
 */
public function loginAction(Request $request)
{
  $authenticationUtils = $this->get('security.authentication_utils');

  // get the login error if there is one
  $error = $authenticationUtils->getLastAuthenticationError();

  // last username entered by the user
  $lastUsername = $authenticationUtils->getLastUsername();

  return $this->render(
    'FoggylineCustomerBundle:default:customer/login.html.twig',
    array(
      // last username entered by the user
      'last_username' => $lastUsername,
      'error'         => $error,
    )
  );
}
```

在这里，我们只是简单地检查用户是否已经尝试登录，如果是，我们将把这些信息和潜在的错误一起传递给模板。然后我们编辑 `src/Foggyline/CustomerBundle/Resources/views/default/customer/login.html.twig` 文件，内容如下：

```markup
{% extends 'base.html.twig' %}
{% block body %}
{% if error %}
<div>{{ error.messageKey|trans(error.messageData, 'security') }}</div>
{% endif %}

<form action="{{ path('foggyline_customer_login') }}" method="post">
  <label for="username">Username:</label>
  <input type="text" id="username" name="_username" value="{{ last_username }}"/>
  <label for="password">Password:</label>
  <input type="password" id="password" name="_password"/>
  <button type="submit">login</button>
</form>

<div class="row">
  <a href="{{ path('customer_forgotten_password') }}">Forgot your password?</a>
</div>
{% endblock %}
```

一旦登录，用户将被重定向到`/customer/account`页面。我们通过在`CustomerController.php`文件中添加`accountAction`方法来创建这个页面，如下所示:

```php
/**
 * Finds and displays a Customer entity.
 *
 * @Route("/account", name="customer_account")
 * @Method({"GET", "POST"})
 */
public function accountAction(Request $request)
{
  if (!$this->get('security.authorization_checker')->isGranted('ROLE_USER')) {
    throw $this->createAccessDeniedException();
  }

  if ($customer = $this->getUser()) {

    $editForm = $this->createForm('Foggyline\CustomerBundle\Form\CustomerType', $customer, array( 'action' => $this->generateUrl('customer_account')));
    $editForm->handleRequest($request);

    if ($editForm->isSubmitted() && $editForm->isValid()) {
      $em = $this->getDoctrine()->getManager();
      $em->persist($customer);
      $em->flush();

      $this->addFlash('success', 'Account updated.');
      return $this->redirectToRoute('customer_account');
    }

    return $this->render('FoggylineCustomerBundle:default:customer/account.html.twig', array(
    'customer' => $customer,
    'form' => $editForm->createView(),
    'customer_orders' => $this->get('foggyline_customer.customer_orders')->getOrders()
    ));
  } else {
    $this->addFlash('notice', 'Only logged in customers can access account page.');
    return $this->redirectToRoute('foggyline_customer_login');
  }
}
```

使用 `$this->getUser()` 我们检查是否设置了登录用户，如果是，则将其信息传递给模板。然后我们编辑 `src/Foggyline/CustomerBundle/Resources/views/Default/customer/account.html.twig` 文件，内容如下:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<h1>My Account</h1>
{{ form_start(form) }}
<div class="row">
  <div class="medium-6 columns">
    {{ form_row(form.email) }}
    {{ form_row(form.username) }}
    {{ form_row(form.plainPassword.first) }}
    {{ form_row(form.plainPassword.second) }}
    {{ form_row(form.firstName) }}
    {{ form_row(form.lastName) }}
    {{ form_row(form.company) }}
    {{ form_row(form.phoneNumber) }}
  </div>
  <div class="medium-6 columns">
    {{ form_row(form.country) }}
    {{ form_row(form.state) }}
    {{ form_row(form.city) }}
    {{ form_row(form.postcode) }}
    {{ form_row(form.street) }}
    <button type="submit">Save</button>
  </div>
</div>
{{ form_end(form) }}
<!-- customer_orders -->
{% endblock %}
```

有了这个，我们解决了 "我的账户 "页面的实际客户信息部分。在当前状态下，这个页面应该呈现一个编辑表单，如下面的截图所示，使我们能够编辑所有的客户信息:

![](../.gitbook/assets/image%20%28235%29.png)

然后，我们对`<！--customer_orders-->`进行处理，将其替换为以下内容:

```markup
{% block customer_orders %}
<h2>My Orders</h2>
<div class="row">
  <table>
    <thead>
      <tr>
        <th width="200">Order Id</th>
        <th>Date</th>
        <th width="150">Ship To</th>
        <th width="150">Order Total</th>
        <th width="150">Status</th>
        <th width="150">Actions</th>
      </tr>
    </thead>
    <tbody>
      {% for order in customer_orders %}
      <tr>
        <td>{{ order.id }}</td>
        <td>{{ order.date }}</td>
        <td>{{ order.ship_to }}</td>
        <td>{{ order.order_total }}</td>
        <td>{{ order.status }}</td>
        <td>
          <div class="small button-group">
            {% for action in order.actions %}
            <a class="button" href="{{ action.path }}">{{ action.label }}</a>
            {% endfor %}
          </div>
        </td>
      </tr>
      {% endfor %}
    /tbody>
  </table>
</div>
{% endblock %}
```

现在，"我的账户 "页面的 "我的订单 "部分应该会出现如下图所示：

![](../.gitbook/assets/image%20%28229%29.png)

这只是来自`src/Foggyline/CustomerBundle/Resources/config/services.xml`中定义的服务的虚拟数据。在后面的章节中，当我们进入销售模块时，我们将确保它覆盖`foggyline_customer.customer_orders`服务，以便在这里插入真实的客户数据。

### 实施登出程序

在定义防火墙时，我们对`security.yml`做了一个修改，就是配置注销路径，我们将其指向`/customer/logout`。这个路径的实现是在`CustomerController.php`文件中完成的，如下所示:

```php
/**
 * @Route("/logout", name="customer_logout")
 */
public function logoutAction()
{

}
```

注意，`logoutAction`方法其实是空的。没有这样的实现。不需要实现，因为 Symfony 会拦截请求并为我们处理注销。但是，我们需要定义这个路由，因为我们从 `system.xml` 文件中引用了它。

### 管理忘记的密码

忘记密码功能要作为一个单独的页面来实现。我们编辑`CustomerController.php`文件，在其中添加忘记密码功能，如下所示:

```php
/**
 * @Route("/forgotten_password", name="customer_forgotten_password")
 * @Method({"GET", "POST"})
 */
public function forgottenPasswordAction(Request $request)
{

  // Build a form, with validation rules in place
  $form = $this->createFormBuilder()
  ->add('email', EmailType::class, array(
    'constraints' => new Email()
  ))
  ->add('save', SubmitType::class, array(
    'label' => 'Reset!',
    'attr' => array('class' => 'button'),
  ))
  ->getForm();

  // Check if this is a POST type request and if so, handle form
  if ($request->isMethod('POST')) {
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
      $this->addFlash('success', 'Please check your email for reset password.');

      // todo: Send an email out to website admin or something...

      return $this->redirect($this->generateUrl('foggyline_customer_login'));
    }
  }

  // Render "contact us" page
  return $this->render('FoggylineCustomerBundle:default:customer/forgotten_password.html.twig', array(
      'form' => $form->createView()
    ));
}
```

这里我们只是检查 HTTP 请求是 GET 还是 POST，然后发送邮件或者加载模板。为了简单起见，我们还没有真正实现电子邮件的实际发送。这是需要在本书之外解决的事情。渲染后的模板指向`src/Foggyline/CustomerBundle/Resources/views/default/customer/forgotten_password.html.twig`文件，内容如下:

```php
{% extends 'base.html.twig' %}
{% block body %}
<div class="row">
  <h1>Forgotten Password</h1>
</div>

<div class="row">
  {{ form_start(form) }}
  {{ form_widget(form) }}
  {{ form_end(form) }}
</div>
{% endblock %}
```

## 单元测试

除了自动生成的 `Customer` 实体和它的CRUD控制器之外，我们只创建了两个自定义服务类作为这个模块的一部分。由于我们并不追求完整的代码覆盖率，所以我们将只覆盖`CustomerOrders`和`CustomerMenu`服务类作为单元测试的一部分。

我们首先在`phpunit.xml.dist`文件的`testsuites`元素下添加以下一行。

```markup
<directory>src/Foggyline/CustomerBundle/Tests</directory>
```

有了这些，从我们的商店根目录下运行`phpunit`命令，就可以在`src/Foggyline/CustomerBundle/Tests/`目录下找到我们定义的测试。

现在让我们继续为`CustomerOrders`服务创建一个测试。我们创建一个 `src/Foggyline/CustomerBundle/Tests/Service/CustomerOrders.php` 文件，内容如下：

```php
namespace Foggyline\CustomerBundle\Tests\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class CustomerOrders extends KernelTestCase
{
  private $container;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
  }

  public function testGetItemsViaService()
  {
    $orders = $this->container->get('foggyline_customer.customer_orders');
    $this->assertNotEmpty($orders->getOrders());
  }

  public function testGetItemsViaClass()
  {
    $orders = new \Foggyline\CustomerBundle\Service\CustomerOrders();
    $this->assertNotEmpty($orders->getOrders());
  }
}
```

这里我们一共有两个测试，一个通过服务实例化类，另一个直接实例化类。我们使用`setUp`方法只是为了设置容器属性，然后在`testGetItemsViaService`方法中重用。

接下来，我们在目录内创建`CustomerMenu`测试，具体如下：

```php
namespace Foggyline\CustomerBundle\Tests\Service\Menu;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class CustomerMenu extends KernelTestCase
{
  private $container;
  private $tokenStorage;
  private $router;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
    $this->tokenStorage = $this->container->get('security.token_storage');
    $this->router = $this->container->get('router');
  }

  public function testGetItemsViaService()
  {
    $menu = $this->container->get('foggyline_customer.customer_menu');
    $this->assertNotEmpty($menu->getItems());
  }

  public function testGetItemsViaClass()
  {
    $menu = new \Foggyline\CustomerBundle\Service\Menu\CustomerMenu(
      $this->tokenStorage,
      $this->router
    );

    $this->assertNotEmpty($menu->getItems());
  }
}
```

现在，如果我们运行`phpunit`命令，我们应该会看到我们的测试和其他测试一起被选中并执行。我们甚至可以通过执行`phpunit`命令来专门针对这两个测试进行全类路径的测试，如图所示：

```bash
phpunit src/Foggyline/CustomerBundle/Tests/Service/CustomerOrders.php
phpunit src/Foggyline/CustomerBundle/Tests/Service/Menu/CustomerMenu.php
```

## 功能测试

自动生成CRUD工具在`src/Foggyline/CustomerBundle/Tests/Controller/`目录下为我们生成了`CustomerControllerTest.php`文件。在上一章中，我们展示了如何向`static::createClient`传递一个认证参数，以使其模拟用户登录。然而，这与我们客户将要使用的登录方式不同。我们不再使用一个基本的HTTP认证，而是一个完整的登录表单。

为了解决登录表单的测试，让我们继续编辑`src/Foggyline/CustomerBundle/Tests/Controller/CustomerControllerTest.php`文件如下：

```php
namespace Foggyline\CustomerBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\BrowserKit\Cookie;
use Symfony\Component\Security\Core\Authentication\Token\UsernamePasswordToken;

class CustomerControllerTest extends WebTestCase
{
  private $client = null;

  public function setUp()
  {
    $this->client = static::createClient();
  }

  public function testMyAccountAccess()
  {
    $this->logIn();
    $crawler = $this->client->request('GET', '/customer/account');

    $this->assertTrue($this->client->getResponse()->
      isSuccessful());
    $this->assertGreaterThan(0, $crawler->filter('html:contains("My Account")')->count());
  }

  private function logIn()
  {
    $session = $this->client->getContainer()->get('session');
    $firewall = 'foggyline_customer'; // firewall name
    $em = $this->client->getContainer()->get('doctrine')->getManager();
    $user = $em->getRepository('FoggylineCustomerBundle:Customer')->findOneByUsername('john@test.loc');
    $token = new UsernamePasswordToken($user, null, $firewall, array('ROLE_USER'));
    $session->set('_security_' . $firewall, serialize($token));
    $session->save();
    $cookie = new Cookie($session->getName(), $session->getId());
    $this->client->getCookieJar()->set($cookie);
  }
}
```

在这里，我们首先创建了`logIn`方法，其目的是模拟登录，通过在会话中设置适当的 `token` 值，并将该session ID通过cookie传递给客户端。然后我们创建了`testMyAccountAccess`方法，该方法首先调用`logIn`方法，然后检查爬虫是否能够访问我的账户页面。这种方法最大的好处是，我们不需要输入用户密码的代码，只需要输入它的用户名。

现在，我们继续解决客户注册表的问题，在`CustomerControllerTest`中加入以下内容：

```php
public function testRegisterForm()
{
  $crawler = $this->client->request('GET', '/customer/register');
  $uniqid = uniqid();
  $form = $crawler->selectButton('Register!')->form(array(
    'customer[email]' => 'john_' . $uniqid . '@test.loc',
    'customer[username]' => 'john_' . $uniqid,
    'customer[plainPassword][first]' => 'pass123',
    'customer[plainPassword][second]' => 'pass123',
    'customer[firstName]' => 'John',
    'customer[lastName]' => 'Doe',
    'customer[company]' => 'Foggyline',
    'customer[phoneNumber]' => '00 385 111 222 333',
    'customer[country]' => 'HR',
    'customer[state]' => 'Osijek',
    'customer[city]' => 'Osijek',
    'customer[postcode]' => '31000',
    'customer[street]' => 'The Yellow Street',
  ));

  $this->client->submit($form);
  $crawler = $this->client->followRedirect();
  //var_dump($this->client->getResponse()->getContent());
  $this->assertGreaterThan(0, $crawler->filter('html:contains("customer/login")')->count());
}
```

在上一章中我们已经看到了一个类似于这个的测试。这里我们只是打开一个客户/注册页面，然后找到一个带有`Register！`标签的按钮，这样我们就可以通过它来获取整个表单。然后我们设置所有需要的表单数据，并模拟表单提交。如果成功，我们观察重定向体，并对其中的预期值进行断言。

现在运行`phpunit`命令应该可以成功执行我们的测试。

## 小结

在本章中，我们构建了一个微型但功能完善的客户模块。该模块假定在我们的`security.yml`文件上做了一定程度的设置，如果我们要重新发布它，可以将其作为模块文档的一部分。这些变化包括定义我们自己的自定义防火墙，以及自定义安全提供者。安全提供者指向我们的`customer`类，而`customer`类又是以符合Symfony `UserInterface`的方式构建的。然后我们构建了一个注册、登录和忘记密码的表单。虽然每一个都带有一套最小的功能，但我们看到了构建一个完全自定义的注册和登录系统是多么简单。

此外，我们还应用了一些前瞻性的思维，通过使用专门定义的服务，在 "我的账户 "页面下设置了 "我的订单 "部分。这是目前最理想的方式，而且它也有一定的作用，因为我们以后会从销售模块中干净利落地覆盖这个服务。

接下来，在下一章，我们将建立一个支付模块。

