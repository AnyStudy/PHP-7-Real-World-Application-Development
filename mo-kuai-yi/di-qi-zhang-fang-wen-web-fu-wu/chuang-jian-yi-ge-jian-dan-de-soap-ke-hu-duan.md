# 创建一个简单的SOAP客户端

与实现REST客户端或服务器的过程相比，使用SOAP是非常容易的，因为有一个PHP SOAP扩展提供了这两种功能。

{% hint style="info" %}
一个经常被问到的问题是 "SOAP和REST之间有什么区别？" SOAP内部使用XML作为其数据格式。SOAP使用HTTP，但只用于传输，除此之外对其他HTTP方法无法进行识别。REST直接操作HTTP，可以使用任何数据格式，但首选JSON。另一个关键的区别是，SOAP可以与WSDL一起操作，这使得服务自我描述，从而更加公开。因此，SOAP服务通常由国家卫生组织等公共机构提供。
{% endhint %}

## 如何做...

在这个例子中，我们将对美国国家气象服务提供的现有SOAP服务进行SOAP请求。

1.首先要考虑的是识别WSDL文档。WSDL是描述服务的XML文档。

```php
$wsdl = 'http://graphical.weather.gov/xml/SOAP_server/'
  . 'ndfdXMLserver.php?wsdl';
```

2.接下来，我们使用 WSDL 创建一个 soap 客户端实例。

```php
$soap = new SoapClient($wsdl, array('trace' => TRUE));
```

3.然后，我们可以自由地初始化一些变量，以应对天气预报的请求。

```php
$units = 'm';
$params = '';
$numDays = 7;
$weather = '';
$format = '24 hourly';
$startTime = new DateTime();
```

4. 然后，我们可以发出`LatLonListCityNames()`SOAP请求，在WSDL中被标识为一个操作，以获取服务支持的城市列表。该请求以 XML 格式返回，建议创建一个 `SimpleXLMElement` 实例。

```php
$xml = new SimpleXMLElement($soap->LatLonListCityNames(1));
```

5. 不幸的是，城市列表和它们对应的经纬度是在单独的XML节点中。因此，我们使用`array_combine()`PHP函数来创建一个关联数组，其中经纬度是键，城市名称是值。然后我们可以使用这个数组来呈现一个HTML `SELECT`下拉列表，使用`asort()`来对列表进行字母排序。

```php
$cityNames = explode('|', $xml->cityNameList);
$latLonCity = explode(' ', $xml->latLonList);
$cityLatLon = array_combine($latLonCity, $cityNames);
asort($cityLatLon);
```

6. 然后，我们可以从网络请求中获得城市数据，如下所示。

```php
$currentLatLon = (isset($_GET['city'])) ? strip_tags(urldecode($_GET['city'])) : '';
```

7.我们希望进行的 SOAP 调用是 `NDFDgenByDay()`。我们可以通过检查 WSDL 来确定提供给 SOAP 服务器的参数的性质。

```php
<message name="NDFDgenByDayRequest">
<part name="latitude" type="xsd:decimal"/>
<part name="longitude" type="xsd:decimal"/>
<part name="startDate" type="xsd:date"/>
<part name="numDays" type="xsd:integer"/>
<part name="Unit" type="xsd:string"/>
<part name="format" type="xsd:string"/>
</message>
```

8. 如果`$currentLatLon`的值被设置，我们就可以处理这个请求。我们用 `try {} catch {}` 块来处理请求，以防出现异常。

```php
if ($currentLatLon) {
  list($lat, $lon) = explode(',', $currentLatLon);
  try {
      $weather = $soap->NDFDgenByDay($lat,$lon,
        $startTime->format('Y-m-d'),$numDays,$unit,$format);
  } catch (Exception $e) {
      $weather .= PHP_EOL;
      $weather .= 'Latitude: ' . $lat . ' | Longitude: ' . $lon;
      $weather .= 'ERROR' . PHP_EOL;
      $weather .= $e->getMessage() . PHP_EOL;
      $weather .= $soap->__getLastResponse() . PHP_EOL;
  }
}
?>
```

## 如何运行...

将前面所有的代码复制到`chap_07_simple_soap_client_weather_service.php`文件中。然后你可以添加视图逻辑来显示城市列表的表单以及结果。

```php
<form method="get" name="forecast">
<br> City List: 
<select name="city">
<?php foreach ($cityLatLon as $latLon => $city) : ?>
<?php $select = ($currentLatLon == $latLon) ? ' selected' : ''; ?>
<option value="<?= urlencode($latLon) ?>" <?= $select ?>>
<?= $city ?></option>
<?php endforeach; ?>
</select>
<br><input type="submit" value="OK"></td>
</form>
<pre>
<?php var_dump($weather); ?>
</pre>
```

这是在浏览器中请求俄亥俄州克利夫兰市天气预报的结果。

![](../../.gitbook/assets/image%20%28107%29.png)

## 参考

关于SOAP和REST之间的区别，请参考[http://stackoverflow.com/questions/209905/representational-state-transfer-rest-and-simple-object-access-protocol-soap?lq=1](http://stackoverflow.com/questions/209905/representational-state-transfer-rest-and-simple-object-access-protocol-soap?lq=1)。

