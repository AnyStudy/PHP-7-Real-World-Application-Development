# 创建通用表单元素生成器

要创建一个简单输出表单输入标签的函数是非常容易的，比如`<input type="text" name="whatever">`。然而，为了使表单生成器具有通用性，我们需要考虑更多的情况。下面是基本输入标签之外的一些其他考虑因素。

* 表单输入标签及其关联的HTML属性 
* 告诉用户他们正在输入什么信息的标签 
* 验证后显示输入错误的能力（稍后会介绍更多！） 
* 某种包装器，例如标记或HTML表标记

## 如何做...

1.首先，我们定义一个`Application\Form\Generic`类。这个类以后也将作为专门的表单元素的基类。

```php
namespace Application\Form;

class Generic
{
  // 一些代码 ...
}
```

2. 接下来，我们定义了一些类常量，这些常量一般会在表单元素生成中使用。

3. 前三个将成为与单个表单元素的主要组件相关联的键。然后我们定义支持的输入类型和默认值。

```php
const ROW = 'row';
const FORM = 'form';
const INPUT = 'input';
const LABEL = 'label';
const ERRORS = 'errors';
const TYPE_FORM = 'form';
const TYPE_TEXT = 'text';
const TYPE_EMAIL = 'email';
const TYPE_RADIO = 'radio';
const TYPE_SUBMIT = 'submit';
const TYPE_SELECT = 'select';
const TYPE_PASSWORD = 'password';
const TYPE_CHECKBOX = 'checkbox';
const DEFAULT_TYPE = self::TYPE_TEXT;
const DEFAULT_WRAPPER = 'div';
```

​4. 接下来，我们可以定义属性和一个设置属性的构造函数。

5. 在这个例子中，我们需要两个属性，`$name`和`$type`，因为没有这些属性我们就不能有效地使用元素。其他构造参数是可选的。此外，为了使一个表单元素基于另一个表单元素，我们包含了一个规定，即第二个参数`$type`可以是`Application\Form\Generic`的一个实例，在这种情况下，我们只需运行`getters`（后面讨论）来填充属性。

```php
protected $name;
protected $type    = self::DEFAULT_TYPE;
protected $label   = '';
protected $errors  = array();
protected $wrappers;
protected $attributes;    // HTML 表单属性
protected $pattern =  '<input type="%s" name="%s" %s>';

public function __construct($name, 
                $type, 
                $label = '',
                array $wrappers = array(), 
                array $attributes = array(),
                array $errors = array())
{
  $this->name = $name;
  if ($type instanceof Generic) {
      $this->type       = $type->getType();
      $this->label      = $type->getLabelValue();
      $this->errors     = $type->getErrorsArray();
      $this->wrappers   = $type->getWrappers();
      $this->attributes = $type->getAttributes();
  } else {
      $this->type       = $type ?? self::DEFAULT_TYPE;
      $this->label      = $label;
      $this->errors     = $errors;
      $this->attributes = $attributes;
      if ($wrappers) {
          $this->wrappers = $wrappers;
      } else {
          $this->wrappers[self::INPUT]['type'] =
            self::DEFAULT_WRAPPER;
          $this->wrappers[self::LABEL]['type'] = 
            self::DEFAULT_WRAPPER;
          $this->wrappers[self::ERRORS]['type'] = 
            self::DEFAULT_WRAPPER;
    }
  }
  $this->attributes['id'] = $name;
}
```

{% hint style="warning" %}
注意，`$wrappers`有三个主要的子键。`INPUT`, `LABEL`, 和`ERRORS`. 这允许我们为标签、输入标签和错误定义单独的包装器。
{% endhint %}

6. 在定义为标签、输入标签和错误产生HTML的核心方法之前，我们应该定义一个 `getWrapperPattern()` 方法，该方法将为标签、输入和错误显示产生相应的包装标签。

7. 例如，如果包装器被定义为`<div>`，并且它的属性包括`['class'=>'label']`，那么这个方法将返回一个`sprintf()`格式的模式，看起来像这样。`<div class="label">%s</div>`。例如，最终产生的HTML标签将替换`%s`。

8. 下面是`getWrapperPattern()`方法的样子。

```php
public function getWrapperPattern($type)
{
  $pattern = '<' . $this->wrappers[$type]['type'];
  foreach ($this->wrappers[$type] as $key => $value) {
    if ($key != 'type') {
      $pattern .= ' ' . $key . '="' . $value . '"';
    }
  }
  $pattern .= '>%s</' . $this->wrappers[$type]['type'] . '>';
  return $pattern;
}
```

9. 现在我们准备好定义`getLabel()`方法了。这个方法需要做的就是使用`sprintf()`将标签插入到包装器中。

```php
public function getLabel()
{
  return sprintf($this->getWrapperPattern(self::LABEL), 
                 $this->label);
}
```

10. 为了生成核心输入标签，我们需要有一种方法来组装属性。幸运的是，只要将它们以关联数组的形式提供给构造函数，就可以轻松实现这一点。在这种情况下，我们需要做的就是定义一个 `getAttribs()` 方法，该方法产生一个由键值对组成的字符串，并以空格分隔。我们使用`trim()`返回最终值，以去除多余的空格。

11. 如果元素包含`value`或`href`属性，则出于安全原因，我们假定这些值是用户提供的（因此值得怀疑），所以我们需要转义。 因此，我们需要添加一个`if`语句来检查然后使用`htmlspecialchars()`或`urlencode()`：

```php
public function getAttribs()
{
  foreach ($this->attributes as $key => $value) {
    $key = strtolower($key);
    if ($value) {
      if ($key == 'value') {
        if (is_array($value)) {
            foreach ($value as $k => $i) 
              $value[$k] = htmlspecialchars($i);
        } else {
            $value = htmlspecialchars($value);
        }
      } elseif ($key == 'href') {
          $value = urlencode($value);
      }
      $attribs .= $key . '="' . $value . '" ';
    } else {
        $attribs .= $key . ' ';
    }
  }
  return trim($attribs);
}
```

12. 对于核心的输入标签，我们将逻辑分成两个独立的方法。主要方法`getInputOnly()`只产生HTML输入标签。第二个方法，`getInputWithWrapper()`，产生嵌入包装器中的输入。分离的原因是，当创建衍生类时，例如生成单选按钮的类，我们将不需要包装器。

```php
public function getInputOnly()
{
  return sprintf($this->pattern, $this->type, $this->name, 
                 $this->getAttribs());
}

public function getInputWithWrapper()
{
  return sprintf($this->getWrapperPattern(self::INPUT), 
                 $this->getInputOnly());
}
```

13.现在我们定义一个显示元素验证错误的方法。我们假设错误是以数组的形式提供的，如果没有错误，我们返回一个空字符串。否则，错误将以`<ul><li>error 1</li><li>error 2</li></ul>`等形式显示。

```php
public function getErrors()
{
  if (!$this->errors || count($this->errors == 0)) return '';
  $html = '';
  $pattern = '<li>%s</li>';
  $html .= '<ul>';
  foreach ($this->errors as $error)
  $html .= sprintf($pattern, $error);
  $html .= '</ul>';
  return sprintf($this->getWrapperPattern(self::ERRORS), $html);
}
```

14. 对于某些属性，我们可能需要对属性的各个方面进行更有限的控制。例如，我们可能需要在已经存在的错误数组中添加一个错误。另外，设置一个单一的属性也可能是有用的。

```php
public function setSingleAttribute($key, $value)
{
  $this->attributes[$key] = $value;
}
public function addSingleError($error)
{
  $this->errors[] = $error;
}
```

15. 最后，我们定义了`getter`和`setter`，允许我们检索或设置属性的值。例如，你可能已经注意到`$pattern`的默认值是`<input type="%s" name="%s" %s>`。对于某些标签（例如，选择和表单标签），我们需要将此属性设置为不同的值。

```php
public function setPattern($pattern)
{
  $this->pattern = $pattern;
}
public function setType($type)
{
  $this->type = $type;
}
public function getType()
{
  return $this->type;
}
public function addSingleError($error)
{
  $this->errors[] = $error;
}
// 定义类似的获取和设置方法
// 用于名称、标签、包装物、错误和属性。
```

16. 我们还需要添加能给出标签值（不是HTML）的方法，以及错误数组。

```php
public function getLabelValue()
{
  return $this->label;
}
public function getErrorsArray()
{
  return $this->errors;
}
```

## 如何运行...

请确保将前面所有的代码复制到一个单一的`Application\Form\Generic`类中。然后你可以定义一个`chap_06_form_element_generator.php`调用脚本来设置自动加载和锚定新类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Form\Generic;
```

接下来，定义包装器。为了说明问题，我们将使用HTML表格数据和头标签。注意标签使用`TH`，而输入和错误使用`TD`。

```php
$wrappers = [
  Generic::INPUT => ['type' => 'td', 'class' => 'content'],
  Generic::LABEL => ['type' => 'th', 'class' => 'label'],
  Generic::ERRORS => ['type' => 'td', 'class' => 'error']
];
```

现在你可以通过向构造函数传递参数来定义一个电子邮件元素。

```php
$email = new Generic('email', Generic::TYPE_EMAIL, 'Email', $wrappers,
                    ['id' => 'email',
                     'maxLength' => 128,
                     'title' => 'Enter address',
                     'required' => '']);
```

或者，使用设置器定义密码元素。

```php
$password = new Generic('password', $email);
$password->setType(Generic::TYPE_PASSWORD);
$password->setLabel('Password');
$password->setAttributes(['id' => 'password',
                          'title' => 'Enter your password',
                          'required' => '']);
```

最后，一定要定义一个提交按钮。

```php
$submit = new Generic('submit', 
  Generic::TYPE_SUBMIT,
  'Login',
  $wrappers,
  ['id' => 'submit','title' => 'Click to login','value' => 'Click Here']);
```

实际的显示逻辑可能是这样的。

```php
<div class="container">
  <!-- Login Form -->
  <h1>Login</h1>
  <form name="login" method="post">
  <table id="login" class="display" 
    cellspacing="0" width="100%">
    <tr><?= $email->render(); ?></tr>
    <tr><?= $password->render(); ?></tr>
    <tr><?= $submit->render(); ?></tr>
    <tr>
      <td colspan=2>
        <br>
        <?php var_dump($_POST); ?>
      </td>
    </tr>
  </table>
  </form>
</div>
```

下面是实际输出。

![](../../.gitbook/assets/image%20%2881%29.png)

