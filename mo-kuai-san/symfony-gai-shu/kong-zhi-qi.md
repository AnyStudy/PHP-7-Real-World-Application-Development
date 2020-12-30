# 控制器

控制器在网络应用中扮演着重要的角色，它处于任何应用输出的最前沿。它们是端点，是在每个URL后面执行的代码。用更技术化的方式，我们可以说控制器是任何接受HTTP请求并返回HTTP响应的可调用（一个函数、一个对象上的方法或一个闭包）。响应并不像HTML那样拘泥于单一的格式，它可以是XML、JSON、CSV、图像、重定向、错误等任何形式。

让我们来看看之前创建的`src/AppBundle/Controller/CustomerController.php`文件，更准确的说是它的`newAction`方法：

```php
/**
 * Creates a new Customer entity.
 *
 * @Route("/new", name="customer_new")
 * @Method({"GET", "POST"})
 */
public function newAction(Request $request)
{
  //...

  return $this->render('customer/new.html.twig', array(
    'customer' => $customer,
    'form' => $form->createView(),
  ));
}
```

如果我们忽略实际的数据检索部分（//...），在这个小例子中，有三件重要的事情需要注意：

* @Route：这是Symfony的注解方式，指定HTTP端点，也就是我们要用来访问的URL。第一个`"/new "`参数说明了实际的端点，第二个`name="customer_new"`参数设置了这个路由的名称，然后我们可以在模板等URL生成函数中使用这个名称作为别名。值得注意的是，这是在实际定义方法的`CustomerController`类上设置的`@Route("/customer")`注解的基础上，从而使得完整的URL是类似于`http://test.app/customer/new`。
* @Method：这是一个或多个HTTP方法的名称。这意味着只有当HTTP请求与之前定义的`@Route`相匹配，并且是`@Method`中定义的一个或多个HTTP方法类型时，`newAction`方法才会触发。
* $this-&gt;render：这将返回`Response`对象。`$this->render`调用`Symfony/Bundle/FrameworkBundle/Controller/Controller`类的`render`函数，该函数实例化新的`Response()`，设置其内容，并返回该对象的整个实例。

现在让我们看看控制器中的`editAction`方法，部分代码块如下所示：

```php
/**
 * Displays a form to edit an existing Customer entity.
 *
 * @Route("/{id}/edit", name="customer_edit")
 * @Method({"GET", "POST"})
 */
public function editAction(Request $request, Customer $customer)
{
  //...
}
```

这里，我们看到了一个接受单一ID的路由，在第一个`@Route`注解参数中标记为`{id}`。方法的主体（这里不包括在内），不包含任何获取id参数的直接引用。我们可以看到`editAction`函数接受两个参数，一个是`Request`，另一个是`Customer`。但该方法如何知道接受`Customer`对象呢？这就是Symfony的`@ParamConverter`注解发挥作用的地方。它调用转换器将请求参数转换为对象。

`@ParamConverter`注解的好处是，我们可以显式或隐式地使用它。也就是说，如果我们没有添加`@ParamConverter`注解，但在方法参数中添加了类型提示，Symfony就会尝试为我们加载对象。这正是我们上面例子中的情况，因为我们没有明确地输入`@ParamConverter`注解。

从术语上来说，控制器经常被交换为路由。然而，它们并不是一回事。

