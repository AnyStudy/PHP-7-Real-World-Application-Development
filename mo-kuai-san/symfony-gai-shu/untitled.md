# 路由

简而言之，路由就是将控制器与浏览器中输入的URL链接起来。现代网络应用需要漂亮的URL。这意味着从`/index.php?product_id=23`这样的URL转为`/catalog/product/t-shirt`这样的URL。这就是路由的作用。

Symfony 有一个强大的路由机制，让我们可以做到以下几点：

* 创建映射到控制器的复杂路由 
* 在模板内生成URL 
* 在控制器内生成URL 
* 从不同地点加载路由资源

在 Symfony 中，路由的工作方式是所有的请求都通过`app.php`。然后，Symfony 核心要求路由器检查该请求。路由器会将传入的URL与特定的路由进行匹配，并返回有关该路由的信息。这些信息，包括应该执行的控制器。最后，Symfony 内核执行控制器，控制器返回一个响应对象。

所有的应用路由都是从一个路由配置文件加载的，通常是`app/config/routing.yml`文件，如我们的测试应用所示：

```yaml
app:
  resource: "@AppBundle/Controller/"
  type:     annotation
```

App 只是众多可能的条目之一。它的资源值指向 `AppBundle` 控制器目录，类型设置为`annotation`，这意味着将读取类`annotations`来指定精确的路由。

我们可以用几种变化来定义一个路由。其中的一个变化如下块所示：

```php
// Basic Route Configuration
/**
 * @Route("/")
 */
public function homeAction()
{
  // ...
}

// Routing with Placeholders
/**
 * @Route("/catalog/product/{sku}")
 */
public function showAction($sku)
{
  // ...
}

// >>Required<< and Optional Placeholders
/**
 * @Route("/catalog/product/{id}")
 */
public function indexAction($id)
{
  // ...
}
// Required and >>Optional<< Placeholders
/**
 * @Route("/catalog/product/{id}", defaults={"id" = 1})
 */
public function indexAction($id)
{
  // ...
}
```

前面的例子展示了几种定义路由的方法。有趣的是，有必要参数和可选参数的情况。如果我们考虑一下，从最新的例子中去掉 ID，就会和前面的例子用 sku 匹配。Symfony 路由器总是会选择第一个找到的匹配路由。我们可以通过在`@Route`注解上添加正则表达式要求来解决这个问题，如下所示：

```php
@Route(
  "/catalog/product/{id}",
  defaults={"id": 1},
  requirements={"id": "\d+"}
)
```

关于控制器和路由，还有更多的内容要讲，一旦我们开始构建我们的应用程序，我们就会看到。

