# 过滤$\_POST数据

过滤数据的过程可以包括以下任何或全部内容：

* 删除不需要的字符（即删除`<script>`标签） 
* 对数据进行转换（即，将报价转换为 `&quot;`）
* 对数据进行加密或解密

加密的内容在本章的最后一个示例中介绍。除此之外，我们将介绍一个基本的机制，可以用来过滤表单提交后到达的`$_POST`数据。

## 如何做...

1.首先，你需要对`$_POST`中的数据有一个认识。另外，也许更重要的是，您需要了解存储表格数据的数据库表所施加的限制。举个例子，看看`prospects`表的数据库结构。

```bash
COLUMN          TYPE              NULL   DEFAULT
first_name      varchar(128)      No     None     NULL
last_name       varchar(128)      No     None     NULL
address         varchar(256)      Yes    None     NULL
city            varchar(64)       Yes    None     NULL
state_province  varchar(32)       Yes    None     NULL
postal_code     char(16)          No     None     NULL
phone           varchar(16)       No     None     NULL
country         char(2)           No     None     NULL
email           varchar(250)      No     None     NULL
status          char(8)           Yes    None     NULL
budget          decimal(10,2)     Yes    None     NULL
last_updated    datetime          Yes    None     NULL
```

2. 一旦你完成了对要发布和存储的数据的分析，你就可以确定要进行哪种类型的过滤，哪些PHP函数可以达到这个目的。

3.例如，如果你需要去除用户提供的表单数据中的开头和结尾空白，你可以使用PHP的`trim()`函数。根据数据库结构，所有的字符数据都有长度限制。因此，你可以考虑使用`substr()`来确保长度不被超过。如果你想删除非字母字符，你可以考虑使用`preg_replace()`配合适当的模式来删除。

4. 现在我们可以将所需的PHP函数集归为一个回调数组。下面是一个基于表单数据过滤需求的例子，这些数据最终将被存储在`prospector`表中。

```php
$filter = [
  'trim' => function ($item) { return trim($item); },
  'float' => function ($item) { return (float) $item; },
  'upper' => function ($item) { return strtoupper($item); },
  'email' => function ($item) { 
     return filter_var($item, FILTER_SANITIZE_EMAIL); },
  'alpha' => function ($item) { 
     return preg_replace('/[^A-Za-z]/', '', $item); },
  'alnum' => function ($item) { 
     return preg_replace('/[^0-9A-Za-z ]/', '', $item); },
  'length' => function ($item, $length) { 
     return substr($item, 0, $length); },
  'stripTags' => function ($item) { return strip_tags($item); },
];
```

5. 接下来，我们定义一个与`$_POST`中预期的字段名相匹配的数组。在这个数组中，我们指定了`$filter`数组中的键，以及任何参数。注意第一个键`*`。我们将使用它作为通配符应用于所有字段。

```php
$assignments = [
  '*'             => ['trim' => NULL, 'stripTags' => NULL],
  'first_name'    => ['length' => 32, 'alnum' => NULL],
  'last_name'     => ['length' => 32, 'alnum' => NULL],
  'address'       => ['length' => 64, 'alnum' => NULL],
  'city'          => ['length' => 32],
  'state_province'=> ['length' => 20],
  'postal_code'   => ['length' => 12, 'alnum' => NULL],
  'phone'         => ['length' => 12],
  'country'       => ['length' => 2, 'alpha' => NULL, 
                      'upper' => NULL],
  'email'         => ['length' => 128, 'email' => NULL],
  'budget'        => ['float' => NULL],
];
```

6. 然后，我们循环浏览数据集（即来自`$_POST`），并依次应用回调。我们首先运行所有分配给通配符（`*`）键的回调。

{% hint style="info" %}
实现通配符过滤器以避免多余的设置是很重要的。在前面的例子中，我们希望对每个项目应用代表PHP函数`strip_tags()`和`trim()`的过滤器。
{% endhint %}

7. 接下来，我们运行分配给特定数据字段的所有回调。当我们完成后，`$data`中的所有值将被过滤。

```php
foreach ($data as $field => $item) {
  foreach ($assignments['*'] as $key => $option) {
    $item = $filter[$key]($item, $option);
  }
  foreach ($assignments[$field] as $key => $option) {
    $item = $filter[$key]($item, $option);
  }
}
```

## 如何运行...

将步骤4到步骤6中的代码放入一个名为`chap_12_post_data_filtering_basic.php`的文件中。你还需要定义一个数组来模拟`$_POST`中的数据. 在这种情况下，你可以定义两个数组，一个是好数据，一个是坏数据。

```php
$testData = [
  'goodData'   => [
    'first_name'    => 'Doug',
    'last_name'     => 'Bierer',
    'address'       => '123 Main Street',
    'city'          => 'San Francisco',
    'state_province'=> 'California',
    'postal_code'   => '94101',
    'phone'         => '+1 415-555-1212',
    'country'       => 'US',
    'email'         => 'doug@unlikelysource.com',
    'budget'        => '123.45',
  ],
  'badData' => [
    'first_name' => 'This+Name<script>bad tag</script>Valid!',
    'last_name' 	=> 'ThisLastNameIsWayTooLongAbcdefghijklmnopqrstuvwxyz0123456789Abcdefghijklmnopqrstuvwxyz0123456789Abcdefghijklmnopqrstuvwxyz0123456789Abcdefghijklmnopqrstuvwxyz0123456789',
    //'address' 	=> '',    // missing
    'city'      => 'ThisCityNameIsTooLong012345678901234567890123456789012345678901234567890123456789  ',
    //'state_province'=> '',    // missing
    'postal_code'     => '!"£$%^Non Alpha Chars',
    'phone'           => ' 12345 ',
    'country'         => '12345',
    'email'           => 'this.is@not@an.email',
    'budget'          => 'XXX',
  ]
];
```

最后，你需要循环过滤分配，呈现好的和坏的数据。

```php
foreach ($testData as $data) {
  foreach ($data as $field => $item) {
    foreach ($assignments['*'] as $key => $option) {
      $item = $filter[$key]($item, $option);
    }
    foreach ($assignments[$field] as $key => $option) {
      $item = $filter[$key]($item, $option);
    }
    printf("%16s : %s\n", $field, $item);
  }
}
```

下面是这个例子的输出结果。

![](../../.gitbook/assets/image%20%28143%29.png)

请注意，这些名字被截断，标签被删除。您还会注意到，虽然电子邮件地址被过滤，但它仍然不是一个有效的地址。需要注意的是，为了正确处理数据，可能需要验证和过滤。



