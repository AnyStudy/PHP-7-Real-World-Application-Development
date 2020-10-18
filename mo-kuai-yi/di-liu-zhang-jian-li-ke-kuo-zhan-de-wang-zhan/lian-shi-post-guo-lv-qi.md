# 链式 $\_POST 过滤器

在处理用户从在线表单提交的数据时，适当的过滤和验证是一个常见的问题。这也可以说是网站的头号安全漏洞。此外，过滤器和验证器分散在应用程序中是相当尴尬的。一个链式机制将整齐地解决这些问题，并允许你对过滤器和验证器的处理顺序进行控制。

## 如何做...

1.有一个鲜为人知的PHP函数，`filter_input_array()`，乍一看，似乎很适合这个任务。然而，如果深入研究它的功能，很快就会发现这个函数是在早期设计的，不符合现代防攻击和灵活性的要求。相应地，我们将提出一种更灵活的机制，基于一系列回调来执行过滤和验证。

{% hint style="info" %}
过滤和验证之间的区别在于，过滤可能会删除或转换值。另一方面，验证使用适合数据性质的标准测试数据，并返回一个布尔结果。
{% endhint %}

2.为了增加灵活性，我们将使我们的基础过滤器和验证类相对较轻。我们的意思是不定义任何特定的过滤器或验证方法。相反，我们将完全基于回调的配置数组来操作。为了保证过滤和验证结果的兼容性，我们还将定义一个特定的结果对象，`Application\Filter\Result`。

3. `Result`类的主要功能是保存一个`$item`值，它将是过滤后的值或验证后的布尔结果。另一个属性，`$messages`，将持有一个在过滤或验证操作期间填充的消息数组。在构造函数中，为`$messages`提供的值被表述为一个数组。你可能会发现这两个属性都被定义为`public`。这是为了方便访问。

```php
namespace Application\Filter;

class Result
{
  
  public $item;  // (mixed) filtered data | (bool) result of validation
  public $messages = array();  // [(string) message, (string) message ]
  
  public function __construct($item, $messages)
  {
    $this->item = $item;
    if (is_array($messages)) {
        $this->messages = $messages;
    } else {
        $this->messages = [$messages];
    }
  }
```

4. 我们还定义了一个方法，允许我们将这个 `Result` 实例与另一个合并。这一点很重要，因为在某些时候，我们将通过一连串的过滤器来处理同一个值。在这种情况下，我们希望新过滤的值覆盖现有的值，但我们希望消息被合并。

```php
public function mergeResults(Result $result)
{
  $this->item = $result->item;
  $this->mergeMessages($result);
}

public function mergeMessages(Result $result)
{
  if (isset($result->messages) && is_array($result->messages)) {
    $this->messages = array_merge($this->messages, $result->messages);
  }
}
```

5. 最后，为了完成这个类的方法，我们添加一个合并验证结果的方法。这里重要的考虑因素是，无论在验证链的上下游，任何一个`FALSE`的值都必须导致整个结果为`FALSE`。

```php
public function mergeValidationResults(Result $result)
{
  if ($this->item === TRUE) {
    $this->item = (bool) $result->item;
  }
  $this->mergeMessages($result);
  }

}
```

6. 接下来，为了确保回调产生兼容的结果，我们将定义一个`Application\Filter\CallbackInterface`接口。你会注意到我们正在利用PHP 7的能力对返回值进行数据类型化，以确保我们得到的是一个`Result`实例。

```php
namespace Application\Filter;
interface CallbackInterface
{
  public function __invoke ($item, $params) : Result;
}
```

7. 每个回调都应该引用相同的消息集。相应地，我们定义了一个`Application\Filter\Messages`类，该类具有一系列静态属性。我们提供的方法可以设置所有的消息，或者只设置一条消息。为了便于访问，`$messages`属性已经被公开。

```php
namespace Application\Filter;
class Messages
{
  const MESSAGE_UNKNOWN = 'Unknown';
  public static $messages;
  public static function setMessages(array $messages)
  {
    self::$messages = $messages;
  }
  public static function setMessage($key, $message)
  {
    self::$messages[$key] = $message;
  }
  public static function getMessage($key)
  {
    return self::$messages[$key] ?? self::MESSAGE_UNKNOWN;
  }
}
```

8. 我们现在可以定义一个实现核心功能的`Application\Web\AbstractFilter`类。如前所述，这个类将是相对轻量级的，我们不需要担心具体的过滤器或验证器，因为它们将通过配置提供。我们使用PHP 7标准PHP库（SPL）中提供的`UnexpectedValueException`类，以便在其中一个回调没有实现`CallbackInterface`时抛出一个描述性异常。

```php
namespace Application\Filter;
use UnexpectedValueException;
abstract class AbstractFilter
{
  // code described in the next several bullets
```

9. 首先，我们定义了有用的类常量，这些常量持有各种内务管理值。这里显示的最后四个常量控制了消息的显示格式，以及如何描述丢失的数据。

```php
const BAD_CALLBACK = 'Must implement CallbackInterface';
const DEFAULT_SEPARATOR = '<br>' . PHP_EOL;
const MISSING_MESSAGE_KEY = 'item.missing';
const DEFAULT_MESSAGE_FORMAT = '%20s : %60s';
const DEFAULT_MISSING_MESSAGE = 'Item Missing';
```

10. 接下来，我们定义核心属性。`$separator`与过滤和验证消息一起使用。`$callbacks`表示执行过滤和验证的回调数组。`$assignments`将数据字段映射到过滤器和/或验证器。`$missingMessage`被表示为一个属性，这样它就可以被覆盖（也就是对于多语言网站）。最后，`$results`是一个`Application\Filter\Result`对象的数组，由过滤或验证操作填充。

```php
protected $separator;    // used for message display
protected $callbacks;
protected $assignments;
protected $missingMessage;
protected $results = array();
```

11. 此时，我们可以构建`__construct()`方法。它的主要功能是设置回调和赋值的数组。它还为分隔符（用于消息显示）和缺失的消息设置值或接受默认值。

```php
public function __construct(array $callbacks, array $assignments, 
                            $separator = NULL, $message = NULL)
{
  $this->setCallbacks($callbacks);
  $this->setAssignments($assignments);
  $this->setSeparator($separator ?? self::DEFAULT_SEPARATOR);
  $this->setMissingMessage($message 
                           ?? self::DEFAULT_MISSING_MESSAGE);
}
```

12. 接下来，我们定义了一系列方法，允许我们设置或删除回调。请注意，我们允许获取和设置一个回调。如果你有一个通用的回调集，并且只需要修改一个回调，这就很有用。你还会注意到`setOneCall()`会检查回调是否实现了`CallbackInterface`。如果没有，就会抛出一个`UnexpectedValueException`。

```php
public function getCallbacks()
{
  return $this->callbacks;
}

public function getOneCallback($key)
{
  return $this->callbacks[$key] ?? NULL;
}

public function setCallbacks(array $callbacks)
{
  foreach ($callbacks as $key => $item) {
    $this->setOneCallback($key, $item);
  }
}

public function setOneCallback($key, $item)
{
  if ($item instanceof CallbackInterface) {
      $this->callbacks[$key] = $item;
  } else {
      throw new UnexpectedValueException(self::BAD_CALLBACK);
  }
}

public function removeOneCallback($key)
{
  if (isset($this->callbacks[$key])) 
  unset($this->callbacks[$key]);
}
```

13. 结果处理的方法很简单。为了方便，我们增加了`getItemsAsArray()`，否则`getResults()`将返回一个`Result`对象的数组。

```php
public function getResults()
{
  return $this->results;
}

public function getItemsAsArray()
{
  $return = array();
  if ($this->results) {
    foreach ($this->results as $key => $item) 
    $return[$key] = $item->item;
  }
  return $return;
}
```

14. 检索消息只是一个循环浏览`$this ->results`数组并提取`$messages`属性的问题。为了方便起见，我们还添加了带有一些格式选项的 `getMessageString()` 。为了方便地生成一个消息数组，我们使用了 PHP 7 的 `yield from` 语法。这样做的效果是把 `getMessages()` 变成了一个委托生成器。消息数组成为一个子生成器。

```php
public function getMessages()
{
  if ($this->results) {
      foreach ($this->results as $key => $item) 
      if ($item->messages) yield from $item->messages;
  } else {
      return array();
  }
}

public function getMessageString($width = 80, $format = NULL)
{
  if (!$format)
  $format = self::DEFAULT_MESSAGE_FORMAT . $this->separator;
  $output = '';
  if ($this->results) {
    foreach ($this->results as $key => $value) {
      if ($value->messages) {
        foreach ($value->messages as $message) {
          $output .= sprintf(
            $format, $key, trim($message));
        }
      }
    }
  }
  return $output;
}
```

15. 最后，我们定义了一组mixed的getter和setter。

```php
  public function setMissingMessage($message)
  {
    $this->missingMessage = $message;
  }
  public function setSeparator($separator)
  {
    $this->separator = $separator;
  }
  public function getSeparator()
  {
    return $this->separator;
  }
  public function getAssignments()
  {
    return $this->assignments;
  }
  public function setAssignments(array $assignments)
  {
    $this->assignments = $assignments;
  }
  // closing bracket for class AbstractFilter
}
```

16. 过滤和验证虽然经常一起进行，但也同样经常单独进行。因此，我们为每一个定义了离散的类。我们先从`Application\Filter\Filter`开始。我们让这个类扩展`AbstractFilter`，以便提供前面描述的核心功能。

```php
namespace Application\Filter;
class Filter extends AbstractFilter
{
  // code
}
```

17. 在这个类中，我们定义了一个核心的`process()`方法，它扫描一个数据数组，并根据分配的数组应用过滤器。如果没有为这个数据集分配过滤器，我们就简单地返回`NULL`。

```php
public function process(array $data)
{
  if (!(isset($this->assignments) 
      && count($this->assignments))) {
        return NULL;
  }
```

18. 否则，我们将`$this->results`初始化为一个Result对象数组，其中`$item`属性是`$data`的原始值，`$messages`属性是一个空数组。

```php
foreach ($data as $key => $value) {
  $this->results[$key] = new Result($value, array());
}
```

19. 然后我们复制`$this->assignments`并检查是否有全局过滤器（由"\*"键标识）。如果有，我们运行`processGlobal()`，然后取消设置"\*"键。

```php
$toDo = $this->assignments;
if (isset($toDo['*'])) {
  $this->processGlobalAssignment($toDo['*'], $data);
  unset($toDo['*']);
}
```

20. 最后，我们循环处理任何剩余的赋值，调用`processAssignment()`。

```php
foreach ($toDo as $key => $assignment) {
  $this->processAssignment($assignment, $key);
}
```

21. 您还记得，每个赋值都与数据字段有键，并代表该字段的回调数组。因此，在`processGlobalAssignment()`中，我们需要循环浏览回调数组。但在这种情况下，由于这些赋值是全局的，我们还需要循环浏览整个数据集，并依次应用每个全局过滤器。

```php
protected function processGlobalAssignment($assignment, $data)
{
  foreach ($assignment as $callback) {
    if ($callback === NULL) continue;
    foreach ($data as $k => $value) {
      $result = $this->callbacks[$callback['key']]($this->results[$k]->item,
      $callback['params']);
      $this->results[$k]->mergeResults($result);
    }
  }
}
```

{% hint style="info" %}
棘手的是这行代码。

```php
$result = $this->callbacks[$callback['key']]($this ->results[$k]->item, $callback['params']);
```

请记住，每个回调实际上是一个匿名类，定义了PHP神奇的`__invoke()`方法。提供的参数是要过滤的实际数据项和一个参数数组。通过运行 `$this->callbacks[$callback['key']]()`，我们实际上是在神奇地调用 ****`__invoke()`。
{% endhint %}

22.当我们定义`processAssignment()`时，以类似于`processGlobalAssignment()`的方式，我们需要执行分配给每个数据键的剩余回调。

```php
 protected function processAssignment($assignment, $key)
  {
    foreach ($assignment as $callback) {
      if ($callback === NULL) continue;
      $result = $this->callbacks[$callback['key']]($this->results[$key]->item, 
                                 $callback['params']);
      $this->results[$key]->mergeResults($result);
    }
  }
}  // closing brace for Application\Filter\Filter
```

{% hint style="info" %}
重要的是，任何改变用户提供的原始数据的过滤操作都应该显示一条消息，说明已经进行了更改。这可以成为审计跟踪的一部分，以保障您在用户不知情或未经用户同意的情况下进行更改时不会承担潜在的法律责任。
{% endhint %}

## 如何运行...

创建一个`Application\Filter`文件夹。在这个文件夹中，使用前面步骤中的代码，创建以下类文件。

| Application\Filter\\* 类文件 | 这些步骤中描述的代码 |
| :--- | :--- |
| `Result.php` | 3 - 5 |
| `CallbackInterface.php` | 6 |
| `Messages.php` | 7 |
| `AbstractFilter.php` | 8 - 15 |
| `Filter.php` | 16 - 22 |

接下来，使用步骤5中讨论的代码，在`chap_06_post_data_config_messages.php`文件中配置一个消息数组。每个回调都会引用`Messages::$messages`属性。下面是一个配置示例。

```php
<?php
use Application\Filter\Messages;
Messages::setMessages(
  [
    'length_too_short' => 'Length must be at least %d',
    'length_too_long'  => 'Length must be no more than %d',
    'required'         => 'Please be sure to enter a value',
    'alnum'            => 'Only letters and numbers allowed',
    'float'            => 'Only numbers or decimal point',
    'email'            => 'Invalid email address',
    'in_array'         => 'Not found in the list',
    'trim'             => 'Item was trimmed',
    'strip_tags'       => 'Tags were removed from this item',
    'filter_float'     => 'Converted to a decimal number',
    'phone'            => 'Phone number is [+n] nnn-nnn-nnnn',
    'test'             => 'TEST',
    'filter_length'    => 'Reduced to specified length',
  ]
);
```

接下来，创建一个`chap_06_post_data_config_callbacks.php`回调配置文件，其中包含过滤回调的配置，如步骤4所述。每个回调都应该遵循这个通用模板。

```php
'callback_key' => new class () implements CallbackInterface 
{
  public function __invoke($item, $params) : Result
  {
    $changed  = array();
    $filtered = /* perform filtering operation on $item */
    if ($filtered !== $item) $changed = Messages::$messages['callback_key'];
    return new Result($filtered, $changed);
  }
}
```

回调本身必须实现该接口并返回一个 `Result` 实例。我们可以利用 PHP 7 的匿名类功能，让我们的回调返回一个实现 `CallbackInterface`的匿名类。下面是一个过滤回调数组的样子。

```php
use Application\Filter\ { Result, Messages, CallbackInterface };
$config = [ 'filters' => [
  'trim' => new class () implements CallbackInterface 
  {
    public function __invoke($item, $params) : Result
    {
      $changed  = array();
      $filtered = trim($item);
      if ($filtered !== $item) 
      $changed = Messages::$messages['trim'];
      return new Result($filtered, $changed);
    }
  },
  'strip_tags' => new class () 
  implements CallbackInterface 
  {
    public function __invoke($item, $params) : Result
    {
      $changed  = array();
      $filtered = strip_tags($item);
      if ($filtered !== $item)     
      $changed = Messages::$messages['strip_tags'];
      return new Result($filtered, $changed);
    }
  },
  // etc.
]
];
```

为了测试的目的，我们将使用`prospects`表作为目标。我们将不提供来自`$_POST`的数据，而是构建一个好数据和坏数据的数组。

![](../../.gitbook/assets/image%20%2887%29.png)

现在你可以创建一个`chap_06_post_data_filtering.php`脚本，设置自动加载，包括消息和回调配置文件。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
include __DIR__ . '/chap_06_post_data_config_messages.php';
include __DIR__ . '/chap_06_post_data_config_callbacks.php';
```

然后，你需要定义赋值，代表数据字段和过滤器回调之间的映射。使用`*`键来定义一个适用于所有数据的全局过滤器。

```php
$assignments = [
  '*'   => [ ['key' => 'trim', 'params' => []], 
          ['key' => 'strip_tags', 'params' => []] ],
  'first_name'  => [ ['key' => 'length', 
   'params' => ['length' => 128]] ],
  'last_name'  => [ ['key' => 'length', 
   'params' => ['length' => 128]] ],
  'city'          => [ ['key' => 'length', 
   'params' => ['length' => 64]] ],
  'budget'     => [ ['key' => 'filter_float', 'params' => []] ],
];
```

其次，定义好的和坏的测试数据。

```php
$goodData = [
  'first_name'      => 'Your Full',
  'last_name'       => 'Name',
  'address'         => '123 Main Street',
  'city'            => 'San Francisco',
  'state_province'  => 'California',
  'postal_code'     => '94101',
  'phone'           => '+1 415-555-1212',
  'country'         => 'US',
  'email'           => 'your@email.address.com',
  'budget'          => '123.45',
];
$badData = [
  'first_name'      => 'This+Name<script>bad tag</script>Valid!',
  'last_name'       => 'ThisLastNameIsWayTooLongAbcdefghijklmnopqrstuvwxyz0123456789Abcdefghijklmnopqrstuvwxyz0123456789Abcdefghijklmnopqrstuvwxyz0123456789Abcdefghijklmnopqrstuvwxyz0123456789',
  //'address'       => '',    // missing
  'city'            => '  ThisCityNameIsTooLong012345678901234567890123456789012345678901234567890123456789  ',
  //'state_province'=> '',    // missing
  'postal_code'     => '!"£$%^Non Alpha Chars',
  'phone'           => ' 12345 ',
  'country'         => 'XX',
  'email'           => 'this.is@not@an.email',
  'budget'          => 'XXX',
];
```

最后，你可以创建一个`Application\Filter\Filter`实例，并测试数据。

```php
$filter = new Application\Filter\Filter(
$config['filters'], $assignments);
$filter->setSeparator(PHP_EOL);
  $filter->process($goodData);
echo $filter->getMessageString();
  var_dump($filter->getItemsAsArray());

$filter->process($badData);
echo $filter->getMessageString();
var_dump($filter->getItemsAsArray());
```

处理好的数据时，除了一个表示`float`字段的值已从字符串转换为`float`的信息外，不会产生其他信息。而坏数据则会产生以下输出：

![](../../.gitbook/assets/image%20%2886%29.png)

您还会注意到，`first_name`中的标签被删除，`last_name`和`city`都被截断。

## 更多...

`filterinput_array()`函数有两个参数：一个是输入源\(输入源的形式是一个预定义的常量，用来表示`$_*` PHP超级全局之一，即`$_POST`\)，另一个是一个匹配的字段定义的数组，作为键和过滤器或验证器的值。这个函数不仅执行过滤操作，而且还执行验证。标有 sanitize 的标志实际上是过滤器。

## 参见

`filter_input_array()`的文档和例子可以在[http://php.net/manual/en/function.filter-input-array.php](http://php.net/manual/en/function.filter-input-array.php)。你也可以在 [http://php.net/manual/en/filter.filters.php](http://php.net/manual/en/filter.filters.php) 上查看不同类型的过滤器。

