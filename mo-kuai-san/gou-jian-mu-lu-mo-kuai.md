# 构建目录模块

目录模块是每个网店应用程序的重要组成部分。在最基本的层面上，它负责种类和产品的管理和展示。它是后面的模块\(如 checkout\)的基础，这些模块向我们的 web shop 应用程序添加了实际的销售功能。

更强大的目录功能可能包括大量产品倒入、产品导出、多仓库库存管理、私人会员类别等。不过，这些都不在本章的讨论范围之内。

在本章中，我们将讨论以下主题:

* 要求
* 依赖
* 实施
* 单元测试
* 功能测试

## 要求

根据前面的模块化 商店应用的需求规范中定义的高级应用程序需求，我们的模块将实现多个实体和其他特定功能。

以下是所需模块实体清单:

* 类别
* 产品

类别实体包括以下属性及其数据类型:

* `id`: integer, auto-increment
* `title`: string
* `url_key`: string, unique
* `description`: text
* `image`: string

产品实体包括以下属性:

* `id`: integer, auto-increment
* `category_id`: integer, 引用类别表ID列的外键
* `title`: string
* `price`: decimal
* `sku`: string, unique
* `url_key`: string, unique
* `description`: text
* `qty`: integer
* `image`: string
* `onsale`: boolean

除了只添加这些实体和它们的 CRUD 页面外，我们还需要覆盖核心模块服务，负责构建分类菜单和在售商品。

## 依赖

该模块与其他模块之间没有牢固的依赖关系。Symfony 框架服务层使我们能够以这样的方式对模块进行编码，在大多数情况下，它们之间不需要依赖。虽然模块确实覆盖了核心模块中定义的服务，但模块本身并不依赖于它，因为如果缺少了覆盖的服务，也不会发生任何中断。

## 实施

我们首先创建一个名为 `Foggyline\CatalogBundle` 的新模块。我们在控制台的帮助下，通过运行以下命令来实现:

```bash
php bin/console generate:bundle --namespace=Foggyline/CatalogBundle
```

这个命令触发了一个交互过程，在这个过程中会问我们几个问题，如下面的截图所示:

![](../.gitbook/assets/image%20%28210%29.png)

一旦完成，将为我们生成以下结构:

![](../.gitbook/assets/image%20%28225%29.png)

如果我们现在看一下 `app/AppKernel.php` 文件，我们会看到 `registerBundles` 方法下面的行:

```php
new Foggyline\CatalogBundle\FoggylineCatalogBundle()
```

类似地，`app/config/routing.yml` 添加了以下路由定义:

```yaml
foggyline_catalog:
  resource: "@FoggylineCatalogBundle/Resources/config/routing.xml"
  prefix: /
```

这里我们需要将 `prefix: /` 更改为 `prefix: /catalog/`，这样我们就不会与核心模块路由发生冲突。如果不改变`prefix`，那么会覆盖 `AppBundle` 路由，从而导致输出 `helloworld！`来自 `src/foggyline/catalogbundle/resources/views/default/index.html`。我们希望保持事物的美好和分离。这意味着模块不为自己定义根路由。

### 创建实体

让我们继续创建一个 `Category` 实体，我们使用控制台来实现，如下所示:

```bash
php bin/console generate:doctrine:entity
```

![](../.gitbook/assets/image%20%28220%29.png)

这将在 `src/Foggyline/CatalogBundle/` 目录中创建 `Entity/Category.php` 和 `Repository/CategoryRepository.php` 文件。在这之后，我们需要更新数据库，所以它拉入了 `Category` 实体，如下面的命令行实例所示:

```bash
php bin/console doctrine:schema:update --force
```

这样就得到了一个和下面截图类似的屏幕:

![](../.gitbook/assets/image%20%28215%29.png)

有了实体，我们就可以生成它的 CRUD 了:

```bash
php bin/console generate:doctrine:crud
```

这个结果与交互式输出如下所示:

![](../.gitbook/assets/image%20%28209%29.png)

这将创建 `src/Foggyline/CatalogBundle/Controller/CategoryController.php`。还为我们的`app/config/routing.yml`文件添加了一个条目，如下所示。

```yaml
foggyline_catalog_category:
  resource: "@FoggylineCatalogBundle/Controller/CategoryController.php"
  type:     annotation
```

此外，视图文件被创建在 `app/Resources/views/category/` 目录下，这不是我们所期望的。我们希望它们在我们的模块 `src/Foggyline/CatalogBundle/Resources/views/Default/category/` 目录下，所以我们需要把它们复制过来。此外，我们需要修改 `CategoryController` 中的所有 `$this->render` 调用，将 `FoggylineCatalogBundle:default: string` 附加到每个模板路径。

接下来，我们继续使用前面讨论过的交互式生成器创建 `Product` 实体:

```bash
php bin/console generate:doctrine:entity
```

我们遵循交互式生成器，遵循以下属性的最小值: `title、 price、 sku、 url_key、 description、 qty、 category 和 image`。除了十进制和整数类型的 `price` 和 `qty` 之外，所有其他属性都是 `string` 类型的。此外，`sku` 和 `url_key` 被标记为唯一的。这将在 `src/Foggyline/CatalogBundle/`目录中创建`Entity/Product.php` 和 `Repository/ProductRepository.php` 文件。

与我们对 `Category` 视图模板所做的类似，我们需要对 `Product` 视图模板进行修改。也就是说，将它们从 `app/Resources/views/product/` 目录下复制到`src/Foggyline/CatalogBundle/Resources/views/Default/product/`，然后更新 `ProductController` 中的所有 `$this->render`调用，将`FoggylineCatalogBundle:default:string` 附加到每个模板路径。

在这一点上，我们不会急于更新模式，因为我们要在代码中添加适当的关系。每一个产品都应该能够与一个`Category` 实体建立关系。为了达到这个目的，我们需要在 `src/Foggyline/CatalogBundle/Entity/` 目录下编辑 `Category.php` 和 `Product.php`，如下所示:

```php
// src/Foggyline/CatalogBundle/Entity/Category.php

/**
 * @ORM\OneToMany(targetEntity="Product", mappedBy="category")
 */
private $products;

public function __construct()
{
  $this->products = new \Doctrine\Common\Collections\ArrayCollection();
}

// src/Foggyline/CatalogBundle/Entity/Product.php

/**
 * @ORM\ManyToOne(targetEntity="Category", inversedBy="products")
 * @ORM\JoinColumn(name="category_id", referencedColumnName="id")
 */
private $category;
```

我们还需要编辑`Category.php`文件，在其中添加`__toString方`法实现，如下所示:

```php
public function __toString()
{
    return $this->getTitle();
}
```

我们这样做的原因是，以后我们的产品编辑表单就会知道在`Category`选择下要列出哪些标签，否则系统会抛出以下错误。

```bash
Catchable Fatal Error: Object of class Foggyline\CatalogBundle\Entity\Category could not be converted to string
```

有了上述变化，我们现在可以运行模式更新，如下:

```bash
php bin/console doctrine:schema:update --force
```

如果我们现在看一下我们的数据库，产品表的CREATE命令语法如下:

```sql
CREATE TABLE `product` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `category_id` int(11) DEFAULT NULL,
  `title` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `price` decimal(10,2) NOT NULL,
  `sku` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `url_key` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `description` longtext COLLATE utf8_unicode_ci,
  `qty` int(11) NOT NULL,
  `image` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UNIQ_D34A04ADF9038C4` (`sku`),
  UNIQUE KEY `UNIQ_D34A04ADDFAB7B3B` (`url_key`),
  KEY `IDX_D34A04AD12469DE2` (`category_id`),
  CONSTRAINT `FK_D34A04AD12469DE2` FOREIGN KEY (`category_id`) REFERENCES `category` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```

我们可以看到，根据提供给我们的交互式实体生成器的条目，定义了两个唯一键和一个外键约束。现在我们已经准备好为我们的`Product`实体生成 CRUD。为此，我们运行 `generate:doctrine:crud` 命令，并按照这里所示的交互式生成器进行操作:

![](../.gitbook/assets/image%20%28212%29.png)

### 管理图像上传

此时，如果我们访问 `/category/new/` 或 `/product/new/` URL，图片字段只是一个简单的输入文本字段，而不是我们想要的实际图片上传。要想把它变成图片上传字段，我们需要编辑`Category.php`和`Product.php`的`$image`属性，如下所示:

```php
//…
use Symfony\Component\Validator\Constraints as Assert;
//…
class [Category|Product]
{
  //…
  /**
  * @var string
  *
  * @ORM\Column(name="image", type="string", length=255, nullable=true)
  * @Assert\File(mimeTypes={ "image/png", "image/jpeg" }, mimeTypesMessage="Please upload the PNG or JPEG image file.")
  */
  private $image;
  //…
}

```

只要我们这样做，输入字段就会变成文件上传字段，如图所示:

![](../.gitbook/assets/image%20%28208%29.png)

接下来，我们将继续在表单中实现上传功能。

我们首先定义处理实际上传的服务。通过在 `src/Foggyline/CatalogBundle/Resources/config/services.xml` 文件的 `services` 元素下添加以下条目来定义服务:

```markup
<service id="foggyline_catalog.image_uploader" class="Foggyline\CatalogBundle\Service\ImageUploader">
  <argument>%foggyline_catalog_images_directory%</argument>
</service>
```

`%foggyline_catalog_images_directory%`参数值是我们即将定义的参数名称。

然后我们创建 `src/Foggyline/CatalogBundle/service/ImageUploader.php` 文件，内容如下:

```php
namespace Foggyline\CatalogBundle\Service;

use Symfony\Component\HttpFoundation\File\UploadedFile;

class ImageUploader
{
  private $targetDir;

  public function __construct($targetDir)
  {
    $this->targetDir = $targetDir;
  }

  public function upload(UploadedFile $file)
  {
    $fileName = md5(uniqid()) . '.' . $file->guessExtension();
    $file->move($this->targetDir, $fileName);
    return $fileName;
  }
}
```

然后我们在 `src/Foggyline/CatalogBundle/Resources/config` 目录下创建自己的 `parameters.yml` 文件，内容如下:

```yaml
parameters:
  foggyline_catalog_images_directory: "%kernel.root_dir%/../web/uploads/foggyline_catalog_images"
```

这是我们的服务期望找到的参数。如果需要的话，它可以很容易地在`app/config/parameters.yml`下被覆盖。

为了让我们的bundle能找到`parameters.yml`文件，我们仍然需要在`src/Foggyline/CatalogBundle/DependencyInjection/`目录下编辑`FoggylineCatalogExtension.php`文件，在`load` 方法的最后添加以下`loader`:

```php
$loader = new Loader\YamlFileLoader($container, new FileLocator(__DIR__.'/../Resources/config'));
$loader->load('parameters.yml');
```

在这一点上，我们的 Symfony 模块能够读取它的 `parameters.yml`，从而使定义的服务能够为它的参数获取合适的值。剩下的就是调整新建表单和编辑表单的代码，为它们附加上传功能。由于两个表单都是一样的，下面是一个`Category`的例子，同样也适用于`Product`表单:

```php
public function newAction(Request $request) {
  // ...

  if ($form->isSubmitted() && $form->isValid()) {
    /* @var $image \Symfony\Component\HttpFoundation\File\UploadedFile */
    if ($image = $category->getImage()) {
      $name = $this->get('foggyline_catalog.image_uploader')->upload($image);
      $category->setImage($name);
    }

    $em = $this->getDoctrine()->getManager();
    // ...
  }

  // ...
}

public function editAction(Request $request, Category $category) {
  $existingImage = $category->getImage();
  if ($existingImage) {
    $category->setImage(
      new File($this->getParameter('foggyline_catalog_images_directory') . '/' . $existingImage)
    );
  }

  $deleteForm = $this->createDeleteForm($category);
  // ...

  if ($editForm->isSubmitted() && $editForm->isValid()) {
    /* @var $image \Symfony\Component\HttpFoundation\File\UploadedFile */
    if ($image = $category->getImage()) {
      $name = $this->get('foggyline_catalog.image_uploader')->upload($image);
      $category->setImage($name);
    } elseif ($existingImage) {
      $category->setImage($existingImage);
    }

    $em = $this->getDoctrine()->getManager();
    // ...
  }

  // ...
}
```

现在，新建表单和编辑表单都应该能够处理文件上传。

### 覆盖核心模块服务

现在我们继续来解决分类菜单和发售商品的问题。早在构建核心模块的时候，我们在`app/config/config.yml`文件的`twig:global`部分下定义了全局变量。这些变量是指向`app/config/services.yml`文件中定义的服务。为了让我们改变分类菜单和在售商品的内容，我们需要覆盖这些服务。

我们首先在 `src/Foggyline/CatalogBundle/Resources/config/services.xml` 文件下添加以下两个服务定义:

```markup
<service id="foggyline_catalog.category_menu" class="Foggyline\CatalogBundle\Service\Menu\Category">
  <argument type="service" id="doctrine.orm.entity_manager" />
  <argument type="service" id="router" />
</service>

<service id="foggyline_catalog.onsale" class="Foggyline\CatalogBundle\Service\Menu\OnSale">
  <argument type="service" id="doctrine.orm.entity_manager" />
  <argument type="service" id="router" />
</service>
```

这两个服务都接受 Doctrine ORM 实体管理器和路由器服务参数，因为我们需要在内部使用这些参数。

然后，我们在 `src/Foggyline/CatalogBundle/Service/Menu/` 目录中创建实际的 `Category` 和 `OnSale` 服务类，如下所示:

```php
//Category.php

namespace Foggyline\CatalogBundle\Service\Menu;

class Category
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
    $categories = array();
    $_categories = $this->em->getRepository('FoggylineCatalogBundle:Category')->findAll();

    foreach ($_categories as $_category) {
      /* @var $_category \Foggyline\CatalogBundle\Entity\Category */
      $categories[] = array(
        'path' => $this->router->generate('category_show', array('id' => $_category->getId())),
        'label' => $_category->getTitle(),
      );
    }

    return $categories;
  }
}
 //OnSale.php

namespace Foggyline\CatalogBundle\Service\Menu;

class OnSale
{
  private $em;
  private $router;

  public function __construct(\Doctrine\ORM\EntityManager $entityManager, $router)
  {
    $this->em = $entityManager;
    $this->router = $router;
  }

  public function getItems()
  {
    $products = array();
    $_products = $this->em->getRepository('FoggylineCatalogBundle:Product')->findBy(
        array('onsale' => true),
        null,
        5
    );

    foreach ($_products as $_product) {
      /* @var $_product \Foggyline\CatalogBundle\Entity\Product */
      $products[] = array(
        'path' => $this->router->generate('product_show', array('id' => $_product->getId())),
        'name' => $_product->getTitle(),
        'image' => $_product->getImage(),
        'price' => $_product->getPrice(),
        'id' => $_product->getId(),
      );
    }

    return $products;
  }
}
```

仅仅这样是不会触发核心模块服务的覆盖的。在`src/Foggyline/CatalogBundle/DependencyInjection/Compiler/`目录内，我们需要创建一个`OverrideServiceCompilerPass`类，实现`CompilerPassInterface`。在它的`process`方法中，我们就可以更改服务的定义，如下图:

```php
namespace Foggyline\CatalogBundle\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class OverrideServiceCompilerPass implements CompilerPassInterface
{
  public function process(ContainerBuilder $container)
  {
    // Override the core module 'category_menu' service
    $container->removeDefinition('category_menu');
    $container->setDefinition('category_menu', $container->getDefinition('foggyline_catalog.category_menu'));

    // Override the core module 'onsale' service
    $container->removeDefinition('onsale');
    $container->setDefinition('onsale', $container->getDefinition('foggyline_catalog.onsale'));
  }
}
```

最后，我们需要编辑`src/Foggyline/CatalogBundle/FoggylineCatalogBundle.php`文件的构建方法，使编译通过，如图所示：

```php
public function build(ContainerBuilder $container)
{
  parent::build($container);
  $container->addCompilerPass(new \Foggyline\CatalogBundle\DependencyInjection\Compiler\OverrideServiceCompilerPass());
}
```

现在，我们的`Category`和`OnSale`服务已经覆盖核心模块中定义的服务，从而为主页的Category菜单和On Sale部分提供正确的值。

### 设置分类页面

自动生成的CRUD为我们做了一个`Category`页面，布局如下:

![](../.gitbook/assets/image%20%28223%29.png)

这与我们在模块化网店应用的需求规范中定义的`Category`页面有很大不同。因此，我们需要修改分类展示页面，修改`src/Foggyline/CatalogBundle/Resources/views/default/category/`目录下的`show.html.twig`文件。我们用以下代码替换`body block`的全部内容:

```markup
<div class="row">
  <div class="small-12 large-12 columns text-center">
    <h1>{{ category.title }}</h1>
    <p>{{ category.description }}</p>
  </div>
</div>

<div class="row">
  <img src="{{ asset('uploads/foggyline_catalog_images/' ~ category.image) }}"/>
</div>

{% set products = category.getProducts() %}
{% if products %}
<div class="row products_onsale text-center small-up-1 medium-up-3 large-up-5" data-equalizer data-equalize-by-row="true">
{% for product in products %}
<div class="column product">
  <img src="{{ asset('uploads/foggyline_catalog_images/' ~ product.image) }}" 
    alt="missing image"/>
  <a href="{{ path('product_show', {'id': product.id}) }}">{{ product.title }}</a>

  <div>${{ product.price }}</div>
  <div><a class="small button" href="{{ path('product_show', {'id': product.id}) }}">View</a></div>
  </div>
  {% endfor %}
</div>
{% else %}
<div class="row">
  <p>There are no products assigned to this category.</p>
</div>
{% endif %}

{% if is_granted('ROLE_ADMIN') %}
<ul>
  <li>
    <a href="{{ path('category_edit', { 'id': category.id }) }}">Edit</a>
  </li>
  <li>
    {{ form_start(delete_form) }}
    <input type="submit" value="Delete">
    form_end(delete_form) }}
  </li>
</ul>
{% endif %}
```

现在正文分为三个方面。首先，我们要解决类别标题和描述的输出。然后，我们正在获取并循环处理分配给类别的产品列表，呈现每个单独的产品。最后，我们使用`is_granted Twig`扩展来检查当前用户角色是否为`ROLE_ADMIN`，在这种情况下，我们将显示类别的编辑和删除链接。

### 设置产品页面

自动生成的CRUD为我们做了一个产品页面，布局如下:

![](../.gitbook/assets/image%20%28218%29.png)

这与我们在模块化网店应用的需求规范中定义的产品页面不同。为了解决这个问题，我们需要修改产品展示页面，修改`src/Foggyline/CatalogBundle/Resources/views/Default/product/`目录下的`show.html.twig`文件。我们用下面的代码替换整个`body block`的内容:

```markup
<div class="row">
  <div class="small-12 large-6 columns">
    <img class="thumbnail" src="{{ asset('uploads/foggyline_catalog_images/' ~ product.image) }}"/>
  </div>
  <div class="small-12 large-6 columns">
    <h1>{{ product.title }}</h1>
    <div>SKU: {{ product.sku }}</div>
    {% if product.qty %}
    <div>IN STOCK</div>
    {% else %}
    <div>OUT OF STOCK</div>
    {% endif %}
    <div>$ {{ product.price }}</div>
    <form action="{{ add_to_cart_url.getAddToCartUrl
      (product.id) }}" method="get">
      <div class="input-group">
        <span class="input-group-label">Qty</span>
        <input class="input-group-field" type="number">
        <div class="input-group-button">
          <input type="submit" class="button" value="Add to Cart">
        </div>
      </div>
    </form>
  </div>
</div>

<div class="row">
  <p>{{ product.description }}</p>
</div>

{% if is_granted('ROLE_ADMIN') %}
<ul>
  <li>
    <a href="{{ path('product_edit', { 'id': product.id }) }}">Edit</a>
  </li>
  <li>
    {{ form_start(delete_form) }}
    <input type="submit" value="Delete">
    {{ form_end(delete_form) }}
  </li>
</ul>
{% endif %}
```

现在正文主要分为两个方面。首先，我们要解决的是产品图片、标题、库存状态和添加到购物车的输出。添加到购物车表单使用`add_to_cart_url`服务来提供正确的链接。这个服务是在核心模块下定义的，此时，只提供一个虚链接。稍后，当我们到了结账模块时，我们将为这个服务实现一个覆盖，并注入正确的`add to cart`链接。然后，我们输出描述部分。最后，我们使用`is_granted Twig`扩展，就像我们在`Category`例子上做的那样，确定用户是否可以访问产品的`Edit`和`Delete`链接。

## 单元测试

我们现在有几个与控制器无关的类文件，这意味着我们可以针对它们运行单元测试。不过，作为本书的一部分，我们还是不会去追求完整的代码覆盖，而是专注于一些小事情，比如在我们的测试类中使用容器。

我们首先在`phpunit.xml.dist`文件的`testuites`元素下添加以下一行:

```markup
<directory>src/Foggyline/CatalogBundle/Tests</directory>
```

设置好之后，从我们商店的根目录运行 `phpunit` 命令应该可以获取我们在 `src/Foggyline/CatalogBundle/Tests/` 目录下定义的所有测试。

现在，让我们继续为类别服务菜单创建一个测试。 为此，我们创建了一个 `src/Foggyline/CatalogBundle/Tests/Service/Menu/CategoryTest.php`文件，其内容如下：

```php
namespace Foggyline\CatalogBundle\Tests\Service\Menu;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Foggyline\CatalogBundle\Service\Menu\Category;

class CategoryTest extends KernelTestCase
{
  private $container;
  private $em;
  private $router;

  public function setUp()
  {
    static::bootKernel();
    $this->container = static::$kernel->getContainer();
    $this->em = $this->container->get('doctrine.orm.entity_manager');
    $this->router = $this->container->get('router');
  }

  public function testGetItems()
  {
    $service = new Category($this->em, $this->router);
    $this->assertNotEmpty($service->getItems());
  }

  protected function tearDown()
  {
    $this->em->close();
    unset($this->em, $this->router);
  }
}
```

前面的例子展示了 `setUp` 和 `tearDown` 方法调用的用法，它们的行为类似于 PHP 的 `__construct` **和 `__destruct`** 方法。我们使用 `setUp` 方法来设置实体管理器和路由器服务，我们可以在类的其余部分中使用。`tearDown`方法只是一个清理方法。现在如果我们运行`phpunit`命令，我们应该会看到我们的测试和其他测试一起被接收和执行。

我们甚至可以通过执行`phpunit`命令来指定这个类的完整路径，如图所示:

```bash
phpunit src/Foggyline/CatalogBundle/Tests/Service/Menu/CategoryTest.php
```

与我们对`CategoryTest`所做的类似，我们可以继续创建`OnSaleTest`。 两者之间的唯一区别是类名。

## 功能测试

自动生成 CRUD工具最大的好处是，它甚至可以为我们生成功能测试。更具体地说，在本例中，它在`src/Foggyline/CatalogBundle/Tests/Controller/`目录下生成了`CategoryControllerTest.php`和`ProductControllerTest.php`文件。

{% hint style="info" %}
自动生成的功能测试在类体中有一个被注释掉的方法. 这将在 `phpunit` 运行时抛出一个错误。我们至少需要在其中定义一个虚拟测试方法来允许`phpunit`忽略它们.
{% endhint %}

如果我们查看这两个文件，我们可以看到它们都定义了一个`testCompleteScenario`方法，这个方法被完全注释掉了。让我们继续修改`CategoryControllerTest.php`的内容，如下所示:

```php
// Create a new client to browse the application
$client = static::createClient(
  array(), array(
    'PHP_AUTH_USER' => 'john',
    'PHP_AUTH_PW' => '1L6lllW9zXg0',
  )
);

// Create a new entry in the database
$crawler = $client->request('GET', '/category/');
$this->assertEquals(200, $client->getResponse()->getStatusCode(), "Unexpected HTTP status code for GET /product/");
$crawler = $client->click($crawler->selectLink('Create a new entry')->link());

// Fill in the form and submit it
$form = $crawler->selectButton('Create')->form(array(
  'category[title]' => 'Test',
  'category[urlKey]' => 'Test urlKey',
  'category[description]' => 'Test description',
));

$client->submit($form);
$crawler = $client->followRedirect();

// Check data in the show view
$this->assertGreaterThan(0, $crawler->filter('h1:contains("Test")')->count(), 'Missing element h1:contains("Test")');

// Edit the entity
$crawler = $client->click($crawler->selectLink('Edit')->link());

$form = $crawler->selectButton('Edit')->form(array(
  'category[title]' => 'Foo',
  'category[urlKey]' => 'Foo urlKey',
  'category[description]' => 'Foo description',
));

$client->submit($form);
$crawler = $client->followRedirect();

// Check the element contains an attribute with value equals "Foo"
$this->assertGreaterThan(0, $crawler->filter('[value="Foo"]')->count(), 'Missing element [value="Foo"]');

// Delete the entity
$client->submit($crawler->selectButton('Delete')->form());
$crawler = $client->followRedirect();

// Check the entity has been delete on the list
$this->assertNotRegExp('/Foo title/', $client->getResponse()->getContent());
```

我们首先将 `PHP_AUTH_USER` 和 `PHP_AUTH_PW` 设置为 `createClient` 方法的参数。这是因为我们的`/new`和`/edit`路由受到核心模块的安全保护。这些设置允许我们沿着请求传递基本的 HTTP 认证。然后，我们测试了是否可以访问分类列表页面，以及是否可以点击其创建新条目链接。此外，我们还测试了创建和编辑表单，以及它们的结果。

剩下的就是重复刚才 `CategoryControllerTest.php` 和 `ProductControllerTest.php` 的方法。我们只需要修改 `ProductControllerTest` 类文件中的一些标签，使其与产品路线和预期结果相匹配。

现在运行 `phpunit`命令应该可以成功执行我们的测试。

## 小结

在本章中，我们已经建立了一个微型的，但功能性的目录模块。它允许我们创建、编辑和删除类别和产品。通过在自动生成的 CRUD 之上添加几行自定义代码，我们能够实现类别和产品的图片上传功能。我们还看到了如何覆盖核心模块服务，只需删除现有的服务定义并提供一个新的服务。在测试方面，我们看到了如何沿着我们的请求传递认证，以测试受保护的路由。

接下来，在下一章，我们将建立一个客户模块。

