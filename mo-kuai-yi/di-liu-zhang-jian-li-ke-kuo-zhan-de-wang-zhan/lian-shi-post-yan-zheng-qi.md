# 链式 $\_POST 验证器

这个示例的重头戏已经在前面的示例中完成了，核心功能由`Application\Filter\AbstractFilter`定义。核心功能由`Application\Filter\AbstractFilter`定义。实际的验证是由一个验证回调数组来完成的。

## 如何做...

1.查看前面的示例，链式$\_POST过滤器。我们将在这个示例中使用所有的类和配置文件，除非这里有说明。

2. 首先，我们定义一个验证回调的配置数组。与前面的配方一样，每个回调应该实现`Application\Filter\CallbackInterface`，并且应该返回`Application\Filter\Result`的实例。验证器将采取这种通用形式。

```php
use Application\Filter\ { Result, Messages, CallbackInterface };
$config = [
  // validator callbacks
  'validators' => [
    'key' => new class () implements CallbackInterface 
    {
      public function __invoke($item, $params) : Result
      {
        // validation logic goes here
        return new Result($valid, $error);
      }
    },
    // etc.
```

3.接下来，我们定义了一个`Application\Filter\Validator`类，它在分配的数组中循环，根据其分配的验证器回调测试每个数据项。我们使这个类扩展了`AbstractFilter`，以便提供前面描述的核心功能。

```php
namespace Application\Filter;
class Validator extends AbstractFilter
{
  // code
}
```

4. 在这个类中，我们定义了一个核心的`process()`方法，它扫描一个数据数组，并根据分配的数组应用验证器。如果没有为这个数据集分配验证器，我们就简单地返回`$valid`的当前状态（即为`TRUE`）。

```php
public function process(array $data)
{
  $valid = TRUE;
  if (!(isset($this->assignments) 
      && count($this->assignments))) {
        return $valid;
  }
```

5. 否则，我们将`$this->results`初始化为一个`Result`对象数组，其中`$item`属性设置为`TRUE`，`$messages`属性为空数组。

```php
foreach ($data as $key => $value) {
  $this->results[$key] = new Result(TRUE, array());
}
```

6. 然后我们复制`$this->assignments`并检查是否有全局过滤器（由"\*"键标识）。如果有，我们运行`processGlobal()`，然后取消设置"\*"键。

```php
$toDo = $this->assignments;
if (isset($toDo['*'])) {
  $this->processGlobalAssignment($toDo['*'], $data);
  unset($toDo['*']);
}
```

7. 最后，我们循环处理任何剩余的赋值，调用 `processAssignment()`。这是检查`assignments`数组中是否有任何字段从数据中丢失的理想场所。注意，如果任何验证回调返回`FALSE`，我们将`$valid`设置为`FALSE`。

```php
foreach ($toDo as $key => $assignment) {
  if (!isset($data[$key])) {
      $this->results[$key] = 
      new Result(FALSE, $this->missingMessage);
  } else {
      $this->processAssignment(
        $assignment, $key, $data[$key]);
  }
  if (!$this->results[$key]->item) $valid = FALSE;
  }
  return $valid;
}
```

8. 您还记得，每个赋值都与数据字段有键，并代表该字段的回调数组。因此，在`processGlobalAssignment()`中，我们需要循环浏览回调数组。但在这种情况下，由于这些赋值是全局的，我们还需要循环浏览整个数据集，并依次应用每个全局过滤器。

9. 与等效的`Application\Filter\Fiter::processGlobalAssignment()`方法相比，我们需要调用`mergeValidationResults()`。原因是，如果`$result->item`的值已经是`FALSE`，我们需要确保它不会随后被一个`TRUE`的值覆盖。链中任何返回`FALSE`的验证器必须覆盖任何其他验证结果。

```php
protected function processGlobalAssignment($assignment, $data)
{
  foreach ($assignment as $callback) {
    if ($callback === NULL) continue;
    foreach ($data as $k => $value) {
      $result = $this->callbacks[$callback['key']]
      ($value, $callback['params']);
      $this->results[$k]->mergeValidationResults($result);
    }
  }
}
```

10. 当我们定义了`processAssignment()`，以类似于`processGlobalAssignment()`的方式，我们需要执行分配给每个数据键的剩余回调，再次调用`mergeValidationResults()`。

```php
protected function processAssignment($assignment, $key, $value)
{
  foreach ($assignment as $callback) {
    if ($callback === NULL) continue;
        $result = $this->callbacks[$callback['key']]
       ($value, $callback['params']);
        $this->results[$key]->mergeValidationResults($result);
    }
  }
```

## 如何做...

与前面的示例一样，一定要定义以下类。

* `Application\Filter\Result`
* `Application\Filter\CallbackInterface`
* `Application\Filter\Messages`
* `Application\Filter\AbstractFilter`

你可以使用`chap_06_post_data_config_messages.php`文件，前面的示例中也有介绍。

接下来，在`Application\Filter`文件夹中创建一个`Validator.php`文件。放置步骤3至10中描述的代码。

接下来，创建一个`chap_06_post_data_config_callbacks.php`回调配置文件，其中包含验证回调的配置，如步骤2所述。每个回调都应该遵循这个通用模板。

```php
'validation_key' => new class () implements CallbackInterface 
{
  public function __invoke($item, $params) : Result
  {
    $error = array();
    $valid = /* perform validation operation on $item */
    if (!$valid) 
    $error[] = Messages::$messages['validation_key'];
    return new Result($valid, $error);
  }
}
```

现在你可以创建一个`chap_06_post_data_validation.php`调用脚本，初始化自动加载并包含配置脚本。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
include __DIR__ . '/chap_06_post_data_config_messages.php';
include __DIR__ . '/chap_06_post_data_config_callbacks.php';
```

接下来，定义一个赋值数组，将数据字段映射到验证器回调键。

```php
$assignments = [
  'first_name'       => [ ['key' => 'length',  
  'params'   => ['min' => 1, 'max' => 128]], 
                ['key' => 'alnum',   
  'params'   => ['allowWhiteSpace' => TRUE]],
                ['key'   => 'required','params' => []] ],
  'last_name'=> [ ['key' => 'length',  
  'params'   => ['min'   => 1, 'max' => 128]],
                ['key'   => 'alnum',   
  'params'   => ['allowWhiteSpace' => TRUE]],
                ['key'   => 'required','params' => []] ],
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
  'phone'         => [ ['key' => 'phone', 'params' => []] ],
  'country'       => [ ['key' => 'in_array',
  'params'        => $countries ], 
                     ['key' => 'required','params' => []] ],
  'email'         => [ ['key' => 'email', 'params' => [] ],
                     ['key' => 'length',  
  'params'        => ['max' => 250] ], 
                     ['key' => 'required','params' => [] ] ],
  'budget'        => [ ['key' => 'float', 'params' => []] ]
];
```

对于测试数据，使用前面配方中描述的`chap_06_post_data_filtering.php`文件中定义的好数据和坏数据。之后，你就可以创建一个`Application\Filter\Validator`实例，并测试数据。

```php
$validator = new Application\Filter\Validator($config['validators'], $assignments);
$validator->setSeparator(PHP_EOL);
$validator->process($badData);
echo $validator->getMessageString(40, '%14s : %-26s' . PHP_EOL);
var_dump($validator->getItemsAsArray());
$validator->process($goodData);
echo $validator->getMessageString(40, '%14s : %-26s' . PHP_EOL);
var_dump($validator->getItemsAsArray());
```

正如预期的那样，好的数据不会产生任何验证错误。另一方面，坏数据会产生以下输出：

![](../../.gitbook/assets/image%20%2888%29.png)

注意，缺失的字段，`address` 和`state_province`验证为`FALSE`，并返回缺失项目消息。

