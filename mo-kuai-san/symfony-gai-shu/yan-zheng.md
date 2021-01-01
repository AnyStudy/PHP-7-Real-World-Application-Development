# 验证

验证在现代应用中起着至关重要的作用。在讨论 web 应用程序时，我们可以说我们区分了两种主要的验证类型: 表单数据验证和持久数据验证。通过 web 表单从用户获取输入应该被验证，就像任何进入数据库的持久数据一样。

Symfony 在这方面表现优异，它提供了一个基于 JSR 303 Bean Validation（ http://beanvalidation.org/1.0/spec/） 的验证组件。如果我们回顾一下在框架根元素下的 `app/config/config.yml`，我们可以看到验证服务是默认打开的:

```yaml
framework:
  validation:{ enable_annotations: true }
```

我们可以通过简单的调用`$this->get('validator')`表达式从任何控制器类中访问验证服务，如下例所示：

```php
$customer = new Customer();

$validator = $this->get('validator');

$errors = $validator->validate($customer);

if (count($errors) > 0) {
  // Handle error state
}

// Handle valid state
```

上面这个例子的问题是验证不会返回任何错误。这样做的原因是我们没有在类上设置任何断言。控制台自动生成的 CRUD 并没有真正定义 `Customer` 类上的任何约束。我们可以通过尝试添加一个新客户并在 e-mail 字段中键入任何文本来确认这一点，我们可以看到电子邮件不会被验证。

让我们继续编辑 `src/AppBundle/Entity/Customer.php` 文件，在 `$Email` 属性中添加 `@assert\Email` 函数，如下所示:

```php
//…
use Symfony\Component\Validator\Constraints as Assert;
//…
class Customer
{
  //…
  /**
  * @var string
  *
  * @ORM\Column(name="email", type="string", length=255, unique=true)
  * @Assert\Email(
    *      checkMX = true,
    *      message = "Email '{{ value }}' is invalid.",
    * )
    */
  private $email;
  //…
}
```

关于断言约束的好处是它们接受参数就像接受函数一样。因此，我们可以根据我们的具体需要对个别约束进行微调。如果我们现在尝试跳过或添加错误的电子邮件地址，就会得到类似 **Email "john@gmail.test" is invalid** 的消息。

有许多限制可以利用，对于完整的列表，我们可以参考 [http://symfony.com/doc/current/book/validation.html](http://symfony.com/doc/current/book/validation.html) 网页。

约束可以应用于类属性或公共 getter 方法。虽然属性约束是最常见和最容易使用的，但 getter 方法约束允许我们指定更复杂的验证规则。

让我们看看 `src/AppBundle/Controller/CustomerController.php` 文件的 `newAction` 方法，如下所示:

```php
$customer = new Customer();
$form = $this->createForm('AppBundle\Form\CustomerType', $customer);
$form->handleRequest($request);

if ($form->isSubmitted() && $form->isValid()) {
// …
```

这里我们看到一个`CustomerType`表单的实例被绑定到`Customer`实例上。实际的 GET 或 POST 请求数据通过 `handleRequest`方法传递给表单的一个实例。现在表单能够理解实体验证约束，并通过 `isValid` 方法调用做出正确的响应。这意味着我们不需要自己使用验证服务进行手动验证，表单可以为我们做这件事。

我们将继续在验证特性方面进行扩展，逐步完成单独的 bundle。

