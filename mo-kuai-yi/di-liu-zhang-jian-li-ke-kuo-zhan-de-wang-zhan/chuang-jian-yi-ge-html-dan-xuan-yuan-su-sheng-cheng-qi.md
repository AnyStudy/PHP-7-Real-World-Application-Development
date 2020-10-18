# 创建一个HTML单选元素生成器

单选按钮元素生成器将与通用HTML表单元素生成器有相似之处。与任何通用元素一样，一组单选按钮需要能够显示整体标签和错误。然而，有两个主要的区别。

* 通常情况下，你会需要两个或更多的单选按钮。 
* 每个按钮都需要有自己的标签

## 如何做...

1.首先，创建一个新的 `Application\Form\Element\Radio`类，该类扩展`Application\Form\Generic`。

```php
namespace Application\Form\Element;
use Application\Form\Generic;
class Radio extends Generic
{
  // code
}
```

2. 接下来，我们定义与一组单选按钮的特殊需求有关的类常量和属性。

3. 在这个图示中，我们需要一个间隔符，它将被放置在单选按钮和它的标签之间，我们还需要决定将单选按钮标签放在实际按钮之前还是之后，因此，我们使用$after标志。如果我们需要一个默认值，或者如果我们要重新显示现有的表单数据，我们需要一种指定所选按键的方法。最后，我们需要一个选项数组，我们将从中填充按钮列表。

```php
const DEFAULT_AFTER = TRUE;
const DEFAULT_SPACER = '&nbps;';
const DEFAULT_OPTION_KEY = 0;
const DEFAULT_OPTION_VALUE = 'Choose';
      
protected $after = self::DEFAULT_AFTER;
protected $spacer = self::DEFAULT_SPACER;
protected $options = array();
protected $selectedKey = DEFAULT_OPTION_KEY;
```

4.鉴于我们正在扩展 `Application\Form\Generic`，我们可以选择扩展`__construct()`方法，或者，简单地定义一个可以用来设置特定选项的方法。在本例中，我们选择了后一种方法。

5. 为了确保属性`$this->options`被填充，第一个参数（`$options`）被定义为强制性的（没有默认值）。所有其他参数都是可选的。

```php
public function setOptions(array $options, 
  $selectedKey = self::DEFAULT_OPTION_KEY, 
  $spacer = self::DEFAULT_SPACER,
  $after  = TRUE)
{
  $this->after = $after;
  $this->spacer = $spacer;
  $this->options = $options;
  $this->selectedKey = $selectedKey;
}  
```

6. 最后，我们准备覆盖核心的`getInputOnly()`方法。

7. 我们将`id`属性保存到一个独立的变量`$baseId`中，之后再与`$count`结合，这样每个`id`属性都是唯一的。如果定义了与所选键相关联的选项，则将其分配为值；否则，我们使用默认值。

```php
public function getInputOnly()
{
  $count  = 1;
  $baseId = $this->attributes['id'];
```

8. 在`foreach()`循环里面，我们检查是否是选择的那个键。如果是，则为该单选按钮添加 `checked` 属性。然后我们调用父类`getInputOnly()`方法来返回每个按钮的HTML。注意，输入元素的值属性是选项数组key。按钮标签是选项数组元素的值。

```php
foreach ($this->options as $key => $value) {
  $this->attributes['id'] = $baseId . $count++;
  $this->attributes['value'] = $key;
  if ($key == $this->selectedKey) {
      $this->attributes['checked'] = '';
  } elseif (isset($this->attributes['checked'])) {
            unset($this->attributes['checked']);
  }
  if ($this->after) {
      $html = parent::getInputOnly() . $value;
  } else {
      $html = $value . parent::getInputOnly();
  }
  $output .= $this->spacer . $html;
  }
  return $output;
}
```

## 如何运行...

将前面的代码复制到`Application/Form/Element`文件夹下的`Radio.php`文件中。然后你可以定义一个`chap_06_form_element_radio.php`调用脚本来设置自动加载和锚定新类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Form\Generic;
use Application\Form\Element\Radio;
```

接下来，使用前面示例中定义的`$wrappers`数组定义包装器。

然后你可以定义一个`$status`数组，并通过向构造函数传递参数来创建一个元素实例。

```php
$statusList = [
  'U' => 'Unconfirmed',
  'P' => 'Pending',
  'T' => 'Temporary Approval',
  'A' => 'Approved'
];

$status = new Radio('status', 
          Generic::TYPE_RADIO, 
          'Status',
          $wrappers,
          ['id' => 'status']);
```

现在你可以看看是否有来自`$_GET`的状态输入，并设置选项。任何输入都将成为选定的键。否则，选定的键就是默认的。

```php
$checked = $_GET['status'] ?? 'U';
$status->setOptions($statusList, $checked, '<br>', TRUE);          
```

最后，别忘了定义一个提交按钮。

```php
$submit = new Generic('submit', 
          Generic::TYPE_SUBMIT,
          'Process',
          $wrappers,
          ['id' => 'submit','title' => 
          'Click to process','value' => 'Click Here']);
```

显示逻辑可能是这样的。

```php
<form name="status" method="get">
<table id="status" class="display" cellspacing="0" width="100%">
  <tr><?= $status->render(); ?></tr>
  <tr><?= $submit->render(); ?></tr>
  <tr>
    <td colspan=2>
      <br>
      <pre><?php var_dump($_GET); ?></pre>
    </td>
  </tr>
</table>
</form>
```

下面是实际输出。

![](../../.gitbook/assets/image%20%2884%29.png)

## 更多...

复选框元素生成器与HTML单选按钮生成器几乎相同。主要的区别是，一组复选框可以有多个值被选中。因此，你可以使用PHP数组符号来表示元素名称。元素类型应该是 `Generic::TYPE_CHECKBOX`。

