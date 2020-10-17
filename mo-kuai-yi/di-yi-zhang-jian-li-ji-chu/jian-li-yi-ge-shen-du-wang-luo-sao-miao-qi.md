# 建立一个深度网络扫描器

有时您需要扫描一个网站，但是要更深一层。例如，你想建立一个网站的网络树图。这可以通过寻找所有 `<A>` 标签，并跟随 `HREF` 属性到下一个网页来实现。一旦你获得了子页面，你就可以继续扫描，以便完成树状图。

## 如何做...

1.如前所述，深度网络扫描器的核心组件是基本的 `Hoover` 类。本示例中介绍的基本过程是扫描目标网站并收集所有 `HREF` 属性。为此，我们定义了 `Application\Web\Deep` 类。 我们添加一个代表 DNS 域的属性：

```php
namespace Application\Web;
class Deep
{
    protected $domain;
```

2.接下来，我们定义了一个方法，该方法将对扫描列表中代表的每个网站的标签进行抓取。为了防止扫描器搜索整个**万维网**（**WWW**），我们将扫描范围限制在目标域。之所以加入了`yield from` ，是因为我们需要输出由 `Hoover::getTags()` 产生的整个数组。`yield from` 语法允许我们将数组视为一个子生成器：

```php
public function scan($url, $tag)
{
    $vac    = new Hoover();
    $scan   = $vac->getAttribute($url, 'href', 
       $this->getDomain($url));
    $result = array();
    foreach ($scan as $subSite) {
        yield from $vac->getTags($subSite, $tag);
    }
    return count($scan);
}
```

{% hint style="info" %}
使用 `yield from` 可以将 `scan()` 方法变成一个PHP 7的委托生成器。通常情况下，你会倾向于将扫描的结果存储到一个数组中。问题是，在这种情况下，检索到的信息量可能是巨大的。因此，为了节省内存和立即产生结果，最好的方式是使用 `yield`。否则，就会有一个漫长的等待，这很可能会出现内存不足的错误。
{% endhint %}

3.为了保持在同一个域内，我们需要一个方法来从URL中返回域。我们使用方便的 `parse_url()` 函数来实现这一目的：

```php
public function getDomain($url)
{
    if (!$this->domain) {
        $this->domain = parse_url($url, PHP_URL_HOST);
    }
    return $this->domain;
}
```

## 如何运行...

首先，先去定义前面定义的 `Application\Web\Deep` 类，以及前面配方中定义的 `Application\Web\Hoover` 类。

接下来，在 `chap_01_deep_scan_website.php`中定义一个代码块来设置自动加载\(如本章前面所述\)：

```php
<?php
// 根据需要修改
define('DEFAULT_URL', 'unlikelysource.com');
define('DEFAULT_TAG', 'img');

require __DIR__ . '/../../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/../..');
```

接下来，获取新类的一个实例：

```php
$deep = new Application\Web\Deep();
```

此时，你可以从 URL 参数中检索 URL 和标签信息。PHP 7的null 合并运算符对于建立回退值非常有用：

```php
$url = strip_tags($_GET['url'] ?? DEFAULT_URL);
$tag = strip_tags($_GET['tag'] ?? DEFAULT_TAG);
```

一些简单的HTML将显示结果：

```php
foreach ($deep->scan($url, $tag) as $item) {
    $src = $item['attributes']['src'] ?? NULL;
    if ($src && (stripos($src, 'png') || stripos($src, 'jpg'))) {
        printf('<br><img src="%s"/>', $src);
    }
}
```

## 参考

关于生成器和`yield from`的更多信息，请参阅 [http://php.net/manual/en/language.generators.syntax.php](http://php.net/manual/en/language.generators.syntax.php)

