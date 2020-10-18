# 创建一个HTML选择元素生成器

生成一个HTML单选元素的过程与生成单选按钮的过程类似。但标签的结构不同，因为需要生成一个`SELECT`标签和一系列`OPTION`标签。

## 如何做...

1.首先，创建一个新的`Application\Form\Element\Select`类，该类扩展`Application\Form\Generic`。

2. 我们之所以要扩展`Generic`而不是`Radio`，是因为元素的结构完全不同。

```php
namespace Application\Form\Element;

use Application\Form\Generic;

class Select extends Generic
{
  // code
}
```

3. 类的常量和属性只需要稍微添加到 `Application\Form\Generic`中即可。与单选按钮或复选框不同，不需要考虑间隔符或所选文本的位置。

```php
const DEFAULT_OPTION_KEY = 0;
const DEFAULT_OPTION_VALUE = 'Choose';

protected $options;
protected $selectedKey = DEFAULT_OPTION_KEY;
```

4. 现在我们把注意力转移到设置选项上。由于HTML选择元素可以选择单个或多个值，所以`$selectedKey`属性可以是字符串或数组。因此，我们不为这个属性添加类型提示。然而，重要的是，我们要确定是否已经设置了多重属性。这可以通过继承父类的`$this->attributes`属性获得。

5. 如果已经设置了多重属性，就必须将`name`属性表述为一个数组。因此，如果是这种情况，我们将在名称后面加上`[]`。

```php
public function setOptions(array $options, $selectedKey = self::DEFAULT_OPTION_KEY)
{
  $this->options = $options;
  $this->selectedKey = $selectedKey;
  if (isset($this->attributes['multiple'])) {
    $this->name .= '[]';
  } 
}
```

{% hint style="info" %}
在PHP中，如果HTML select `multiple`属性被设置，而`name`属性没有被指定为一个数组，那么将只返回一个值。
{% endhint %}

6. 在我们定义核心的`getInputOnly()`方法之前，我们需要定义一个方法来生成`select`标签。然后我们使用`sprintf()`返回最终的HTML，使用`$pattern`、`$name`和`getAttribs()`的返回值作为参数。

7. 我们用`<select name="%s" %s>`替换`$pattern`的默认值。然后，我们在属性中循环，将它们添加为键值对，中间有空格。

```php
protected function getSelect()
{
  $this->pattern = '<select name="%s" %s> ' . PHP_EOL;
  return sprintf($this->pattern, $this->name, 
  $this->getAttribs());
}
```

8. 接下来，我们定义一个方法来获取将与选择标签相关联的选项标签。

9. 你还记得，`$this->options`数组中的`key`代表返回值，而数组中的`value`部分代表屏幕上出现的文本。如果`$this->selectedKey`是数组形式，我们检查值是否在数组中。否则，我们假设`$this->selectedKey`是一个字符串，我们简单地判断它是否等于键。如果选定的键匹配，我们添加选定的属性。

```php
protected function getOptions()
{
  $output = '';
  foreach ($this->options as $key => $value) {
    if (is_array($this->selectedKey)) {
        $selected = (in_array($key, $this->selectedKey)) 
        ? ' selected' : '';
    } else {
        $selected = ($key == $this->selectedKey) 
        ? ' selected' : '';
    }
        $output .= '<option value="' . $key . '"' 
        . $selected  . '>' 
        . $value 
        . '</option>';
  }
  return $output;
}
```

10. 最后我们准备覆盖核心的`getInputOnly()`方法。

11. 你会注意到，这个方法的逻辑只需要捕获前面代码中描述的`getSelect()`和`getOptions()`方法的返回值。我们还需要添加最后的`</select>`标签。

```php
public function getInputOnly()
{
  $output = $this->getSelect();
  $output .= $this->getOptions();
  $output .= '</' . $this->getType() . '>'; 
  return $output;
}
```

## 如何运行...

将上述代码复制到`Application/Form/Element`文件夹下新建一个`Select.php`文件。然后定义一个`chap_06_form_element_select.php`调用脚本，设置自动加载和锚定新类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Form\Generic;
use Application\Form\Element\Select;
```

接下来，使用第一个示例中定义的数组`$wrappers`定义包装器。你也可以使用在创建HTML单选元素生成器配方中定义的`$statusList`数组。然后你可以创建`select`元素的实例。第一个实例是单选，第二个是多选。

```php
$status1 = new Select('status1', 
           Generic::TYPE_SELECT, 
           'Status 1',
           $wrappers,
           ['id' => 'status1']);
$status2 = new Select('status2', 
           Generic::TYPE_SELECT, 
           'Status 2',
           $wrappers,
           ['id' => 'status2', 
            'multiple' => '', 
            'size' => '4']);
```

查看是否有来自`$_GET`的状态输入，并设置选项。任何输入都将成为选定的键。否则，选定的键就是默认值。你会记得，第二个实例是多重选择，所以从`$_GET`获得的值和默认设置都应该是数组的形式。

```php
$checked1 = $_GET['status1'] ?? 'U';
$checked2 = $_GET['status2'] ?? ['U'];
$status1->setOptions($statusList, $checked1);
$status2->setOptions($statusList, $checked2);
```

最后，一定要定义一个提交按钮（如本章创建通用表单元素生成器示例中所示）。

实际的显示逻辑与单选按钮示例相同，只是我们需要渲染两个独立的HTML选择实例。

```php
<form name="status" method="get">
<table id="status" class="display" cellspacing="0" width="100%">
  <tr><?= $status1->render(); ?></tr>
  <tr><?= $status2->render(); ?></tr>
  <tr><?= $submit->render(); ?></tr>
  <tr>
    <td colspan=2>
      <br>
      <pre>
        <?php var_dump($_GET); ?>
      </pre>
    </td>
  </tr>
</table>
</form>
```

下面是实际输出。

![](../../.gitbook/assets/image%20%2883%29.png)

此外，您还可以看到元素在查看源页面中的出现方式。

![](../../.gitbook/assets/image%20%2882%29.png)

