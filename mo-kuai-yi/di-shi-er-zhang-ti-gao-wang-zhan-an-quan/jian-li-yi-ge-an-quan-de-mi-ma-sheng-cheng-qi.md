# 建立一个安全的密码生成器

一个常见的误解是，攻击者破解哈希密码的唯一方法是使用蛮力攻击和彩虹表。虽然这通常是攻击序列中的第一道关口，但攻击者会在第二道、第三道或第四道关口使用更复杂的攻击。其他攻击包括组合、字典、掩码和基于规则的攻击。字典攻击是利用数据库中字面上的词来猜测密码。组合是将字典中的词组合起来。掩码攻击类似于蛮力，但选择性更强，从而缩短破解时间。基于规则的攻击会检测到诸如用数字0代替字母o的情况。

好消息是，只要将密码的长度增加到6个字符的神奇长度之外，就会成倍地增加破解哈希密码的时间。其他因素，如将大写字母与小写字母随机穿插、随机数字和特殊字符，也会对破解时间产生指数级影响。最后，我们需要牢记，人类最终需要输入创建的密码，这意味着需要至少有一定的记忆力。

{% hint style="info" %}
**最佳实践**

密码应该以哈希值的形式存储，而不是纯文本。MD5和SHA_不再被认为是安全的（尽管SHA比MD5好得多）_。使用`oclHashcat`这样的工具，攻击者平均每秒可以对使用MD5散列的密码产生550亿次的尝试，而这个密码是通过漏洞（即成功的SQL注入攻击）获得的。
{% endhint %}

## 如何做...

1.首先，我们定义了一个`Application\Security\PassGen`类，它将持有密码生成所需的方法。我们还定义了某些类常量和属性，这些常量和属性将作为过程的一部分被使用。

```php
namespace Application\Security;
class PassGen
{
  const SOURCE_SUFFIX = 'src';
  const SPECIAL_CHARS = 
    '\`¬|!"£$%^&*()_-+={}[]:@~;\'#<>?,./|\\';
  protected $algorithm;
  protected $sourceList;
  protected $word;
  protected $list;
```

2. 然后我们定义了用于生成密码的低级方法。顾名思义，`digits()`产生的是随机数字，`special()`产生的是`SPECIAL_CHARS`类常量中的一个字符。

```php
public function digits($max = 999)
{
  return random_int(1, $max);
}

public function special()
{
  $maxSpecial = strlen(self::SPECIAL_CHARS) - 1;
  return self::SPECIAL_CHARS[random_int(0, $maxSpecial)];
}
```

{% hint style="info" %}
请注意，我们在这个例子中经常使用新的 PHP 7 函数 `random_int()`。虽然速度稍慢，但与更古老的 `rand()` 函数相比，这个方法提供了真正的加密安全伪随机数生成器 \(CSPRNG\) 功能。
{% endhint %}

3. 现在是棘手的部分：生成一个难以猜测的单词。这就是`$wordSource`构造函数参数发挥作用的地方。它是一个网站的数组，我们的词库将从这个数组中产生。相应地，我们需要一个方法，从指定的来源中提取一个唯一的单词列表，并将结果存储在一个文件中。我们接受`$wordSource`数组作为参数，并循环浏览每个URL。我们使用`md5()`产生一个网站名称的哈希值，然后将其建立为一个文件名。新生成的文件名将存储在`$sourceList`中。

```php
public function processSource(
$wordSource, $minWordLength, $cacheDir)
{
  foreach ($wordSource as $html) {
    $hashKey = md5($html);
    $sourceFile = $cacheDir . '/' . $hashKey . '.' 
    . self::SOURCE_SUFFIX;
    $this->sourceList[] = $sourceFile;
```

4. 如果文件不存在，或者是零字节，我们将处理其内容。如果源文件是HTML，我们只接受`<body>`标签内的内容。然后，我们使用`str_word_count()`从字符串中提取一个单词列表，同时使用`strip_tags()`来删除任何标记。

```php
if (!file_exists($sourceFile) || filesize($sourceFile) == 0) {
    echo 'Processing: ' . $html . PHP_EOL;
    $contents = file_get_contents($html);
    if (preg_match('/<body>(.*)<\/body>/i', 
        $contents, $matches)) {
        $contents = $matches[1];
    }
    $list = str_word_count(strip_tags($contents), 1);
```

5. 然后，我们删除太短的单词，并使用`array_unique()`去掉重复的单词。最后的结果存储在一个文件中。

```php
     foreach ($list as $key => $value) {
       if (strlen($value) < $minWordLength) {
         $list[$key] = 'xxxxxx';
       } else {
         $list[$key] = trim($value);
       }
     }
     $list = array_unique($list);
     file_put_contents($sourceFile, implode("\n",$list));
   }
  }
  return TRUE;
}
```

6. 接下来，我们定义了一个将单词中的随机字母翻转为大写的方法。

```php
public function flipUpper($word)
{
  $maxLen   = strlen($word);
  $numFlips = random_int(1, $maxLen - 1);
  $flipped  = strtolower($word);
  for ($x = 0; $x < $numFlips; $x++) {
       $pos = random_int(0, $maxLen - 1);
       $word[$pos] = strtoupper($word[$pos]);
  }
  return $word;
}
```

7. 最后，我们准备定义一个方法，从我们的词源中选择一个词。我们随机选择一个词源，然后使用`file()`函数从相应的缓存文件中读取。

```php
public function word()
{
  $wsKey    = random_int(0, count($this->sourceList) - 1);
  $list     = file($this->sourceList[$wsKey]);
  $maxList  = count($list) - 1;
  $key      = random_int(0, $maxList);
  $word     = $list[$key];
  return $this->flipUpper($word);
}
```

8. 为了使我们不总是产生相同模式的密码，我们定义了一种方法，允许我们将密码的各个组成部分放在最终密码字符串的不同位置。算法被定义为这个类中可用的方法调用数组。因此，例如，`['word'，'digits'，'word'，'special']`的算法可能最终看起来像`hElLo123aUTo!`

```php
public function initAlgorithm()
{
  $this->algorithm = [
    ['word', 'digits', 'word', 'special'],
    ['digits', 'word', 'special', 'word'],
    ['word', 'word', 'special', 'digits'],
    ['special', 'word', 'special', 'digits'],
    ['word', 'special', 'digits', 'word', 'special'],
    ['special', 'word', 'special', 'digits', 
    'special', 'word', 'special'],
  ];
}
```

9. 构造函数接受字源数组、最小字长和缓存目录的位置。然后处理源文件并初始化算法。

```php
public function __construct(
  array $wordSource, $minWordLength, $cacheDir)
{
  $this->processSource($wordSource, $minWordLength, $cacheDir);
  $this->initAlgorithm();
}
```

10. 最后，我们能够定义实际生成密码的方法。它需要做的就是随机选择一个算法，然后循环，调用相应的方法。

```php
public function generate()
{
  $pwd = '';
  $key = random_int(0, count($this->algorithm) - 1);
  foreach ($this->algorithm[$key] as $method) {
    $pwd .= $this->$method();
  }
  return str_replace("\n", '', $pwd);
}

}
```

## 如何运行...

首先，你需要把前面配方中描述的代码放到`Application\Security`文件夹中的一个名为`PassGen.php`的文件中。现在你可以创建一个名为`chap_12_password_generate.php`的调用程序，设置自动加载，使用`PassGen`，并定义缓存目录的位置。

```php
<?php
define('CACHE_DIR', __DIR__ . '/cache');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Security\PassGen;
```

接下来，您需要定义一系列的网站，这些网站将被用作生成密码时使用的词库的来源。在本例中，我们将选择Project Gutenberg文本Ulysses \(J. Joyce\), War and Peace \(L. Tolstoy\), 和Pride and Prejudice \(J. Austen\)。

```php
$source = [
  'https://www.gutenberg.org/files/4300/4300-0.txt',
  'https://www.gutenberg.org/files/2600/2600-h/2600-h.htm',
  'https://www.gutenberg.org/files/1342/1342-h/1342-h.htm',
];
```

接下来，我们创建`PassGen`实例，并运行`generate()`。

```php
$passGen = new PassGen($source, 4, CACHE_DIR);
echo $passGen->generate();
```

下面是`PassGen`制作的几个密码示例。

![](../../.gitbook/assets/image%20%28195%29.png)

## 更多...

一篇关于攻击者如何破解密码的优秀文章可以在http://arstechnica.com/security/2013/05/how-crackers-make-minced-meat-out-of-your-passwords/。要了解更多关于蛮力攻击的信息，可以参考https://www.owasp.org/index.php/Brute\_force\_attack。关于oclHashcat的信息，请看这个网页：http://hashcat.net/oclhashcat/。

