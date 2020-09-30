# 抓取一个网站

很多时候，抓取一个网站并从特定的标签中提取信息是非常有价值的。这种基本的机制可以用来在网络上搜索有价值的信息。在其他时候，你可能需要得到一个 `<IMG>` 标签和 `SRC` 属性的列表，或者 `<A>` 标签和相应的 `HREF` 属性。这种可能性是无穷无尽的。

## 怎么做...

1.首先，我们需要抓取目标网站的内容。乍一看，似乎我们应该发出 cURL 请求，或者干脆使用 `file_get_contents()` 。这些方法的问题是，我们最终将不得不进行大量的字符串操作，很可能不得不过度使用可怕的正则表达式。为了避免这一切，我们将简单地利用已经存在的 PHP 7 类  `DOMDocument` 。所以我们创建一个 `DOMDocument` 实例，并将其设置为**UTF-8**。我们不关心空格，使用方便的 `loadHTMLFile()`  方法将网站的内容加载到对象中：

```php
public function getContent($url)
{
    if (!$this->content) {
        if (stripos($url, 'http') !== 0) {
            $url = 'http://' . $url;
        }
        $this->content = new DOMDocument('1.0', 'utf-8');
        $this->content->preserveWhiteSpace = FALSE;
        // @ used to suppress warnings generated from // improperly configured web pages
        @$this->content->loadHTMLFile($url);
    }
    return $this->content;
}
```

> **小贴士**
>
> 请注意，我们在调用 `loadHTMLFile()` 方法之前加上了 `@`。这并不是为了掩盖错误的编码\(`!`\)，就像在PHP 5中经常发生的那样！而是当解析器遇到写得不好的HTML时，`@` 会抑制产生的通知。相反，当解析器遇到写得不好的HTML时，`@` 会抑制产生的通知。大概我们可以捕获这些通知并记录下来，可能会给我们的 `Hoover` 类提供一个诊断功能。

2.接下来，我们需要提取感兴趣的标签。我们使用 `getElementsByTagName()` 方法来实现这一目的。如果我们希望提取所有标记，我们可以提供  `*` 作为参数：

```php
public function getTags($url, $tag)
{
    $count    = 0;
    $result   = array();
    $elements = $this->getContent($url)
                     ->getElementsByTagName($tag);
    foreach ($elements as $node) {
        $result[$count]['value'] = trim(preg_replace('/\s+/', ' ', $node->nodeValue));
        if ($node->hasAttributes()) {
            foreach ($node->attributes as $name => $attr) 
            {
                $result[$count]['attributes'][$name] = 
                    $attr->value;
            }
        }
        $count++;
    }
    return $result;
}
```

3.提取某些属性而不是标签也可能是有意义的。因此，我们为此定义了另一个方法。在这种情况下，我们需要解析所有的标签并使用 `getAttribute()` 。你会注意到，有一个DNS域的参数。我们添加这个参数是为了使扫描保持在同一个域内（例如，如果你正在构建一个网络树）：

```php
public function getAttribute($url, $attr, $domain = NULL)
{
    $result   = array();
    $elements = $this->getContent($url)
                     ->getElementsByTagName('*');
    foreach ($elements as $node) {
        if ($node->hasAttribute($attr)) {
            $value = $node->getAttribute($attr);
            if ($domain) {
                if (stripos($value, $domain) !== FALSE) {
                    $result[] = trim($value);
                }
            } else {
                $result[] = trim($value);
            }
        }
    }
    return $result;
}
```

## 它是如何工作的...

为了使用新的 `Hoover` 类，请初始化自动加载器（如前所述）并创建 `Hoover` 类的实例。 然后，您可以运行 `Hoover::getTags()` 方法，从指定为参数的URL生成标签数组。

下面是 `chap_01_vacuuming_website.php` 中的一段代码，它使用 `Hoover` 类来扫描O'Reilly网站的 `<A>` 标签：

```php
<?php
// 根据需要修改
define('DEFAULT_URL', 'http://oreilly.com/');
define('DEFAULT_TAG', 'a');

require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');

// 获取 "vacuum" 类
$vac = new Application\Web\Hoover();

// N注意：使用PHP 7 null 合并运算符
$url = strip_tags($_GET['url'] ?? DEFAULT_URL);
$tag = strip_tags($_GET['tag'] ?? DEFAULT_TAG);

echo 'Dump of Tags: ' . PHP_EOL;
var_dump($vac->getTags($url, $tag));
```

输出结果如下所示：

![](../../.gitbook/assets/image%20%286%29.png)

## 参考

更多关于 DOM 的信息，请参阅 PHP 参考 [http://php.net/manual/en/class.domdocument.php](http://php.net/manual/en/class.domdocument.php)。

