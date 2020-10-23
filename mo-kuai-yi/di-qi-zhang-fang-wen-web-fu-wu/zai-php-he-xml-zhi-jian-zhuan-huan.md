# 在PHP和XML之间转换

当考虑PHP本地数据类型和XML之间的转换时，我们通常会将数组作为主要目标。考虑到这一点，从PHP数组到XML的转换过程与反过来的转换过程截然不同。

{% hint style="info" %}
对象也可以考虑转换；但是，在XML中很难呈现对象方法。不过，可以通过使用`get_object_vars()`函数来表示属性，该函数将对象属性读入一个数组。
{% endhint %}

## 如何做...

1.首先，我们定义一个`Application\Parse\ConvertXm`l类。这个类将持有将XML转换为PHP数组的方法，反之亦然。我们将需要SPL中的`SimpleXMLElement`和`SimpleXMLIterator`类。

```php
namespace Application\Parse;
use SimpleXMLIterator;
use SimpleXMLElement;
class ConvertXml
{
}
```

2. 接下来，我们定义一个`xmlToArray()`方法，它将接受一个SimpleXMLIterator实例作为参数。它将被递归调用，并从一个XML文档中产生一个PHP数组。我们利用`SimpleXMLIterator`的能力在XML文档中前进，使用`key()`、`current()`、`next()`和`rewind()`方法来导航。

```php
public function xmlToArray(SimpleXMLIterator $xml) : array
{
  $a = array();
  for( $xml->rewind(); $xml->valid(); $xml->next() ) {
    if(!array_key_exists($xml->key(), $a)) {
      $a[$xml->key()] = array();
    }
    if($xml->hasChildren()){
      $a[$xml->key()][] = $this->xmlToArray($xml->current());
    }
    else{
      $a[$xml->key()] = (array) $xml->current()->attributes();
      $a[$xml->key()]['value'] = strval($xml->current());
    }
  }
  return $a;
}
```

3. 对于反向过程，也叫递归，我们定义了两个方法。第一个方法`arrayToXml()`，设置一个初始`SimpleXMLElement`实例，然后调用第二个方法\`phpToXml\(\)。

```php
public function arrayToXml(array $a)
{
  $xml = new SimpleXMLElement(
  '<?xml version="1.0" standalone="yes"?><root></root>');
  $this->phpToXml($a, $xml);
  return $xml->asXML();
}
```

4. 请注意，在第二个方法中，我们使用`get_object_vars()`来处理数组元素中的一个对象。你还会注意到，单独的数字是不允许作为XML标签的，这意味着要在数字前面添加一些文本。

```php
protected function phpToXml($value, &$xml)
{
  $node = $value;
  if (is_object($node)) {
    $node = get_object_vars($node);
  }
  if (is_array($node)) {
    foreach ($node as $k => $v) {
      if (is_numeric($k)) {
        $k = 'number' . $k;
      }
      if (is_array($v)) {
          $newNode = $xml->addChild($k);
          $this->phpToXml($v, $newNode);
      } elseif (is_object($v)) {
          $newNode = $xml->addChild($k);
          $this->phpToXml($v, $newNode);
      } else {
          $xml->addChild($k, $v);
      }
    }
  } else  {
      $xml->addChild(self::UNKNOWN_KEY, $node);
  }
}
```

## 如何运行...

作为 XML 文档的示例，你可以使用美国国家气象局的 Web 服务定义语言（WSDL）。这是一个描述 SOAP 服务的 XML 文档，可以在 [http://graphical.weather.gov/xml/SOAP\_server/ndfdXMLserver.php?wsdl](http://graphical.weather.gov/xml/SOAP_server/ndfdXMLserver.php?wsdl) 找到。

我们将使用`SimpleXMLIterator`类来提供一个迭代机制，然后你可以配置自动加载，并得到一个`ApplicationParseConvertXml`的实例。使用`xmlToArray()`将WSDL转换为PHP数组。

```php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Parse\ConvertXml;
$wsdl = 'http://graphical.weather.gov/xml/'
. 'SOAP_server/ndfdXMLserver.php?wsdl';
$xml = new SimpleXMLIterator($wsdl, 0, TRUE);
$convert = new ConvertXml();
var_dump($convert->xmlToArray($xml));
```

由此产生的数组如图所示。

![](../../.gitbook/assets/image%20%2891%29.png)

要做相反的事情，使用本配方中描述的`arrayToXml()`方法。作为源文件，你可以使用`source/data/mongo.db.global.php`文件，该文件包含了一个通过O'Reilly Media提供的MongoDB培训视频的大纲（免责声明：由本作者提供！）。使用相同的自动加载器配置和`Application\Parse\ConvertXml`实例，这里是你可以使用的示例代码。

```php
$convert = new ConvertXml();
header('Content-Type: text/xml');
echo $convert->arrayToXml(include CONFIG_FILE);
```

这是在浏览器中的输出。

![](../../.gitbook/assets/image%20%2893%29.png)

