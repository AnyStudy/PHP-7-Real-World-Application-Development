# 实现表单工厂

表单工厂的目的是从单个配置数组中生成一个可用的表单对象。表单对象应该能够检索它所包含的单个元素，以便生成输出。

## 如何做...

1.首先，让我们创建一个名为`Application\Form\Factory`的类来包含工厂代码。它将只有一个属性，`$elements`和一个`getter`。

```php
namespace Application\Form;

class Factory
{
  protected $elements;
  public function getElements()
  {
    return $this->elements;
  }
  // 剩余代码
}
```

2. 在我们定义主要的表单生成方法之前，重要的是要考虑我们计划接收什么配置格式，以及表单生成到底会产生什么。在这个例子中，我们将假设生成将产生一个`Factory`实例，有一个`$elements`属性。这个属性将是一个`Application\Form\Generic`或`Application\Form\Element`类的数组。

3. 我们现在准备好处理`generate()`方法。这将循环浏览配置数组，创建适当的`Application\Form\Generic`或`Application\Form\Element\*`对象，这些对象将被存储在`$elements`数组中。新方法将接受配置数组作为参数。将这个方法定义为静态的方法是很方便的，这样我们就可以根据需要使用不同的配置块来生成任意多的实例。

4. 我们创建一个`Application\Form\Factory`的实例，然后我们开始在配置数组中循环。

```php
public static function generate(array $config)
{
  $form = new self();
  foreach ($config as $key => $p) {
```

5. 接下来，我们检查`Application\Form\Generic`类的构造函数中是否有可选的参数。

```php
  $p['errors'] = $p['errors'] ?? array();
  $p['wrappers'] = $p['wrappers'] ?? array();
  $p['attributes'] = $p['attributes'] ?? array();
```

6. 现在所有的构造函数参数都到位了，我们可以创建表单元素实例，然后将其存储在`$elements`中。

```php
  $form->elements[$key] = new $p['class']
  (
    $key, 
    $p['type'],
    $p['label'],
    $p['wrappers'],
    $p['attributes'],
    $p['errors']
  );
```

7. 接下来，我们将注意力转移到选项上。如果设置了选项参数，我们使用`list()`将数组值提取到变量中。然后我们使用`switch()`测试元素类型，并使用适当数量的参数运行`setOptions()`。

```php
   if (isset($p['options'])) {
      list($a,$b,$c,$d) = $p['options'];
      switch ($p['type']) {
        case Generic::TYPE_RADIO    :
        case Generic::TYPE_CHECKBOX :
          $form->elements[$key]->setOptions($a,$b,$c,$d);
          break;
        case Generic::TYPE_SELECT   :
          $form->elements[$key]->setOptions($a,$b);
          break;
        default                     :
          $form->elements[$key]->setOptions($a,$b);
          break;
      }
    }
  }
```

8. 最后，我们返回表单对象，并结束该方法。

```php
  return $form;
} 
```

9. 理论上，此时，我们可以通过简单地迭代元素数组并运行`render()`方法，轻松地在视图逻辑中渲染表单。视图逻辑可能是这样的。

```php
<form name="status" method="get">
  <table id="status" class="display" cellspacing="0" width="100%">
    <?php foreach ($form->getElements() as $element) : ?>
      <?php echo $element->render(); ?>
    <?php endforeach; ?>
  </table>
</form>
```

10. 最后，我们返回表单对象，并结束该方法。

11. 接下来，我们需要在`Application\Form\Element`下定义一个离散的`Form`类。

```php
namespace Application\Form\Element;
class Form extends Generic
{
  public function getInputOnly()
  {
    $this->pattern = '<form name="%s" %s> ' . PHP_EOL;
    return sprintf($this->pattern, $this->name, 
                   $this->getAttribs());
  }
  public function closeTag()
  {
    return '</' . $this->type . '>';
  }
}
```

12. 回到`Application\Form\Factory`类，我们现在需要定义一个简单的方法来返回一个`sprintf()`包装器模式，作为输入的分装。举个例子，如果包装器是一个属性为`class="test"`的`div`，我们将产生这样的模式。`<div class="test">%s</div>`。然后我们的内容将被`sprintf()`函数替换成`%s`。

```php
protected function getWrapperPattern($wrapper)
{
  $type = $wrapper['type'];
  unset($wrapper['type']);
  $pattern = '<' . $type;
  foreach ($wrapper as $key => $value) {
    $pattern .= ' ' . $key . '="' . $value . '"';
  }
  $pattern .= '>%s</' . $type . '>';
  return $pattern;
}
```

13. 最后，我们准备好定义一个方法来进行整体表单的渲染。我们为每一个表单行获取包装器`sprintf()`模式。然后我们在元素中循环，渲染每一个元素，并将输出包装在行模式中。接下来，我们生成一个`Application\Form\Element\Form`实例。然后，我们检索表单包装器`sprintf()`模式，并检查`form_tag_inside_wrapper`标志，它告诉我们是否需要将表单标签放在表单包装器内部或外部。

```php
public static function render($form, $formConfig)
{
  $rowPattern = $form->getWrapperPattern(
  $formConfig['row_wrapper']);
  $contents   = '';
  foreach ($form->getElements() as $element) {
    $contents .= sprintf($rowPattern, $element->render());
  }
  $formTag = new Form($formConfig['name'], 
                  Generic::TYPE_FORM, 
                  '', 
                  array(), 
                  $formConfig['attributes']); 

  $formPattern = $form->getWrapperPattern(
  $formConfig['form_wrapper']);
  if (isset($formConfig['form_tag_inside_wrapper']) 
      && !$formConfig['form_tag_inside_wrapper']) {
        $formPattern = '%s' . $formPattern . '%s';
        return sprintf($formPattern, $formTag->getInputOnly(), 
        $contents, $formTag->closeTag());
  } else {
        return sprintf($formPattern, $formTag->getInputOnly() 
        . $contents . $formTag->closeTag());
  }
}
```

## 如何运行...

参考前面的代码，创建`Application\Form\Factory`和`Application\Form\Element\Form`类。

接下来，你可以定义一个`chap_06_form_factor.php`调用脚本，设置自动加载和锚定新类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Form\Generic;
use Application\Form\Factory;
```

接下来，使用第一个示例中定义的`$wrappers`数组来定义包装器。您也可以使用第二个示例中定义的`$statusList`数组。

看看是否有来自`$_POST`的任何状态输入。任何输入都将成为选定的键。否则，选定的键是默认的。

```php
$email    = $_POST['email']   ?? '';
$checked0 = $_POST['status0'] ?? 'U';
$checked1 = $_POST['status1'] ?? 'U';
$checked2 = $_POST['status2'] ?? ['U'];
$checked3 = $_POST['status3'] ?? ['U'];
```

现在您可以定义整体的表单配置。名称和属性参数用于配置表单标签本身。另外两个参数代表表单级和行级包装器。最后，我们提供了一个 `form_tag_inside_wrapper` 标志，用于指示表单标签不应该出现在包装器（即`<table>` ）内部。如果包装器是`<div>`，我们会将这个标志设置为`TRUE`。

```php
$formConfig = [ 
  'name'         => 'status_form',
  'attributes'   => ['id'=>'statusForm','method'=>'post', 'action'=>'chap_06_form_factory.php'],
  'row_wrapper'  => ['type' => 'tr', 'class' => 'row'],
  'form_wrapper' => ['type'=>'table','class'=>'table', 'id'=>'statusTable',
                     'class'=>'display','cellspacing'=>'0'],
                     'form_tag_inside_wrapper' => FALSE,
];
```

接下来，定义一个数组，该数组为工厂要创建的每个表单元素保存参数。数组键是表单元素的名称，必须是唯一的。

```php
$config = [
  'email' => [  
    'class'     => 'Application\Form\Generic',
    'type'      => Generic::TYPE_EMAIL, 
    'label'     => 'Email', 
    'wrappers'  => $wrappers,
    'attributes'=> ['id'=>'email','maxLength'=>128, 'title'=>'Enter address',
                    'required'=>'','value'=>strip_tags($email)]
  ],
  'password' => [
    'class'      => 'Application\Form\Generic',
    'type'       => Generic::TYPE_PASSWORD,
    'label'      => 'Password',
    'wrappers'   => $wrappers,
    'attributes' => ['id'=>'password',
    'title'      => 'Enter your password',
    'required'   => '']
  ],
  // etc.
];
```

最后，一定要生成表格。

```php
$form = Factory::generate($config);
```

实际的显示逻辑非常简单，因为我们只需调用表单级别的`render()`方法。

```php
<?= $form->render($form, $formConfig); ?>
```

下面是实际输出。

![](../../.gitbook/assets/image%20%2882%29.png)

