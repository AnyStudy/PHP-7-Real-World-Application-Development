# 验证$\_POST数据

过滤和验证的主要区别在于后者不改变原始数据。另一个区别在于意图。验证的目的是确认数据是否符合根据客户需求建立的某些标准。

## 如何做...

1.我们将在这里介绍的基本验证机制与前面的事例中所示的相同。与过滤一样，了解要验证的数据的性质，它如何符合客户的要求，以及它是否符合数据库执行的标准是至关重要的。例如，如果在数据库中，列的最大宽度是128，那么验证回调可以使用`strlen()`来确认提交的数据长度是否小于或等于128个字符。同样，可以根据情况使用`ctype_alnum()`来确认数据只包含字母和数字。

2. 验证的另一个考虑因素是呈现一个适当的验证失败信息。验证过程在一定意义上也是一个确认过程，大概会有人对验证进行审核，确认成功或失败。如果验证失败，这个人需要知道原因。

3. 在这个例子中，我们将再次关注`prospects`表。现在我们可以将所需的PHP函数集归为一个回调数组。下面是一个基于表单数据验证需求的例子，表单数据最终将被存储在`prospects`表中。

```php
$validator = [
  'email' => [
    'callback' => function ($item) { 
      return filter_var($item, FILTER_VALIDATE_EMAIL); },
    'message'  => 'Invalid email address'],
  'alpha' => [
    'callback' => function ($item) { 
      return ctype_alpha(str_replace(' ', '', $item)); },
    'message'  => 'Data contains non-alpha characters'],
  'alnum' => [
    'callback' => function ($item) { 
      return ctype_alnum(str_replace(' ', '', $item)); },
    'message'  => 'Data contains characters which are '
       . 'not letters or numbers'],
  'digits' => [
    'callback' => function ($item) { 
      return preg_match('/[^0-9.]/', $item); },
    'message'  => 'Data contains characters which '
      . 'are not numbers'],
  'length' => [
    'callback' => function ($item, $length) { 
      return strlen($item) <= $length; },
    'message'  => 'Item has too many characters'],
  'upper' => [
    'callback' => function ($item) { 
      return $item == strtoupper($item); },
    'message'  => 'Item is not upper case'],
  'phone' => [
    'callback' => function ($item) { 
      return preg_match('/[^0-9() -+]/', $item); },
    'message'  => 'Item is not a valid phone number'],
];
```

{% hint style="info" %}
请注意，对于alpha和alnum回调，我们首先使用`str_replace()`删除空白字符。然后我们可以调用`ctype_alpha()`或`ctype_alnum()`，这将决定是否存在任何不允许的字符。
{% endhint %}

4. 接下来，我们定义一个与`$_POST`中的字段名相匹配的赋值数组。在这个数组中，我们指定了`$validator`数组中的key，以及任何参数。

```php
$assignments = [
  'first_name'    => ['length' => 32, 'alpha' => NULL],
  'last_name'     => ['length' => 32, 'alpha' => NULL],
  'address'       => ['length' => 64, 'alnum' => NULL],
  'city'          => ['length' => 32, 'alnum' => NULL],
  'state_province'=> ['length' => 20, 'alpha' => NULL],
  'postal_code'   => ['length' => 12, 'alnum' => NULL],
  'phone'         => ['length' => 12, 'phone' => NULL],
  'country'       => ['length' => 2, 'alpha' => NULL, 
                      'upper' => NULL],
  'email'         => ['length' => 128, 'email' => NULL],
  'budget'        => ['digits' => NULL],
];
```

5. 然后，我们使用嵌套的`foreach()`循环，一次一个字段地迭代数据块。对于每个字段，我们都会循环执行分配给该字段的回调。

```php
foreach ($data as $field => $item) {
  echo 'Processing: ' . $field . PHP_EOL;
  foreach ($assignments[$field] as $key => $option) {
    if ($validator[$key]['callback']($item, $option)) {
        $message = 'OK';
    } else {
        $message = $validator[$key]['message'];
    }
    printf('%8s : %s' . PHP_EOL, $key, $message);
  }
}
```

{% hint style="info" %}
与其直接呼应输出，不如如图所示，你可以将验证成功/失败的信息记录下来，以便在稍后的时间提交给审核者。另外，如第6章 "建立可扩展网站" 中所示，你可以将验证机制工作到表单中，在其匹配的表单元素旁边显示验证消息。
{% endhint %}

## 如何运行...

将步骤3到步骤5中的代码放入一个名为`chap_12_post_data_validation_basic.php`的文件中。你还需要定义一个数据数组来模拟`$_POST`中的数据。在这种情况下，你需要使用前面提到的两个数组，一个是好数据，一个是坏数据。最后的输出应该是这样的。

![](../../.gitbook/assets/image%20%28180%29.png)

## 更多...

在第6章 "构建可扩展网站 "中，题为 "链式$\_POST验证器 "的事例讨论了如何将这里所涵盖的基本验证概念融入到一个全面的过滤器链式机制中。

