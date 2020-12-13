# 构建对象到数组到转化器

这个案例与前一个刚好相反。在这种情况下，我们需要从对象属性中提取值，并返回一个关联数组，其中键是列名。

## 如何做...

1.在这个例子中，我们将建立在前面的案例中定义的`Application\Generic\Hydrator\GetSet`类的基础上。

```php
namespace Application\Generic\Hydrator;
class GetSet
{
  // code
}
```

2.在前面的配方中定义了`hydrate()`方法之后，我们定义了一个`extract()`方法，它以一个对象作为参数。其逻辑与`hydrate()`方法类似，只是这次我们搜索的是`getXXX()`方法。再次使用 `preg_match()` 来匹配方法的前缀和后缀，后缀被认为是数组键。

```php
public static function extract($object)
{
  $array = array();
  $class = get_class($object);
  $methodList = get_class_methods($class);
  foreach ($methodList as $method) {
    preg_match('/^(get)(.*?)$/i', $method, $matches);
    $prefix = $matches[1] ?? '';
    $key    = $matches[2] ?? '';
    $key    = strtolower(substr($key, 0, 1)) . substr($key, 1);
    if ($prefix == 'get') {
      $array[$key] = $object->$method();
    }
  }
  return $array;
}
}
```

{% hint style="info" %}
请注意，为了方便起见，我们将 `hydrate()` 和 `extract()` 定义为静态方法。
{% endhint %}

## 如何运行...

定义一个名为`chap_11_object_to_array.php`的调用程序，设置自动加载，并使用相应的类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Entity\Person;
use Application\Generic\Hydrator\GetSet;
```

接下来，定义一个 `Person` 的实例，为其属性设置值。

```php
$obj = new Person();
$obj->setFirstName('Li\'lAbner');
$obj->setLastName('Yokum');
$obj->setAddress('1DirtStreet');
$obj->setCity('Dogpatch');
$obj->setStateProv('Kentucky');
$obj->setPostalCode('12345');
$obj->setCountry('USA');
```

最后，以静态方式调用 new `extract()`方法。

```php
$a = GetSet::extract($obj);
var_dump($a);
```

输出结果如下截图所示。

![](../../.gitbook/assets/image%20%28134%29.png)

