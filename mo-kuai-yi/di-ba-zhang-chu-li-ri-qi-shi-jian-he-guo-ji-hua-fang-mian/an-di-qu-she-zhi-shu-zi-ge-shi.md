# 按地区设置数字格式

数字表示形式可能会因地区而异。 举一个简单的例子，在英国，人们看到的数字如下所示：

```text
3,080,512.92
```

但是，在法国，相同的数字可能如下所示：

```text
3 080 512,92
```

## 如何做...

在你以特定于本地的方式表示一个数字之前，你需要确定本地。这可以使用前面的配方中讨论的`Application\I18n\Locale`类来完成。本地化可以手动设置或从头信息中设置。

1.接下来，我们将利用`NumberFormatter`类的`format()`方法，以本地特有的格式输出和解析数字。首先我们添加一个属性，该属性将包含`NumberFormatter`类的实例。

```php
use NumberFormatter;
protected $numberFormatter;
```

{% hint style="info" %}
我们最初的想法是考虑使用PHP函数`setlocale()`来产生根据locale格式化的数字。然而，这种传统方法的问题是，所有的东西都将基于这个`locale`来考虑。这可能会引入处理根据数据库规范存储的数据的问题。`setlocale()`的另一个问题是，它是基于过时的标准，包括RFC 1766和ISO 639。最后，`setlocale()`高度依赖于操作系统的locale支持，这将使我们的代码不可移植。
{% endhint %}

2. 通常，下一步是在构造函数中设置`$numberFormatter`。对于我们的`Application\I18n\Locale`类来说，这种方法的问题是，我们最终会得到一个上重下轻的类，因为我们还需要执行货币和日期格式化。相应地，我们添加了一个`getter`，首先检查`NumberFormatter`的实例是否已经被创建。如果没有，则会创建一个实例并返回。new`NumberFormatter`的第一个参数是`locale`代码。第二个参数`NumberFormatter::DECIMAL`代表我们需要的格式化类型。

```php
public function getNumberFormatter()
{
  if (!$this->numberFormatter) {
    $this->numberFormatter = new NumberFormatter($this->getLocaleCode(), NumberFormatter::DECIMAL);
  }
  return $this->numberFormatter;
}
```

3. 然后，我们添加了一个方法，给定任何数字，将产生一个代表该数字的字符串，并根据当地语言进行格式化。

```php
public function formatNumber($number)
{
  return $this->getNumberFormatter()->format($number);
}
```

4. 接下来我们添加了一个方法，它可以用来根据本地语言解析数字，生成一个本地的PHP数值。请注意，根据服务器的ICU版本，解析失败时结果可能不会返回`FALSE`。

```php
public function parseNumber($string)
{
  $result = $this->getNumberFormatter()->parse($string);
  return ($result) ? $result : self::ERROR_UNABLE_TO_PARSE;
}
```

## 如何运行...

在`Application\I18n\Locale`类中添加前面的要点。然后你可以创建一个`chap_08_formatting_numbers.php`文件，该文件设置了自动加载并使用这个类。

```php
<?php
  require __DIR__ . '/../Application/Autoload/Loader.php';
  Application\Autoload\Loader::init(__DIR__ . '/..');
  use Application\I18n\Locale;
```

在这个例子中，创建两个`Locale`实例，一个用于英国，另一个用于法国。你也可以指定一个大的数量来用于测试。

```php
  $localeFr = new Locale('fr_FR');
  $localeUk = new Locale('en_GB');
  $number   = 1234567.89;
?>
```

最后，您可以将`formatNumber()`和`parseNumber()`方法包装在相应的HTML显示逻辑中，并查看结果。

```php
<!DOCTYPE html>
<html>
  <head>
    <title>PHP 7 Cookbook</title>
    <meta http-equiv="content-type" content="text/html;charset=utf-8" />
    <link rel="stylesheet" type="text/css" href="php7cookbook_html_table.css">
  </head>
  <body>
    <table>
      <tr>
        <th>Number</th>
        <td>1234567.89</td>
      </tr>
      <tr>
        <th>French Format</th>
        <td><?= $localeFr->formatNumber($number); ?></td>
      </tr>
      <tr>
        <th>UK Format</th>
        <td><?= $localeUk->formatNumber($number); ?></td>
      </tr>
      <tr>
        <th>UK Parse French Number: <?= $localeFr->formatNumber($number) ?></th>
        <td><?= $localeUk->parseNumber($localeFr->formatNumber($number)); ?></td>
      </tr>
      <tr>
        <th>UK Parse UK Number: <?= $localeUk->formatNumber($number) ?></th>
        <td><?= $localeUk->parseNumber($localeUk->formatNumber($number)); ?></td>
      </tr>
      <tr>
        <th>FR Parse FR Number: <?= $localeFr->formatNumber($number) ?></th>
        <td><?= $localeFr->parseNumber($localeFr->formatNumber($number)); ?></td>
      </tr>
      <tr>
        <th>FR Parse UK Number: <?= $localeUk->formatNumber($number) ?></th>
        <td><?= $localeFr->parseNumber($localeUk->formatNumber($number)); ?></td>
      </tr>
    </table>
  </body>
</html>
```

这是在浏览器上看到的结果。

![](../../.gitbook/assets/image%20%28109%29.png)

{% hint style="info" %}
请注意，如果locale被设置为`fr_FR`，一个英国格式的数字，在解析时不会返回正确的值。同样的，当locale被设置为`en_GB`时，一个法国格式的数字在解析时也不会返回正确的值。因此，您可能需要考虑在尝试解析数字之前添加一个验证检查。
{% endhint %}

## 更多...

* 关于`setlocale()`的使用和滥用的更多信息，请参考这个页面：[http://php.net/manual/en/function.setlocale.php](http://php.net/manual/en/function.setlocale.php)。
* 关于为什么数字格式化在某些服务器上会产生错误，而在其他服务器上不会，请查看ICU（International Components for Unicode）版本。参见本页的评论：[http://php.net/manual/en/numberformatter.parse.php](http://php.net/manual/en/numberformatter.parse.php)。关于ICU格式化的更多信息，请参阅[http://userguide.icu-project.org/formatparse](http://userguide.icu-project.org/formatparse)。

