# 将验证绑定到表单

当表单首次呈现时，将一个表单类（如前面示例中描述的`Application\Form\Factory`）与一个可以执行过滤或验证的类（如前面配方中描述的`Application\Filter\*`）绑定是没有什么价值的。然而，一旦表单数据被提交，兴趣就会增长。如果表单数据未能通过验证，则可以对值进行过滤，然后重新显示。验证错误信息可以与表单元素绑定，并呈现在表单字段旁边。

## 如何做...

1.首先，一定要实现在 《实现表单工厂》、《链式$\_POST过滤器》和 《链式$\_POST验证器》示例中定义的类。

2. 现在我们将注意力转向`Application/Form/Factory`类，并添加属性和设置器，允许我们附加`Application/Filter/Filter`和`Application/Filter/Validator`的实例。我们还需要定义`$data`，它将用于保留过滤和验证的数据。

```php
const DATA_NOT_FOUND = 'Data not found. Run setData()';
const FILTER_NOT_FOUND = 'Filter not found. Run setFilter()';
const VALIDATOR_NOT_FOUND = 'Validator not found. Run setValidator()';

protected $filter;
protected $validator;
protected $data;

public function setFilter(Filter $filter)
{
  $this->filter = $filter;
}

public function setValidator(Validator $validator)
{
  $this->validator = $validator;
}

public function setData($data)
{
  $this->data = $data;
}
```

3. 接下来，我们定义了一个`validate()`方法，该方法调用嵌入式`Application/Filter/Validator`实例的`process()`方法。我们检查`$data`和`$validator`是否存在。如果不存在，就会抛出相应的异常，并指示需要先运行哪个方法。

```php
public function validate()
{
  if (!$this->data)
  throw new RuntimeException(self::DATA_NOT_FOUND);
          
  if (!$this->validator)
  throw new RuntimeException(self::VALIDATOR_NOT_FOUND);
```

4. 调用`process()`方法后，我们将验证结果消息与表单元素消息关联起来。需要注意的是，`process()`方法会返回一个布尔值，代表数据集的整体验证状态。当表单在验证失败后重新显示时，错误信息将出现在每个元素旁边。

```php
$valid = $this->validator->process($this->data);

foreach ($this->elements as $element) {
  if (isset($this->validator->getResults()
      [$element->getName()])) {
        $element->setErrors($this->validator->getResults()
        [$element->getName()]->messages);
      }
    }
    return $valid;
  }
```

5. 以类似的方式，我们定义了一个`filter()`方法，该方法调用嵌入式`Application/Filter/Filter`实例的`process()`方法。与步骤3中描述的`validate()`方法一样，我们需要检查`$data`和`$filter`是否存在。如果缺少任何一个，我们将抛出一个带有适当消息的`RuntimeException`。

```php
public function filter()
{
  if (!$this->data)
  throw new RuntimeException(self::DATA_NOT_FOUND);
          
  if (!$this->filter)
  throw new RuntimeException(self::FILTER_NOT_FOUND);
```

6. 然后我们运行`process()`方法，产生一个`Result`对象数组，其中`$item`属性代表过滤链的最终结果。然后我们对结果进行循环，如果对应的`$element`键匹配，则将值属性设置为过滤后的值。我们还添加过滤过程中产生的任何消息。当表单重新显示时，所有的值属性都会显示过滤后的结果。

```php
$this->filter->process($this->data);
foreach ($this->filter->getResults() as $key => $result) {
  if (isset($this->elements[$key])) {
    $this->elements[$key]
    ->setSingleAttribute('value', $result->item);
    if (isset($result->messages) 
        && count($result->messages)) {
      foreach ($result->messages as $message) {
        $this->elements[$key]->addSingleError($message);
      }
    }
  }      
}
}
```

## 如何运行...

你可以像上面描述的那样，先对`Application/Form/Factory`进行修改。对于测试目标，您可以使用链式`$_POST`过滤器示例中的如何做部分中所示的`professor`数据库表。各种列的设置应该能让您了解要定义哪些表单元素、过滤器和验证器。

作为一个例子，你可以定义一个`chap_06_tying_filters_to_form_definitions.php`文件，它将包含表单包装器、元素和过滤器分配的定义。下面是一些例子:

```php
<?php
use Application\Form\Generic;

define('VALIDATE_SUCCESS', 'SUCCESS: form submitted ok!');
define('VALIDATE_FAILURE', 'ERROR: validation errors detected');

$wrappers = [
  Generic::INPUT  => ['type' => 'td', 'class' => 'content'],
  Generic::LABEL  => ['type' => 'th', 'class' => 'label'],
  Generic::ERRORS => ['type' => 'td', 'class' => 'error']
];

$elements = [
  'first_name' => [  
     'class'     => 'Application\Form\Generic',
     'type'      => Generic::TYPE_TEXT, 
     'label'     => 'First Name', 
     'wrappers'  => $wrappers,
     'attributes'=> ['maxLength'=>128,'required'=>'']
  ],
  'last_name'   => [  
    'class'     => 'Application\Form\Generic',
    'type'      => Generic::TYPE_TEXT, 
    'label'     => 'Last Name', 
    'wrappers'  => $wrappers,
    'attributes'=> ['maxLength'=>128,'required'=>'']
  ],
    // etc.
];

// overall form config
$formConfig = [ 
  'name'       => 'prospectsForm',
  'attributes' => [
'method'=>'post',
'action'=>'chap_06_tying_filters_to_form.php'
],
  'row_wrapper'  => ['type' => 'tr', 'class' => 'row'],
  'form_wrapper' => [
    'type'=>'table',
    'class'=>'table',
    'id'=>'prospectsTable',
    'class'=>'display','cellspacing'=>'0'
  ],
  'form_tag_inside_wrapper' => FALSE,
];

$assignments = [
  'first_name'    => [ ['key' => 'length',  
  'params'        => ['min' => 1, 'max' => 128]], 
                     ['key' => 'alnum',   
  'params'        => ['allowWhiteSpace' => TRUE]],
                     ['key' => 'required','params' => []] ],
  'last_name'     => [ ['key' => 'length',  
  'params'        => ['min' => 1, 'max' => 128]],
                     ['key' => 'alnum',   
  'params'        => ['allowWhiteSpace' => TRUE]],
                     ['key' => 'required','params' => []] ],
  'address'       => [ ['key' => 'length',  
  'params'        => ['max' => 256]] ],
  'city'          => [ ['key' => 'length',  
  'params'        => ['min' => 1, 'max' => 64]] ], 
  'state_province'=> [ ['key' => 'length',  
  'params'        => ['min' => 1, 'max' => 32]] ], 
  'postal_code'   => [ ['key' => 'length',  
  'params'        => ['min' => 1, 'max' => 16] ], 
                     ['key' => 'alnum',   
  'params'        => ['allowWhiteSpace' => TRUE]],
                     ['key' => 'required','params' => []] ],
  'phone'         => [ ['key' => 'phone',   'params' => []] ],
  'country'       => [ ['key' => 'in_array',
  'params'        => $countries ], 
                     ['key' => 'required','params' => []] ],
  'email'         => [ ['key' => 'email',   'params' => [] ],
                     ['key' => 'length',  
  'params'        => ['max' => 250] ], 
                     ['key' => 'required','params' => [] ] ],
  'budget'        => [ ['key' => 'float',   'params' => []] ]
];
```

你可以使用前面示例中介绍的已经存在的`chap_06_post_data_config_callbacks.php`和`chap_06_post_data_config_messages.php`文件。最后，定义一个`chap_06_tying_filters_to_form.php`文件，设置自动加载并包含这三个配置文件。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
include __DIR__ . '/chap_06_post_data_config_messages.php';
include __DIR__ . '/chap_06_post_data_config_callbacks.php';
include __DIR__ . '/chap_06_tying_filters_to_form_definitions.php';
```

接下来，你可以创建表单工厂、过滤器和验证器类的实例。

```php
use Application\Form\Factory;
use Application\Filter\ { Validator, Filter };
$form = Factory::generate($elements);
$form->setFilter(new Filter($callbacks['filters'], $assignments['filters']));
$form->setValidator(new Validator($callbacks['validators'], $assignments['validators']));
```

然后你可以检查是否有任何`$_POST`数据。如果有，则进行验证和过滤。

```php
$message = '';
if (isset($_POST['submit'])) {
  $form->setData($_POST);
  if ($form->validate()) {
    $message = VALIDATE_SUCCESS;
  } else {
    $message = VALIDATE_FAILURE;
  }
  $form->filter();
}
?>
```

视图逻辑非常简单：只需渲染表单。任何验证信息和各种元素的值将作为验证和过滤的一部分被分配。

```php
 <?= $form->render($form, $formConfig); ?>
```

下面是一个使用不良表格数据的例子。

![](../../.gitbook/assets/image%20%2888%29.png)

请注意过滤和验证信息。也请注意不良标签。

![](../../.gitbook/assets/image%20%2887%29.png)

