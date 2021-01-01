# 构建核心模块

到目前为止，我们已经熟悉了PHP 7 的最新变化、设计模式、设计原则和流行的 PHP 框架。我们还更详细地了解了 Symfony 作为我们接下来开发的首选框架。现在我们终于到了可以开始构建模块化应用的阶段。用 Symfony 构建模块化应用是通过 bundles 机制来完成的。从术语上来说，我们会认为 bundle 和模块是一回事。

在本章中，我们将讨论与核心模块有关的以下主题:

* 要求
* 依赖性
* 实施
* 单元测试
* 功能测试

## 要求

回顾前面的模块化网店应用的需求规范，以及展示在那里的线框图，我们可以概述这个模块将具有的一些需求。核心模块将用于设置一般的、应用范围的特性，如下所示:

* 将网站的基础 CSS 包含到项目中
* 建立一个主页
* 构建其他静态页面
* 建立一个联系我们的页面
* 设置一个基本的防火墙，管理员用户可以在其中管理以后从其他模块自动生成的所有 CRUD

## 依赖性

核心模块本身并不依赖于我们将要编写的其他模块，也不依赖于标准 Symfony 安装之外的任何其他第三方模块。

## 实施

我们首先创建一个全新的 Symfony 项目，运行以下控制台命令:

```bash
symfony new shop
```

这将创建一个新的 `shop` 目录，其中包含在浏览器中运行应用程序所需的所有文件。在这些文件和目录中有 `src/AppBundle` 目录，它实际上是我们的核心模块。在我们可以在浏览器中运行我们的应用程序之前，我们需要将新创建的商店目录映射到一个主机名，比如 `shop.app`，这样我们就可以通过 [http://shop.app](http://shop.app) URL 在浏览器中访问它。一旦这样做，如果我们打开[ http://shop.app](%20http://shop.app) ,我们应该会看到**Welcome to Symfony 3.1，**如下所示:

![](../.gitbook/assets/image%20%28216%29.png)

虽然我们现在还不需要数据库，但我们将在后面开发的其他模块中使用数据库连接，因此从一开始就应该设置它。为此，我们使用适当的数据库连接参数配置 `app/config/parameters.yml`。

然后我们从[http://foundation.zurb.com/sites.html](http://foundation.zurb.com/sites.html) 下载Foundation for Sites。下载完成后，我们需要解压并将 `/js` 和 `/css` 目录复制到 `Symfony/web`目录下，如下图所示:

![](../.gitbook/assets/image%20%28217%29.png)

{% hint style="info" %}
值得注意的是，这是我们在模块中使用的Foundation的简化设置，其中我们仅使用CSS和JavaScript文件，而无需设置与Sass相关的任何内容。
{% endhint %}

有了基础 CSS 和 JavaScript 文件，我们编辑 `app/resources/views/base. twig` 文件如下:

```markup
<!doctype html>
<html class="no-js"lang="en">
  <head>
    <meta charset="utf-8"/>
    <meta http-equiv="x-ua-compatible" content="ie=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>{% block title %}Welcome!{% endblock %}</title>
    <link rel="stylesheet"href="{{ asset('css/foundation.css') }}"/>
    {% block stylesheets%}{% endblock %}
  </head>
  <body>
    <!-- START BODY -->
    <!-- TOP-MENU -->
    <!-- SYSTEM-WIDE-MESSAGES -->
    <!-- PER-PAGE-BODY -->
    <!-- FOOTER -->
    <!-- START BODY -->
    <script src="{{ asset('js/vendor/jquery.js') }}"></script>
    <script src="{{ asset('js/vendor/what-input.js') }}"></script>
    <script src="{{ asset('js/vendor/foundation.js') }}"></script>
    <script>
      $(document).foundation();
    </script>
    {% block javascripts%}{% endblock %}
  </body>
</html>
```

在这里，我们将设置整个头部和主体末端区域，并加载所有必需的 CSS 和 JavaScript。 Twigs 标签可帮助我们构建 URL 路径，我们只需传递 URL 路径本身即可为我们构建完整的 URL。 关于页面的实际正文，这里有几件事情要考虑。 我们如何建立类别，客户和结帐菜单？ 在这一点上，我们没有任何这些模块，我们也不想将它们强制转化为我们的核心模块。 那么，如何解决尚未存在的问题呢？

对于类别、顾客和结帐菜单，我们可以做的是为每个菜单项定义 Twig 全局变量，然后使用这些变量来呈现菜单。这些变量将通过适当的服务提交。由于核心 bundle 不知道未来的目录、客户和结帐模块，我们将首先创建一些虚拟服务，并将它们挂接到全局 Twig 变量。稍后，当我们开发目录、客户和结帐模块时，这些模块将覆盖适当的服务，从而为菜单提供正确的值。

这种方法可能不完全适合模块化应用程序的概念，但是它可以满足我们的需要，因为我们不会硬编码任何依赖项。

我们首先在 `app/config/config.yml` 文件中添加以下条目:

```yaml
twig:
# ...
globals:
category_menu: '@category_menu'
customer_menu: '@customer_menu'
checkout_menu: '@checkout_menu'
products_bestsellers: '@bestsellers'
products_onsale: '@onsale'
```

`category_menu_items、customer_menu_items、checkout_menu_items、products_bestsellers和products_onsale`变量成为全局Twig变量，我们可以在任何Twig模板中使用，如下图所示:

```markup
<ul>
  {% for category in category_menu.getItems() %}
  <li>{{ category.name }}</li>
  {% endfor %}
</ul>
```

Twig全局变量配置中的`@`字符用来表示服务名称的开头。这就是将为我们的 Twig 变量提供值对象的服务。接下来，我们继续通过修改 `app/config/services.yml` 来创建实际的`category_menu、customer_menu、checkout_menu、bestsellers和onsale`服务，具体如下:

```yaml
services:
category_menu:
  class: AppBundle\Service\Menu\Category
customer_menu:
  class: AppBundle\Service\Menu\Customer
checkout_menu:
  class: AppBundle\Service\Menu\Checkout
bestsellers:
  class: AppBundle\Service\Menu\BestSellers
onsale:
  class: AppBundle\Service\Menu\OnSale
```

此外，我们在 `src/AppBundle/Service/Menu/` 目录下创建每个列出的服务类。我们从包含以下内容的 `src/AppBundle/Service/Menu/Bestsellers.php` 文件开始:

```php
namespace AppBundle\Service\Menu;

class BestSellers {
  public function getItems() {
    // Note, this can be arranged as per some "Product"interface, so to know what dummy data to return
    return array(
      array('path' =>'iphone', 'name' =>'iPhone', 'img' =>'/img/missing-image.png', 'price' => 49.99, 'add_to_cart_url' =>'#'),
      array('path' =>'lg', 'name' =>'LG', 'img' =>
        '/img/missing-image.png', 'price' => 19.99, 'add_to_cart_url' =>'#'),
      array('path' =>'samsung', 'name' =>'Samsung', 'img'=>'/img/missing-image.png', 'price' => 29.99, 'add_to_cart_url' =>'#'),
      array('path' =>'lumia', 'name' =>'Lumia', 'img' =>'/img/missing-image.png', 'price' => 19.99, 'add_to_cart_url' =>'#'),
      array('path' =>'edge', 'name' =>'Edge', 'img' =>'/img/missing-image.png', 'price' => 39.99, 'add_to_cart_url' =>'#'),
    );
  }
}
```

然后添加 `src/AppBundle/Service/Menu/Category.php` 文件，其内容如下:

```php
class Category {
  public function getItems() {
    return array(
      array('path' =>'women', 'label' =>'Women'),
      array('path' =>'men', 'label' =>'Men'),
      array('path' =>'sport', 'label' =>'Sport'),
    );
  }
}
```

接下来，我们添加 `src/AppBundle/Service/Menu/Checkout.php` 文件，其内容如下所示:

```php
class Checkout
{
  public function getItems()
  {
     // Initial dummy menu
     return array(
       array('path' =>'cart', 'label' =>'Cart (3)'),
       array('path' =>'checkout', 'label' =>'Checkout'),
    );
  }
}
```

完成后，我们将继续向 `src/AppBundle/Service/Menu/Customer.php` 文件添加以下内容:

```php
class Customer
{
  public function getItems()
  {
    // Initial dummy menu
    return array(
      array('path' =>'account', 'label' =>'John Doe'),
      array('path' =>'logout', 'label' =>'Logout'),
    );
  }
}
```

然后添加 `src/AppBundle/Service/Menu/OnSale.php` 文件，其内容如下:

```php
class OnSale
{
  public function getItems()
  {
    // Note, this can be arranged as per some "Product" interface, so to know what dummy data to return
    return array(
      array('path' =>'iphone', 'name' =>'iPhone', 'img' =>'/img/missing-image.png', 'price' => 19.99, 'add_to_cart_url' =>'#'),
      array('path' =>'lg', 'name' =>'LG', 'img' =>'/img/missing-image.png', 'price'      => 29.99, 'add_to_cart_url' =>'#'),
      array('path' =>'samsung', 'name' =>'Samsung', 'img'=>'/img/missing-image.png', 'price' => 39.99, 'add_to_cart_url' =>'#'),
      array('path' =>'lumia', 'name' =>'Lumia', 'img' =>'/img/missing-image.png', 'price' => 49.99, 'add_to_cart_url' =>'#'),
      array('path' =>'edge', 'name' =>'Edge', 'img' =>'/img/missing-image.png', 'price' => 69.99, 'add_to_cart_url' =>'#'),
    ;
  }
}
```

我们现在已经定义了五个全局 Twig 变量，它们将用于构建应用程序菜单。尽管现在变量被挂载到一个虚拟服务，它只返回一个虚拟数组，但我们已经有效地将菜单项解耦到其他即将构建的模块中。当我们稍后开始构建类别、客户和结帐模块时，我们将简单地编写一个服务覆盖，并用项目正确地数据填充菜单项数组。

{% hint style="info" %}
理想情况下，我们希望服务能够按照一定的接口返回数据，以确保不管是谁覆盖它或扩展它都是通过接口来实现的。由于我们试图让我们的应用保持在最低限度，我们将用简单的数组来进行。
{% endhint %}

现在我们可以回到我们的 `app/resources/views/base. html`。使用下面的代码替换前面代码中的 `< ! ! -- top-menu -- >` :

```markup
<div class="title-bar" data-responsive-toggle="appMenu" data-hide-for="medium">
  <button class="menu-icon" type="button" data-toggle></button>
  <div class="title-bar-title">Menu</div>
</div>

<div class="top-bar" id="appMenu">
  <div class="top-bar-left">
    {# category_menu is global twig var filled from service, and later overriden by another module service #}
    <ul class="menu">
      <li><a href="{{ path('homepage') }}">HOME</a></li>
        {% block category_menu %}
        {% for link in category_menu.getItems() %}
      <li><a href="{{ link.path }}">{{ link.label }}</li></a>
      {% endfor %}
      {% endblock %}
    </ul>
  </div>
  <div class="top-bar-right">
    <ul class="menu">
      {# customer_menu is global twig var filled from service, and later overriden by another module service #}
      {% block customer_menu %}
      {% for link in customer_menu.getItems() %}
      <li><a href="{{ link.path }}">{{ link.label }}</li></a>
      {% endfor %}
      {% endblock %}
      {# checkout_menu is global twig var filled from service, and later overriden by another module service #}
      {% block checkout_menu %}
      {% for link in checkout_menu.getItems() %}
      <li><a href="{{ link.path }}">{{ link.label }}</li></a>
      {% endfor %}
      {% endblock %}
    </ul>
  </div>
</div>
```

然后我们可以用以下代码替换 `< ! ! -- system-wide-messages -- >` :

```markup
<div class="row column">
  {% for flash_message in app.session.flashBag.get('alert') %}
  <div class="alert callout">
    {{ flash_message }}
  </div>
  {% endfor %}
  {% for flash_message in app.session.flashBag.get('warning') %}
  <div class="warning callout">
    {{ flash_message }}
  </div>
  {% endfor %}
  {% for flash_message in app.session.flashBag.get('success') %}
  <div class="success callout">
    {{ flash_message }}
  </div>
  {% endfor %}
</div>

```

我们将 `< ! ! ! -- per-page-body -- >` 替换为以下内容:

```markup
<div class="row column">
  {% block body %}{% endblock %}
</div>
```

我们用以下代码替换 `< ! ! -- footer -- >` :

```markup
<div class="row column">
  <ul class="menu">
    <li><a href="{{ path('about') }}">About Us</a></li>
    <li><a href="{{ path('customer_service') }}">Customer Service</a></li>
    <li><a href="{{ path('privacy_cookie') }}">Privacy and Cookie Policy</a></li>
    <li><a href="{{ path('orders_returns') }}">Orders and Returns</a></li>
    <li><a href="{{ path('contact') }}">Contact Us</a></li>
  </ul>
</div>
```

现在我们可以继续编辑 `src/AppBundle/Controller/DefaultController.php` 文件并添加以下代码:

```php
/**
 * @Route("/", name="homepage")
 */
public function indexAction(Request $request)
{
  return $this->render('AppBundle:default:index.html.twig');
}

/**
 * @Route("/about", name="about")
 */
public function aboutAction()
{
  return $this->render('AppBundle:default:about.html.twig');
}

/**
 * @Route("/customer-service", name="customer_service")
 */
public function customerServiceAction()
{
  return $this->render('AppBundle:default:customer-service.html.twig');
}

/**
 * @Route("/orders-and-returns", name="orders_returns")
 */
public function ordersAndReturnsAction()
{
  return $this->render('AppBundle:default:orders-returns.html.twig');
}

/**
 * @Route("/privacy-and-cookie-policy", name="privacy_cookie")
 */
public function privacyAndCookiePolicyAction()
{
  return $this->render('AppBundle:default:privacy-cookie.html.twig');
}

```

所有使用的模板文件\(`about.html.twig`、`customer-service.html.twig`、`orders-returns.html.twig`、`privacy-cookie.html.twig)`在`src/AppBundle/Resources/views/default`目录下，可以类似地定义如下:

```markup
{% extends 'base.html.twig' %}

{% block body %}
<div class="row">
  <h1>About Us</h1>
</div>
<div class="row">
  <p>Loremipsum dolor sit amet, consecteturadipiscingelit...</p>
</div>
{% endblock %}
```

在这里，我们只是用 `row` 类将 header 和 content 封装到 `div` 元素中，只是为了给它提供一些结构。结果应该与这里显示的页面类似:

![](../.gitbook/assets/image%20%28215%29.png)

联系我们页面需要一个不同的方法，因为它将包含一个表单。要构建表单，我们使用 Symfony 的 `Form`组件，在 `src/AppBundle/Controller/DefaultController.php` 文件中添加以下内容:

```php
/**
 * @Route("/contact", name="contact")
 */
public function contactAction(Request $request) {

  // Build a form, with validation rules in place
  $form = $this->createFormBuilder()
  ->add('name', TextType::class, array(
    'constraints' => new NotBlank()
  ))
  ->add('email', EmailType::class, array(
    'constraints' => new Email()
  ))
  ->add('message', TextareaType::class, array(
    'constraints' => new Length(array('min' => 3))
  ))
   ->add('save', SubmitType::class, array(
    'label' =>'Reach Out!',
    'attr' => array('class' =>'button'),
  ))
  ->getForm();

  // Check if this is a POST type request and if so, handle form
  if ($request->isMethod('POST')) {
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
      $this->addFlash(
        'success',
        'Your form has been submitted. Thank you.'
      );

      // todo: Send an email out...

      return $this->redirect($this->generateUrl('contact'));
    }
  }

  // Render "contact us" page
  return $this->render('AppBundle:default:contact.html.twig', array(
    'form' => $form->createView()
  ));
}
```

在这里，我们开始通过表单生成器构建表单。Add 方法同时接受字段定义和字段约束，可以基于这些约束进行验证。然后，我们添加了一个 HTTP POST 方法的检查，在这种情况下，我们向表单提供请求参数，并对其运行验证。

有了 `contactAction` 方法，我们仍然需要一个模板文件来实际呈现表单。为此，我们添加了 `src/appbundle/resources/views/default/contact.html`。Twig 文件，内容如下:

```markup
{% extends 'base.html.twig' %}

{% block body %}

<div class="row">
  <h1>Contact Us</h1>
</div>

<div class="row">
  {{ form_start(form) }}
  {{ form_widget(form) }}
  {{ form_end(form) }}
</div>
{% endblock %}
```

基于这几个标记，Twig 为我们处理表单呈现。页面如下所示:

![](../.gitbook/assets/image%20%28209%29.png)

我们已经差不多准备好所有的页面了。不过，有一点缺失了，那就是我们主页上的主题区域。与其他静态内容的页面不同，这个页面实际上是动态的，因为它列出了畅销书和在售产品。这些数据预计将来自其他模块，这些模块目前还不可用。尽管如此，这并不意味着我们不能为他们准备假的占位符。让我们继续编辑 `app/resources/views/default/index.html`。如下:

```markup
{% extends 'base.html.twig' %}
{% block body %}
<!--products_bestsellers -->
<!--products_onsale -->
{% endblock %}
```

现在我们需要将`<！--products_bestsellers-->`替换为以下内容:

```markup
{% if products_bestsellers %}
<h2 class="text-center">Best Sellers</h2>
<div class="row products_bestsellers text-center small-up-1 medium-up-3 large-up-5" data-equalizer data-equalize-by- row="true">
  {% for product in products_bestsellers.getItems() %}
  <div class="column product">
    <img src="{{ asset(product.img) }}" alt="missing image"/>
    <a href="{{ product.path }}">{{ product.name }}</a>
    <div>${{ product.price }}</div>
    <div><a class="small button"href="{{ product.add_to_cart_url }}">Add to Cart</a></div>
  </div>
  {% endfor %}
</div>
{% endif %}

```

现在我们需要用以下代替 `< ! ! -- products _ onsale -- >` :

```markup
{% if products_onsale %}
<h2 class="text-center">On Sale</h2>
<div class="row products_onsale text-center small-up-1 medium-up-3 large-up-5" data-equalizer data-equalize-by-row="true">
  {% for product in products_onsale.getItems() %}
  <div class="column product">
    <img src="{{ asset(product.img) }}" alt="missing image"/>
    <a href="{{ product.path }}">{{ product.name }}</a>
  <div>${{ product.price }}</div>
  <div><a class="small button"href="{{ product.add_to_cart_url }}">Add to Cart</a></div>
  </div>
  {% endfor %}
</div>
{% endif %}
```

{% hint style="info" %}
Http://dummyimage.com 图片库允许我们为我们的应用程序创建一个占位图片。
{% endhint %}

在这一点上，我们应该看到的主页如下所示:

![](../.gitbook/assets/image%20%28213%29.png)

### 配置整个应用程序的安全

作为我们整个应用程序安全的一部分，我们试图实现的是设置一些基本的保护，以防止未来的客户或任何其他用户能够访问和使用未来自动生成的CRUD控制器。我们通过修改`app/config/security.yml`文件来实现。`security.yml`文件有几个组件我们需要解决。防火墙、访问控制、提供者和编码器。如果我们观察之前测试应用中自动生成的CRUD，就会发现，我们需要保护以下内容不被客户访问:

* `GET|POST /new`
* `GET|POST /{id}/edit`
* `DELETE /{id}`

换句话说，所有URL中含有`/new`和`/edit`的内容，以及所有`DELETE`方法的内容，都需要被客户保护起来。考虑到这一点，我们将使用 Symfony 的安全特性来创建一个角色为`ROLE_ADMIN`的内存用户。然后我们将创建一个访问控制列表，只允许 `ROLE_ADMIN` 访问我们刚才提到的资源，并创建一个防火墙，当我们试图访问这些资源时，触发一个HTTP基本认证登录表单。

使用内存提供者意味着在我们的`security.yml`文件中对用户进行硬编码。在我们的应用中，我们将对管理员类型的用户进行硬编码。然而，实际的密码并不需要硬编码。假设我们将使用`1L6lllW9zXg0`作为密码，让我们跳转到控制台并键入以下命令:

```bash
php bin/console security:encode-password
```

这将产生如下输出:

![](../.gitbook/assets/image%20%28218%29.png)

现在我们可以通过添加内存提供者来编辑`security.yml`，并将生成的编码密码复制粘贴到其中，如图所示:

```yaml
security:
    providers:
        in_memory:
            memory:
                users:
                    john:
                        password: $2y$12$DFozWehwPkp14sVXr7.IbusW8ugvmZs9dQMExlggtyEa/TxZUStnO
                        roles: 'ROLE_ADMIN'
```

在这里，我们使用编码的`1L6lllW9zXg0`密码定义了 `ROLE_admin` 的用户 `john`。

一旦提供者就位，我们就可以继续向 `security.yml` 文件添加编码器。否则 Symfony 将不知道如何使用分配给 `john` 用户的当前密码:

```yaml
security:
    encoders:
        Symfony\Component\Security\Core\User\User:
            algorithm: bcrypt
            cost: 12

```

然后我们添加如下防火墙:

```yaml
security:
    firewalls:
        guard_new_edit:
            pattern: /(new)|(edit)
            methods: [GET, POST]
            anonymous: ~
            http_basic: ~
       guard_delete:
           pattern: /
           methods: [DELETE]
           anonymous: ~
           http_basic: ~
```

`guard_new_edit`和`guard_delete`这两个名字是我们自由赋予这两个应用防火墙的名字。`guard_new_edit`防火墙将拦截所有 GET 和 POST 请求，这些请求将指向任何URL中包含 `/new` 或 `/edit`字符串的路由。`guard_delete`防火墙将拦截任何 URL 上的任何 HTTP DELETE 方法。一旦这些防火墙启动，它们将显示一个 HTTP 基本认证表单，并且只有在用户登录后才允许访问。

然后我们添加访问控制列表如下:

```yaml
security:
    access_control:
      # protect any possible auto-generated CRUD actions from everyone's access
      - { path: /new, roles: ROLE_ADMIN }
      - { path: /edit, roles: ROLE_ADMIN }
      - { path: /, roles: ROLE_ADMIN, methods: [DELETE] }
```

有了这些条目，试图使用 `access_control` 下定义的任何模式访问任何 URL 的用户将会看到如下所示的浏览器登录:

![](../.gitbook/assets/image%20%28207%29.png)

唯一能够登录的用户是密码`1L6lllW9zXg0`的 `john`。经过身份验证后，用户可以访问所有 CRUD 链接。这对于我们的简单应用程序应该足够了。

## 单元测试

我们当前的模块除了控制器类和虚拟服务类之外没有其他特定的类。因此，我们不会在这里讨论单元测试。

## 功能测试

在开始编写功能测试之前，我们需要通过将 bundle `Tests` 目录添加到 `testsuite` 路径来编辑 `phpunit.xml.dist` 文件，如下所示:

```markup
<testsuites>
  <testsuite name="Project Test Suite">
    <-- ... other elements ... -->
      <directory>src/AppBundle/Tests</directory>
    <-- ... other elements ... -->
  </testsuite>
</testsuites>
```

我们的功能测试将只覆盖一个控制器，因为我们没有其他控制器。我们首先创建一个 `src/AppBundle/Tests/Controller/DefaultControllerTest.php` 文件，其内容如下:

```php
namespace AppBundle\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class DefaultControllerTest extends WebTestCase
{
//…
}
```

下一步是测试控制器的每一个动作。至少我们应该测试页面内容是否正确输出。

{% hint style="info" %}
要在我们的 IDE 中实现自动完成，我们可以从官方网站下载 `PHPUnitphar` https://phpunit.de。一旦下载完毕，我们就可以简单地将它添加到项目的根目录中，这样 IDE 就可以像 PHPStorm 一样获取它。这使得跟踪所有这些 `$This-> assert` 方法调用及其参数变得非常容易。
{% endhint %}

我们首先要测试的是我们的主页。为此，我们将以下代码添加到 `DefaultControllerTest` 类的主体中。

```php
{
  // @var \Symfony\Bundle\FrameworkBundle\Client
  $client = static::createClient();
  /** @var \Symfony\Component\DomCrawler\Crawler */
  $crawler = $client->request('GET', '/');

  // Check if homepage loads OK
  $this->assertEquals(200, $client->getResponse()->getStatusCode());

  // Check if top bar left menu is present
  $this->assertNotEmpty($crawler->filter('.top-bar-left li')->count());

  // Check if top bar right menu is present
  $this->assertNotEmpty($crawler->filter('.top-bar-right li')->count());

  // Check if footer is present
  $this->assertNotEmpty($crawler->filter('.footer li')->children()->count());
}
```

这里我们同时检查几项内容。我们检查页面加载是否正常，HTTP 200状态。然后我们抓取左边和右边的菜单，数一数它们的项目，看看是否有。如果各个检查都通过了，就认为`testHomepage`测试通过了。

我们进一步测试所有的静态页面，在 `DefaultControllerTest` 类中添加以下内容:

```php
public function testStaticPages()
{
  // @var \Symfony\Bundle\FrameworkBundle\Client
  $client = static::createClient();
  /** @var \Symfony\Component\DomCrawler\Crawler */

  // Test About Us page
  $crawler = $client->request('GET', '/about');
  $this->assertEquals(200, $client->getResponse()->getStatusCode());
  $this->assertContains('About Us', $crawler->filter('h1')->text());

  // Test Customer Service page
  $crawler = $client->request('GET', '/customer-service');
  $this->assertEquals(200, $client->getResponse()->getStatusCode());
  $this->assertContains('Customer Service', $crawler->filter('h1')->text());

  // Test Privacy and Cookie Policy page
  $crawler = $client->request('GET', '/privacy-and-cookie-policy');
  $this->assertEquals(200, $client->getResponse()->getStatusCode());
  $this->assertContains('Privacy and Cookie Policy', $crawler->filter('h1')->text());

  // Test Orders and Returns page
  $crawler = $client->request('GET', '/orders-and-returns');
  $this->assertEquals(200, $client->getResponse()->getStatusCode());
  $this->assertContains('Orders and Returns', $crawler->filter('h1')->text());

  // Test Contact Us page
  $crawler = $client->request('GET', '/contact');
  $this->assertEquals(200, $client->getResponse()->getStatusCode());
  $this->assertContains('Contact Us', $crawler->filter('h1')->text());
}
```

在这里，我们为所有的页面运行相同的 `tequals` 和 `assertContains` 函数。我们只是试图确认每个页面都加载了 HTTP 200，并且为页面标题返回了正确的值，也就是h1元素。

最后，我们通过在 `DefaultControllerTest` 类中添加以下内容来解决表单提交测试:

```php
public function testContactFormSubmit()
{
  // @var \Symfony\Bundle\FrameworkBundle\Client
  $client = static::createClient();
  /** @var \Symfony\Component\DomCrawler\Crawler */
  $crawler = $client->request('GET', '/contact');

  // Find a button labeled as "Reach Out!"
  $form = $crawler->selectButton('Reach Out!')->form();

  // Note this does not validate form, it merely tests against submission and response page
  $crawler = $client->submit($form);
  $this->assertEquals(200, $client->getResponse()->getStatusCode());
}
```

这里我们通过 **Reach Out！**提交按钮来抓取表单元素。一旦获取到表单，我们就在客户端上触发提交方法，将元素的实例传递给它。值得注意的是，这里没有测试实际的表单验证。即使如此，提交的表单应该会出现HTTP 200状态。

这些测试是决定性的。如果愿意，我们可以将它们编写得更加健壮，因为我们可以针对许多元素进行测试。

## 小结

在这一章中，我们建立了第一个模块，或者用 Symfony 的术语说是 bundle。这个模块本身并没有真正的松散耦合，因为它依赖于 app 目录下的一些东西，比如 `app/Resources/views/base.html.twig` 布局模板。当涉及到核心模块时，我们可以摆脱这种情况，因为它们只是我们为其他模块设置的一个基础。

接下来，在下一章中，我们将建立一个目录模块。这将是我们网店应用的基础。

