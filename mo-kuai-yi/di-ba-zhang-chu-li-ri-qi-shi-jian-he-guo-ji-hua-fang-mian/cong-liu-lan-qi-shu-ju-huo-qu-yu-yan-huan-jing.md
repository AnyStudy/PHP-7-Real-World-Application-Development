# 从浏览器数据获取语言环境

为了改善用户在网站上的体验，以用户所在地区可以接受的格式显示信息是很重要的。本地化是一个通用术语，用来表示世界上的一个区域。I.T.社区已经做出了努力，使用一个由语言和国家代码组成的两部分指定来对地域进行编码。但是，当一个人访问你的网站时，你如何知道他们的地域？最有用的技术可能是检查HTTP语言头。

## 如何做...

1.为了封装locale功能，我们将假设一个类，`Application\I18n\Locale`。我们将让这个类扩展一个现有的类`Locale`，它是PHP `Intl`扩展的一部分。

{% hint style="info" %}
I18n是Internationalization的常见缩写。\(数一数字母的数量！\)
{% endhint %}

```php
namespace Application\I18n;
use Locale as PhpLocale;
class Locale extends PhpLocale
{
  const FALLBACK_LOCALE = 'en';
  // some code
}
```

2. 要了解一个传入的请求是什么样的，可以使用`phpinfo(INFO_VARIABLES)`。在测试后一定要立即关闭这个函数，因为它向潜在的攻击者提供了太多的信息。

```php
<?php phpinfo(INFO_VARIABLES); ?>
```

3. 本地化信息存储在`$_SERVER['HTTP_ACCEPT_LANGUAGE']`中。该值将采取这种一般形式：`ll-CC,rl;q=0.n，ll-CC,rl;q=0.n`，如本表所定义。

| 简称 | 定义 |
| :--- | :--- |
| `ll` | 代表语言的两个字符的小写代码。 |
| `-` | 在地区代码`ll-CC`中把语言和国家分开。 |
| `CC` | 代表国家的两个大写字母代码。 |
| `,` | 分离区域代码和后备根区域代码（通常与语言代码相同）。 |
| `rl` | 两个字符的小写代码，代表建议的root locale。 |
| `;` | 将locale信息与quality分开。如果质量缺失，默认为q=1 \(100%\)概率，这是首选。 |
| `q` | 质量。 |
| `0.n` | 在0.00和1.0之间的某个值。将该值乘以100，得到该游客实际喜欢的语言的概率百分比。 |

4. 可以很容易地列出一个以上的地区。例如，网站访问者可能在他们的计算机上安装了多种语言。碰巧PHP `Locale`类有一个方法`acceptFromHttp()`，它读取`Accept-language`头字符串，并给我们提供所需的设置。

```php
protected $localeCode;
public function setLocaleCode($acceptLangHeader)
{
  $this->localeCode = $this->acceptFromHttp($acceptLangHeader);
}
```

5. 然后我们可以定义相应的getter。`get AcceptLanguage()`方法从`$_SERVER['HTTP_ACCEPT_LANGUAGE']`返回值。

```php
public function getAcceptLanguage()
{
  return $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? self::FALLBACK_LOCALE;
}
public function getLocaleCode()
{
  return $this->localeCode;
}
```

6. 接下来我们定义一个构造函数，允许我们 "手动 "设置locale。否则，本地语言信息将从浏览器中获取。

```php
public function __construct($localeString = NULL)
{
  if ($localeString) {
    $this->setLocaleCode($localeString);
  } else {
    $this->setLocaleCode($this->getAcceptLanguage());
  }
}
```

7. 现在是一个重大的决定：如何处理这些信息! 这在接下来的几个事例中会有介绍。

{% hint style="info" %}
即使访问者看起来接受一种或多种语言，但该访问者并不一定希望内容使用他们的浏览器所指示的语言/语言。因此，虽然您可以根据这些信息设置语言，但您也应该为他们提供一个静态的替代语言列表。
{% endhint %}

## 如何运行...

在这个说明中，我们举三个例子。

* 来自浏览器的信息 
* 预设区域  `fr-FR`
*  取自`RFC 2616`的字符串：`da, en-gb;q=0.8, en;q=0.7`。

将步骤1到6的代码放入一个文件`Locale.php`中，这个文件在`Application\I18n`文件夹中。

接下来，创建一个文件，`chap_08_getting_locale_from_browser.php`，设置自动加载并使用新的类。

```php
<?php
  require __DIR__ . '/../Application/Autoload/Loader.php';
  Application\Autoload\Loader::init(__DIR__ . '/..');
  use Application\I18n\Locale;
```

现在你可以定义一个包含三个测试locale字符串的数组。

```php
$locale = [NULL, 'fr-FR', 'da, en-gb;q=0.8, en;q=0.7'];
```

最后，在三个locale字符串中循环，创建新类的实例。回传从`getLocaleCode()`返回的值，看看做出了什么选择。

```php
echo '<table>';
foreach ($locale as $code) {
  $locale = new Locale($code); 
  echo '<tr>
    <td>' . htmlspecialchars($code) . '</td>
    <td>' . $locale->getLocaleCode() . '</td>
  </tr>';
}
echo '</table>';
```

这是结果（稍微改了下输出格式）。

![](../../.gitbook/assets/image%20%28105%29.png)

## 更多...

* 关于PHP `Locale`类的信息，参见[http://php.net/manual/en/class.locale.php](http://php.net/manual/en/class.locale.php)。 
* 有关`Accept-Language`头的更多信息，请参见RFC 2616的14.4节：[https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)。

