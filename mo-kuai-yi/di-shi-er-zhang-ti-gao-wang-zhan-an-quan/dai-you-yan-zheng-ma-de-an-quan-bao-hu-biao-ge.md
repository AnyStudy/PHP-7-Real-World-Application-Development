# 带有验证码的安全保护表格

CAPTCHA实际上是完全自动公开图灵测试的缩写，以区分计算机和人类。该技术与前文介绍的 "用令牌保护表单安全 "中的技术类似。不同的是，该标记不是存储在一个隐藏的表单输入字段中，而是呈现为一个自动攻击系统难以解读的图形。此外，验证码的目的与表单令牌略有不同：它旨在确认网络访问者是人类，而不是自动系统。

## 如何做...

1.CAPTCHA有几种方法：根据只有人类才会拥有的知识提出问题，文字技巧，以及需要解释的图形图像。

2. 图像方法为网络访问者提供了一个带有严重扭曲的字母和/或数字的图像。然而，这种方法可能很复杂，因为它依赖于GD扩展，而GD扩展可能不是在所有服务器上都能使用。GD扩展可能很难编译，并且严重依赖主机服务器上必须存在的各种库。

3. 文本方法是呈现一系列的字母或数字，并给网络访问者一个简单的指示，如请倒着输入。另一种变化是使用ASCII "艺术 "来形成字符，使人类的网络访问者能够解释。

4. 最后，你可能会有一个问题或答案的方法，问题如The head is attached to the body by what body part，有答案如Arm、Leg、Neck。这种方法的缺点是，自动攻击系统通过测试的几率只有1/3。

### 生成一个文本验证码

1.在这个例子中，我们将从文本方法开始，然后是图像方法。无论哪种情况，我们首先需要定义一个类来生成短语（并由网络访问者解码）。为此，我们定义了一个`Application\Captcha\Phrase`类。我们还定义了短语生成过程中使用的属性和类常量。

```php
namespace Application\Captcha;
class Phrase
{
  const DEFAULT_LENGTH   = 5;
  const DEFAULT_NUMBERS  = '0123456789';
  const DEFAULT_UPPER    = 'ABCDEFGHJKLMNOPQRSTUVWXYZ';
  const DEFAULT_LOWER    = 'abcdefghijklmnopqrstuvwxyz';
  const DEFAULT_SPECIAL  = 
    '¬\`|!"£$%^&*()_-+={}[]:;@\'~#<,>.?/|\\';
  const DEFAULT_SUPPRESS = ['O','l'];

  protected $phrase;
  protected $includeNumbers;
  protected $includeUpper;
  protected $includeLower;
  protected $includeSpecial;
  protected $otherChars;
  protected $suppressChars;
  protected $string;
  protected $length;
```

2. 正如你所期望的那样，构造函数接受各种属性的值，并分配了默认值，因此无需指定任何参数即可创建实例。`$include*`标志用于指示哪些字符集将出现在基础字符串中，而短语将从基础字符串中生成。例如，如果你希望只包含数字，`$includeUpper`和`$includeLower`都会被设置为`FALSE`。提供`$otherChars`是为了增加灵活性。最后，`$suppressChars`代表一个将从基本字符串中删除的字符数组。默认情况下会删除大写字母`O`和小写字母`l`。

```php
public function __construct(
  $length = NULL,
  $includeNumbers = TRUE,
  $includeUpper= TRUE,
  $includeLower= TRUE,
  $includeSpecial = FALSE,
  $otherChars = NULL,
  array $suppressChars = NULL)
  {
    $this->length = $length ?? self::DEFAULT_LENGTH;
    $this->includeNumbers = $includeNumbers;
    $this->includeUpper = $includeUpper;
    $this->includeLower = $includeLower;
    $this->includeSpecial = $includeSpecial;
    $this->otherChars = $otherChars;
    $this->suppressChars = $suppressChars 
      ?? self::DEFAULT_SUPPRESS;
    $this->phrase = $this->generatePhrase();
  }
```

3. 然后我们定义了一系列的getter和setter，每个属性一个。请注意，为了节省空间，我们只显示前两个。

```php
public function getString()
{
  return $this->string;
}

public function setString($string)
{
  $this->string = $string;
}
    
// other getters and setters not shown
```

4. 接下来我们需要定义一个初始化基础字符串的方法。这由一系列简单的if语句组成，这些语句检查各种`$include*`标志，并适当地追加到基础字符串中。最后，我们使用`str_replace()`来删除`$suppressChars`中代表的字符。

```php
public function initString()
{
  $string = '';
  if ($this->includeNumbers) {
      $string .= self::DEFAULT_NUMBERS;
  }
  if ($this->includeUpper) {
      $string .= self::DEFAULT_UPPER;
  }
  if ($this->includeLower) {
      $string .= self::DEFAULT_LOWER;
  }
  if ($this->includeSpecial) {
      $string .= self::DEFAULT_SPECIAL;
  }
  if ($this->otherChars) {
      $string .= $this->otherChars;
  }
  if ($this->suppressChars) {
      $string = str_replace(
        $this->suppressChars, '', $string);
  }
  return $string;
}
```

{% hint style="info" %}
**最佳实践**

把可能与数字混淆的字母去掉（即字母O可以与数字0混淆，小写的l可以与数字1混淆。
{% endhint %}

5. 现在我们准备好定义核心方法，生成验证码呈现给网站访问者的随机短语。我们设置一个简单的`for()`循环，并使用新的PHP 7 `random_int()`函数在基础字符串中跳转。

```php
public function generatePhrase()
{
  $phrase = '';
  $this->string = $this->initString();
  $max = strlen($this->string) - 1;
  for ($x = 0; $x < $this->length; $x++) {
    $phrase .= substr(
      $this->string, random_int(0, $max), 1);
  }
  return $phrase;
}
}
```

6. 现在我们将注意力从短语转移到产生文本验证码的类上。为此，我们首先定义一个接口，以便将来可以创建更多的CAPTCHA类，这些类都可以使用`Application\Captcha\Phrase`。请注意，`getImage()`将返回文本、文本艺术或实际图像，这取决于我们决定使用哪个类。

```php
namespace Application\Captcha;
interface CaptchaInterface
{
  public function getLabel();
  public function getImage();
  public function getPhrase();
}
```

7. 对于文本验证码，我们定义了一个`Application\Captcha/Reverse`类。之所以叫这个名字，是因为这个类产生的不仅仅是文本，而是反向的文本。`__construct()`方法建立了一个`Phrase`的实例。请注意，`getImage()`返回的是反向的短语。

```php
namespace Application\Captcha;
class Reverse implements CaptchaInterface
{
  const DEFAULT_LABEL = 'Type this in reverse';
  const DEFAULT_LENGTH = 6;
  protected $phrase;
  public function __construct(
    $label  = self::DEFAULT_LABEL,
    $length = self:: DEFAULT_LENGTH,
    $includeNumbers = TRUE,
    $includeUpper   = TRUE,
    $includeLower   = TRUE,
    $includeSpecial = FALSE,
    $otherChars     = NULL,
    array $suppressChars = NULL)
  {
    $this->label  = $label;
    $this->phrase = new Phrase(
      $length, 
      $includeNumbers, 
      $includeUpper,
      $includeLower, 
      $includeSpecial, 
      $otherChars, 
      $suppressChars);
    }

  public function getLabel()
  {
    return $this->label;
  }

  public function getImage()
  {
    return strrev($this->phrase->getPhrase());
  }

  public function getPhrase()
  {
    return $this->phrase->getPhrase();
  }

}
```

### 生成一个图像验证码

1.形象的方法，你可以很好地想象，要复杂得多。短语的生成过程是一样的。主要的区别是，我们不仅需要将短语印在图形上，还需要对每个字母进行不同的变形，并以随机点的形式引入噪声。

2. 我们定义了一个`Application\Captcha\Image`类，实现了验证码接口。该类的常量和属性不仅包括生成短语所需要的，还包括生成图像所需要的。

```php
namespace Application\Captcha;
use DirectoryIterator;
class Image implements CaptchaInterface
{

  const DEFAULT_WIDTH = 200;
  const DEFAULT_HEIGHT = 50;
  const DEFAULT_LABEL = 'Enter this phrase';
  const DEFAULT_BG_COLOR = [255,255,255];
  const DEFAULT_URL = '/captcha';
  const IMAGE_PREFIX = 'CAPTCHA_';
  const IMAGE_SUFFIX = '.jpg';
  const IMAGE_EXP_TIME = 300;    // seconds
  const ERROR_REQUIRES_GD = 'Requires the GD extension + '
    .  ' the JPEG library';
  const ERROR_IMAGE = 'Unable to generate image';

  protected $phrase;
  protected $imageFn;
  protected $label;
  protected $imageWidth;
  protected $imageHeight;
  protected $imageRGB;
  protected $imageDir;
  protected $imageUrl;
```

3. 构造函数需要接受所有生成短语所需的参数，如前几步所述。此外，我们还需要接受生成图像所需的参数。两个必选参数是`$imageDir`和`$imageUrl`。第一个是图形将被写入的地方。第二个是基础URL，之后我们将附加生成的文件名。提供`$imageFont`是为了防止我们提供TrueType字体，这将产生一个更安全的验证码。否则，我们只能使用默认字体，引用著名电影中的一句台词，这可不是什么好事。

```php
public function __construct(
  $imageDir,
  $imageUrl,
  $imageFont = NULL,
  $label = NULL,
  $length = NULL,
  $includeNumbers = TRUE,
  $includeUpper= TRUE,
  $includeLower= TRUE,
  $includeSpecial = FALSE,
  $otherChars = NULL,
  array $suppressChars = NULL,
  $imageWidth = NULL,
  $imageHeight = NULL,
  array $imageRGB = NULL
)
{
```

4. 接下来，仍然在构造函数中，我们检查 `imagecreatetruecolor` 函数是否存在。如果返回的是false\`，我们就知道GD扩展不可用。否则，我们为属性分配参数，生成短语，删除旧图片，写出验证码图形。

```php
if (!function_exists('imagecreatetruecolor')) {
    throw new \Exception(self::ERROR_REQUIRES_GD);
}
$this->imageDir   = $imageDir;
$this->imageUrl   = $imageUrl;
$this->imageFont  = $imageFont;
$this->label      = $label ?? self::DEFAULT_LABEL;
$this->imageRGB   = $imageRGB ?? self::DEFAULT_BG_COLOR;
$this->imageWidth = $imageWidth ?? self::DEFAULT_WIDTH;
$this->imageHeight= $imageHeight ?? self::DEFAULT_HEIGHT;
if (substr($imageUrl, -1, 1) == '/') {
    $imageUrl = substr($imageUrl, 0, -1);
}
$this->imageUrl = $imageUrl;
if (substr($imageDir, -1, 1) == DIRECTORY_SEPARATOR) {
    $imageDir = substr($imageDir, 0, -1);
}

$this->phrase = new Phrase(
  $length, 
  $includeNumbers, 
  $includeUpper,
  $includeLower, 
  $includeSpecial, 
  $otherChars, 
  $suppressChars);
$this->removeOldImages();
$this->generateJpg();
}
```

5. 删除旧图片的过程是极其重要的，否则我们最终会发现目录中充满了过期的验证码图片！我们使用`DirectoryIterator`类扫描指定的目录，并检查访问时间。我们使用`DirectoryIterator`类来扫描指定的目录并检查访问时间。我们计算一个旧的图像文件是当前时间减去`IMAGE_EXP_TIME`指定的值。

```php
public function removeOldImages()
{
  $old = time() - self::IMAGE_EXP_TIME;
  foreach (new DirectoryIterator($this->imageDir) 
           as $fileInfo) {
    if($fileInfo->isDot()) continue;
    if ($fileInfo->getATime() < $old) {
      unlink($this->imageDir . DIRECTORY_SEPARATOR 
             . $fileInfo->getFilename());
    }
  }
}
```

6. 我们现在准备好进入正题。首先，我们将`$imageRGB`数组分割成`$red`、`$green`和`$blue`。我们使用核心的`imagecreatetruecolor()`函数来生成指定宽度和高度的基础图形。我们使用RGB值对背景进行着色。

```php
public function generateJpg()
{
  try {
      list($red,$green,$blue) = $this->imageRGB;
      $im = imagecreatetruecolor(
        $this->imageWidth, $this->imageHeight);
      $black = imagecolorallocate($im, 0, 0, 0);
      $imageBgColor = imagecolorallocate(
        $im, $red, $green, $blue);
      imagefilledrectangle($im, 0, 0, $this->imageWidth, 
        $this->imageHeight, $imageBgColor);
```

7. 接下来，我们根据图像的宽度和高度定义x和y边距。然后我们初始化变量，用于将短语写到图形上。然后，我们循环次数与短语的长度相匹配。

```php
$xMargin = (int) ($this->imageWidth * .1 + .5);
$yMargin = (int) ($this->imageHeight * .3 + .5);
$phrase = $this->getPhrase();
$max = strlen($phrase);
$count = 0;
$x = $xMargin;
$size = 5;
for ($i = 0; $i < $max; $i++) {
```

8. 如果指定了`$imageFont`，我们就能够用不同的大小和角度来书写每个字符。我们还需要根据大小调整x轴（即水平）值。

```php
if ($this->imageFont) {
    $size = rand(12, 32);
    $angle = rand(0, 30);
    $y = rand($yMargin + $size, $this->imageHeight);
    imagettftext($im, $size, $angle, $x, $y, $black, 
      $this->imageFont, $phrase[$i]);
    $x += (int) ($size  + rand(0,5));
```

9. 否则，我们就只能使用默认字体。我们使用最大的5号字体，因为较小的字体是不可读的。我们通过交替使用`imagechar()`和`imagecharup()`来提供低水平的失真，`imagechar()`可以正常写入图像，而`imagecharup()`可以横向写入图像。

```php
} else {
    $y = rand(0, ($this->imageHeight - $yMargin));
    if ($count++ & 1) {
        imagechar($im, 5, $x, $y, $phrase[$i], $black);
    } else {
        imagecharup($im, 5, $x, $y, $phrase[$i], $black);
    }
    $x += (int) ($size * 1.2);
  }
} // end for ($i = 0; $i < $max; $i++)
```

10. 接下来我们需要以随机点的形式添加噪声。这是必要的，以便使图像更难被自动系统检测到。同时建议你也添加代码来画几条线。

```php
$numDots = rand(10, 999);
for ($i = 0; $i < $numDots; $i++) {
  imagesetpixel($im, rand(0, $this->imageWidth), 
    rand(0, $this->imageHeight), $black);
}
```

11. 然后，我们使用我们的老朋友`md5(`\)创建一个随机的图像文件名，参数为日期和一个0到9999的随机数。请注意，我们可以安全地使用`md5()`，因为我们并没有试图隐藏任何秘密信息；我们只是想快速生成一个唯一的文件名。为了节省内存，我们也擦掉了图像对象。

```php
$this->imageFn = self::IMAGE_PREFIX 
. md5(date('YmdHis') . rand(0,9999)) 
. self::IMAGE_SUFFIX;
imagejpeg($im, $this->imageDir . DIRECTORY_SEPARATOR 
. $this->imageFn);
imagedestroy($im);
```

12. 整个构造都在`try/catch`块中。如果出现错误或异常，我们会记录消息并采取适当的行动。

```php
} catch (\Throwable $e) {
    error_log(__METHOD__ . ':' . $e->getMessage());
    throw new \Exception(self::ERROR_IMAGE);
}
}
```

13. 最后，我们定义接口所需的方法。注意，`getImage()`会返回一个HTML `<img>`标签，然后可以立即显示。

```php
public function getLabel()
{
  return $this->label;
}

public function getImage()
{
  return sprintf('<img src="%s/%s" />', 
    $this->imageUrl, $this->imageFn);
}

public function getPhrase()
{
  return $this->phrase->getPhrase();
}

}
```

## 如何运行...

一定要定义本事例中所讨论的类，总结在下表中。

| Class | 小节 | 出现步骤 |
| :--- | :--- | :--- |
| `Application\Captcha\Phrase` | Generating a text CAPTCHA | 1 - 5 |
| `Application\Captcha\CaptchaInterface` |  | 6 |
| `Application\Captcha\Reverse` |  | 7 |
| `Application\Captcha\Image` | Generating an image CAPTCHA | 2 - 13 |

接下来，定义一个名为`chap_12_captcha_text.php`的调用程序，实现文本验证码。你首先需要设置自动加载并使用相应的类。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Captcha\Reverse;
```

之后，一定要开始会话。你也会使用适当的措施来保护会话。为了节省空间，我们只展示一个简单的措施，`session_regenerate_id()`。

```php
session_start();
session_regenerate_id();
```

接下来，你可以定义一个创建验证码的函数；检索短语、标签和图像（在本例中，反向文本）；并将值存储在会话中。

```php
function setCaptcha(&$phrase, &$label, &$image)
{
  $captcha = new Reverse();
  $phrase  = $captcha->getPhrase();
  $label   = $captcha->getLabel();
  $image   = $captcha->getImage();
  $_SESSION['phrase'] = $phrase;
}
```

现在是初始化变量和确定登录状态的好时机。

```php
$image      = '';
$label      = '';
$phrase     = $_SESSION['phrase'] ?? '';
$message    = '';
$info       = 'You Can Now See Super Secret Information!!!';
$loggedIn   = $_SESSION['isLoggedIn'] ?? FALSE;
$loggedUser = $_SESSION['user'] ?? 'guest';
```

然后，您可以检查是否已按下登录按钮。如果是，检查是否输入了验证码短语。如果没有，则初始化一条消息，通知用户他们需要输入验证码短语。

```php
if (!empty($_POST['login'])) {
  if (empty($_POST['captcha'])) {
    $message = 'Enter Captcha Phrase and Login Information';
```

如果存在验证码短语，检查它是否与会话中存储的内容相匹配。如果不匹配，则继续进行表单无效的处理。否则，按照其他方式处理登录。在本例中，您可以使用硬编码的用户名和密码来模拟登录。

```php
} else {
    if ($_POST['captcha'] == $phrase) {
        $username = 'test';
        $password = 'password';
        if ($_POST['user'] == $username 
            && $_POST['pass'] == $password) {
            $loggedIn = TRUE;
            $_SESSION['user'] = strip_tags($username);
            $_SESSION['isLoggedIn'] = TRUE;
        } else {
            $message = 'Invalid Login';
        }
    } else {
        $message = 'Invalid Captcha';
    }
}
```

你可能还想添加注销选项的代码，就像保护PHP会话事例中描述的那样。

```php
} elseif (isset($_POST['logout'])) {
  session_unset();
  session_destroy();
  setcookie('PHPSESSID', 0, time() - 3600);
  header('Location: ' . $_SERVER['REQUEST_URI'] );
  exit;
}
```

然后你可以运行`setCaptcha()`。

```php
setCaptcha($phrase, $label, $image);
```

最后，别忘了视图逻辑，在这个例子中，呈现的是一个基本的登录表单。在表单标签里面，你需要添加视图逻辑来显示验证码和标签。

```php
<tr>
  <th><?= $label; ?></th>
  <td><?= $image; ?><input type="text" name="captcha" /></td>
</tr>
```

下面是结果输出。

![](../../.gitbook/assets/image%20%28146%29.png)

为了演示如何使用图片验证码，请将`chap_12_captcha_text.php`中的代码复制到`cha_12_captcha_image.php`中。我们定义了常量来表示验证码图片的目录位置. \(一定要创建这个目录!\)否则，自动加载和使用语句的结构是相似的。注意，我们还定义了一个TrueType字体。不同之处用粗体表示。

```php
<?php
define('IMAGE_DIR', __DIR__ . '/captcha');
define('IMAGE_URL', '/captcha');
define('IMAGE_FONT', __DIR__ . '/FreeSansBold.ttf');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Captcha\Image;

session_start();
session_regenerate_id();
```

{% hint style="danger" %}
字体可能受到版权、商标、专利或其他知识产权法律的保护。如果您使用的字体没有得到授权，您和您的客户可能会被追究法律责任。使用开放源码的字体，或者是在网络服务器上可以使用的字体，你得有一个有效的许可证。
{% endhint %}

当然，在`setCaptcha()`函数中，我们使用`Image`类代替`Reverse`。

```php
function setCaptcha(&$phrase, &$label, &$image)
{
  $captcha = new Image(IMAGE_DIR, IMAGE_URL, IMAGE_FONT);
  $phrase  = $captcha->getPhrase();
  $label   = $captcha->getLabel();
  $image   = $captcha->getImage();
  $_SESSION['phrase'] = $phrase;
  return $captcha;
}
```

变量初始化与上一个脚本相同，登录处理与上一个脚本相同。

```php
$image      = '';
$label      = '';
$phrase     = $_SESSION['phrase'] ?? '';
$message    = '';
$info       = 'You Can Now See Super Secret Information!!!';
$loggedIn   = $_SESSION['isLoggedIn'] ?? FALSE;
$loggedUser = $_SESSION['user'] ?? 'guest';

if (!empty($_POST['login'])) {

  // etc.  -- identical to chap_12_captcha_text.php
```

即使是视图逻辑也是一样的，因为我们使用的是`getImage(`\)，在图片验证码的情况下，它直接返回可用的HTML。下面是使用TrueType字体的输出。

![](../../.gitbook/assets/image%20%28158%29.png)

## 更多...

如果你不愿意使用前面的代码来生成自己的内部验证码，有很多库可以使用。大多数流行的框架都有这种能力。例如，Zend Framework有其`Zend\Captcha`组件类。还有reCAPTCHA，它通常是作为一个服务调用的，在这个服务中，你的应用程序调用一个外部网站，这个网站为你生成验证码和令牌。一个好的地方是[http://www.captcha.net/](http://www.captcha.net/) 网站。

关于将字体作为知识产权加以保护的更多信息，请参阅https://en.wikipedia.org/wiki/Intellectual\_property\_protection\_of\_typefaces。



