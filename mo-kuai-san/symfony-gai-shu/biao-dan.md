# 表单

注册、签到、加入购物车、结账，这些等等都是在网店应用中利用HTML表单等进行的操作。构建表单是开发人员最常见的任务之一。一个经常需要花时间才能做好的工作。

Symfony 有一个表单组件，通过它我们可以用OO的方式来构建HTML表单。该组件本身也是一个独立的库，可以独立于 Symfony 使用。

我们来看看`src/AppBundle/Entity/Customer.php`文件的内容，我们的 `Customer`实体类是我们通过控制台定义时自动生成的。

```php
class Customer {
  private $id;
  private $firstname;
  private $lastname;
  private $email;

  public function getId() {
    return $this->id;
  }

  public function setFirstname($firstname) {
    $this->firstname = $firstname;
    return $this;
  }

  public function getFirstname() {
    return $this->firstname;
  }

  public function setLastname($lastname) {
    $this->lastname = $lastname;
    return $this;
  }

  public function getLastname() {
    return $this->lastname;
  }

  public function setEmail($email) {
    $this->email = $email;
    return $this;
  }

  public function getEmail() {
    return $this->email;
  }
}
```

这里我们有一个普通的PHP类，它没有扩展任何东西，也没有以任何其他方式与 Symfony 链接。它代表了一个单一的客户实体，并为其设置和获取数据。有了这个实体类，我们想渲染一个表单，以获取我们的类所使用的所有相关数据。这就是表单组件的作用。

当我们之前通过控制台使用CRUD生成器时，它在`src/AppBundle/Form/CustomerType.php`文件中为我们的`Customer`实体创建了`Form`类，内容如下：

```php
namespace AppBundle\Form;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class CustomerType extends AbstractType
{
  public function buildForm(FormBuilderInterface $builder, array $options) {
    $builder
    ->add('firstname')
    ->add('lastname')
    ->add('email')
    ;
  }

  public function configureOptions(OptionsResolver $resolver) {
    $resolver->setDefaults(array(
      'data_class' =>'AppBundle\Entity\Customer'
    ));
  }
}
```

我们可以看到表单组件背后的简单性归结为以下几点：

* 扩展表格类型。我们从`Symfony\Component\Form/AbstractType`类中扩展出来。 
* 实现 buildForm 方法。这就是我们添加实际字段的地方，我们希望在表格上显示的字段。 
* 实现 configureOptions。这至少指定了指向我们客户实体的`data_class`配置。

表单生成器对象在这里做的是繁重的工作。它不需要太多的时间就可以创建一个表单。有了表单类之后，让我们来看看负责给模板输入表单的控制器动作。在本例中，我们将关注`src/AppBundle/Controller/CustomerController.php`文件中的`newAction`，内容如下所示：

```php
$customer = new Customer();
$form = $this->createForm('AppBundle\Form\CustomerType', $customer);
$form->handleRequest($request);

if ($form->isSubmitted() && $form->isValid()) {
  $em = $this->getDoctrine()->getManager();
  $em->persist($customer);
  $em->flush();

  return $this->redirectToRoute('customer_show', array('id' =>$customer->getId()));
}

return $this->render('customer/new.html.twig', array(
  'customer' => $customer,
  'form' => $form->createView(),
));
```

前面的代码首先实例化了`Customer`实体类。`$this->createForm(...)`实际上是调用`$this->container->get('form.factory')->create(...)`，传递给它我们的表单类名称和`customer`对象的实例，然后我们有`isSubmitted`和`isValid`检查，看看这是否是GET或有效的POST请求。根据这个检查，代码要么返回客户列表，要么将表单和客户实例设置为使用模板`customer/new.html.twig`。关于实际的验证，我们会在后面再讲。

最后，让我们来看看`app/Resources/views/customer/new.html.twig`文件中的实际模板：

```markup
{% extends 'base.html.twig' %}

{% block body %}
<h1>Customer creation</h1>

{{ form_start(form) }}
{{ form_widget(form) }}
<input type="submit" value="Create" />
{{ form_end(form) }}

<ul>
  <li>
    <a href="{{ path('customer_index') }}">Back to the list</a>
  </li>
</ul>
{% endblock %}
```

在这里我们看到了`extends`和`block`标签，以及一些相关的表单功能。Symfony 在`Twig`中增加了以下几个表单渲染功能：

* `form(view, variables)`
* `form_start(view, variables)`
* `form_end(view, variables)`
* `form_label(view, label, variables)`
* `form_errors(view)`
* `form_widget(view, variables)`
* `form_row(view, variables)`
* `form_rest(view, variables)`

我们的大多数申请表格都会像这样自动生成，所以我们能够得到一个功能完备的CRUD，而不需要太深入地了解表格的其他功能。

