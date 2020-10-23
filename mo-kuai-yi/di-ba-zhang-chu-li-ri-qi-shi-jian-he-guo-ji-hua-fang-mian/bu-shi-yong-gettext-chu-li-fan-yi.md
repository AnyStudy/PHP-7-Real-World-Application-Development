# 不使用gettext处理翻译

翻译是使您的网站能够被国际客户群访问的一个重要部分。其中一种方法是使用PHP的`gettext`函数，它是基于安装在本地服务器上的GNU `gettext`操作系统工具。因此，在这个事例中，我们提出了一种替代的翻译方法，你可以建立自己的适配器。

有一点很重要，那就是 PHP 的程序翻译工具主要是用来提供有限的单词或短语的翻译，被称为 msgid（消息 ID）。相应的翻译被称为 msgid \(消息字符串\)。因此，结合翻译通常只涉及相对不变的项目，如菜单、表单、错误或成功信息等。在本事例中，我们将假设您将实际的网页翻译存储为文本块。

{% hint style="info" %}
如果你需要翻译整个页面的内容，你可以考虑使用谷歌翻译API。不过，这是一项付费服务。另外，您也可以使用Amazon Mechanical Turk将翻译工作外包给具有多语言技能的个人，价格便宜。
{% endhint %}

## 如何做...

1.我们将再次使用适配器软件设计模式，在这种情况下，提供翻译源的替代品。在这个配方中，我们将演示`.ini`文件、`.csv`文件和数据库的适配器。

2. 首先，我们将定义一个接口，稍后将用来标识一个翻译适配器。对翻译适配器的要求非常简单，我们只需要为给定的消息ID返回一个消息字符串。

```php
namespace Application\I18n\Translate\Adapter;
interface TranslateAdapterInterface
{
  public function translate($msgid);
}
```

3. 接下来我们定义一个与接口匹配的trait。该trait将包含实际所需的代码。请注意，如果我们找不到消息字符串，我们只需返回消息ID。

```php
namespace Application\I18n\Translate\Adapter;

trait TranslateAdapterTrait
{
  protected $translation;
  public function translate($msgid)
  {
    return $this->translation[$msgid] ?? $msgid;
  }
}
```

4. 现在我们准备好定义我们的第一个适配器。在这个配方中，我们将从一个使用`.ini`文件作为翻译源的适配器开始。你会注意到的第一件事就是我们使用了之前定义的trait。不同适配器的构造方法会有所不同。在这种情况下，我们使用`parse_ini_file()`来产生一个key/value对的数组，其中key是消息ID。请注意，我们使用`$filePattern`参数来替换locale，然后允许我们加载适当的翻译文件。

```php
namespace Application\I18n\Translate\Adapter;

use Exception;
use Application\I18n\Locale;

class Ini implements TranslateAdapterInterface
{
  use TranslateAdapterTrait;
  const ERROR_NOT_FOUND = 'Translation file not found';
  public function __construct(Locale $locale, $filePattern)
  {
    $translateFileName = sprintf($filePattern, $locale->getLocaleCode());
    if (!file_exists($translateFileName)) {
      error_log(self::ERROR_NOT_FOUND . ':' . $translateFileName);
      throw new Exception(self::ERROR_NOT_FOUND);
    } else {
      $this->translation = parse_ini_file($translateFileName);
    }
  }
}
```

5.下一个适配器，`Application\I18n\Translate\Adapter\Csv`，是相同的，除了我们打开翻译文件并使用`fgetcsv()`循环检索消息ID/消息字符串键对。这里我们只展示了构造函数的不同。

```php
public function __construct(Locale $locale, $filePattern)
{
  $translateFileName = sprintf($filePattern, $locale->getLocaleCode());
  if (!file_exists($translateFileName)) {
    error_log(self::ERROR_NOT_FOUND . ':' . $translateFileName);
    throw new Exception(self::ERROR_NOT_FOUND);
  } else {
    $fileObj = new SplFileObject($translateFileName, 'r');
    while ($row = $fileObj->fgetcsv()) {
      $this->translation[$row[0]] = $row[1];
    }
  }
}
```

{% hint style="warning" %}
这两种适配器的最大缺点是，我们需要预加载整个翻译集，如果有大量的翻译，会对内存造成压力。另外，翻译文件需要打开并解析，这也拖累了性能。
{% endhint %}

6. 现在我们介绍第三个适配器，它执行数据库查询，避免了其他两个适配器的问题。我们使用一个PDO准备好的语句，在一开始就发送给数据库，而且只有一次。然后我们根据需要执行多次，提供消息ID作为参数。你还会注意到，我们需要重写trait中定义的`translate()`方法。最后，你可能已经注意到了`PDOStatement::fetchColumn()`的使用，因为我们只需要一个值。

```php
namespace Application\I18n\Translate\Adapter;

use Exception;
use Application\Database\Connection;
use Application\I18n\Locale;

class Database implements TranslateAdapterInterface
{
  use TranslateAdapterTrait;
  protected $connection;
  protected $statement;
  protected $defaultLocaleCode;
  public function __construct(Locale $locale, 
                              Connection $connection, 
                              $tableName)
  {
    $this->defaultLocaleCode = $locale->getLocaleCode();
    $this->connection = $connection;
    $sql = 'SELECT msgstr FROM ' . $tableName 
       . ' WHERE localeCode = ? AND msgid = ?';
    $this->statement = $this->connection->pdo->prepare($sql);
  }
  public function translate($msgid, $localeCode = NULL)
  {
    if (!$localeCode) $localeCode = $this->defaultLocaleCode;
    $this->statement->execute([$localeCode, $msgid]);
    return $this->statement->fetchColumn();
  }
}
```

7. 我们现在准备好定义核心的`Translation`类，它与一个（或多个）适配器相连。我们分配一个类常量来表示默认的locale，以及locale、适配器和文本文件模式的属性（后面会解释）。

```php
namespace Application\I18n\Translate;

use Application\I18n\Locale;
use Application\I18n\Translate\Adapter\TranslateAdapterInterface;

class Translation
{
  const DEFAULT_LOCALE_CODE = 'en_GB';
  protected $defaultLocaleCode;
  protected $adapter = array();
  protected $textFilePattern = array();
```

8. 在构造函数中，我们确定locale，并将初始适配器设置为该locale。这样一来，我们就可以托管多个适配器。

```php
public function __construct(TranslateAdapterInterface $adapter, 
              $defaultLocaleCode = NULL, 
              $textFilePattern = NULL)
{
  if (!$defaultLocaleCode) {
    $this->defaultLocaleCode = self::DEFAULT_LOCALE_CODE;
  } else {
    $this->defaultLocaleCode = $defaultLocaleCode;
  }
  $this->adapter[$this->defaultLocaleCode] = $adapter;
  $this->textFilePattern[$this->defaultLocaleCode] = $textFilePattern;
}
```

9. 接下来我们定义了一系列的设定器，这让我们有更多的灵活性。

```php
public function setAdapter($localeCode, TranslateAdapterInterface $adapter)
{
  $this->adapter[$localeCode] = $adapter;
}
public function setDefaultLocaleCode($localeCode)
{
  $this->defaultLocaleCode = $localeCode;
}
public function setTextFilePattern($localeCode, $pattern)
{
  $this->textFilePattern[$localeCode] = $pattern;
}
```

10. 然后我们定义了PHP的神奇方法`__invoke()`，让我们直接调用翻译器实例，返回给定消息ID的消息字符串。

```php
public function __invoke($msgid, $locale = NULL)
{
  if ($locale === NULL) $locale = $this->defaultLocaleCode;
  return $this->adapter[$locale]->translate($msgid);
}
```

11. 最后，我们还添加了一个方法，可以从文本文件中返回翻译的文本块。请记住，这可以被修改为使用数据库来代替。我们没有在适配器中包含这个功能，因为它的目的是完全不同的；我们只是想返回给定一个键的大代码块，可以想象这个键是翻译后的文本文件的文件名。

```php
public function text($key, $localeCode = NULL)
{
  if ($localeCode === NULL) $localeCode = $this->defaultLocaleCode;
  $contents = $key;
  if (isset($this->textFilePattern[$localeCode])) {
    $fn = sprintf($this->textFilePattern[$localeCode], $localeCode, $key);
    if (file_exists($fn)) {
      $contents = file_get_contents($fn);
    }
  }
  return $contents;
}
```

## 如何运行...

首先您需要定义一个目录结构来存放翻译文件。在这个例子中，你可以建立一个目录，`/path/to/project/files/data/languages`。在这个目录结构下，创建代表不同语言的子目录。在这个例子中，你可以使用这些：`de_DE`、`fr_FR`、`en_GB`和`es_ES`，代表德语、法语、英语和西班牙语。

接下来你需要创建不同的翻译文件。举个例子，这里是一个具有代表性的西班牙语的`data/languages/es_ES/translation.ini`文件。

```php
Welcome=Bienvenido
About Us=Sobre Nosotros
Contact Us=Contáctenos
Find Us=Encontrarnos
click=clic para más información
```

同样，为了演示CSV适配器，也可以创建一个和CSV文件一样的东西，`data/languages/es_ES/translation.csv`。

```text
"Welcome","Bienvenido"
"About Us","Sobre Nosotros"
"Contact Us","Contáctenos"
"Find Us","Encontrarnos"
"click","clic para más información"
```

最后，创建数据库表，进行翻译，并使用相同的数据填充它。 主要区别在于数据库表将具有三个字段：`msgid`，`msgstr`和`locale_code`：

```php
CREATE TABLE `translation` (
  `msgid` varchar(255) NOT NULL,
  `msgstr` varchar(255) NOT NULL,
  `locale_code` char(6) NOT NULL DEFAULT '',
  PRIMARY KEY (`msgid`,`locale_code`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

接下来，使用本事例中所示的代码，定义前面提到的类。

* `Application\I18n\Translate\Adapter\TranslateAdapterInterface`
* `Application\I18n\Translate\Adapter\TranslateAdapterTrait`
* `Application\I18n\Translate\Adapter\Ini`
* `Application\I18n\Translate\Adapter\Csv`
* `Application\I18n\Translate\Adapter\Database`
* `Application\I18n\Translate\Translation`

现在你可以创建一个测试文件，`chap_08_translation_database.php`，来测试数据库翻译适配器。它应该实现自动加载，使用适当的类，并创建一个`Locale`和`Connection`实例。注意`TEXT_FILE_PATTERN`常量是一个`sprintf()`模式，其中locale代码和文件名被替换。

```php
<?php
define('DB_CONFIG_FILE', '/../config/db.config.php');
define('TEXT_FILE_PATTERN', __DIR__ . '/../data/languages/%s/%s.txt');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\I18n\Locale;
use Application\I18n\Translate\ { Translation, Adapter\Database };
use Application\Database\Connection;

$conn = new Connection(include __DIR__ . DB_CONFIG_FILE);
$locale = new Locale('fr_FR');
```

接下来，创建一个翻译适配器实例，并使用它来创建一个翻译实例。

```php
$adapter = new Database($locale, $conn, 'translation');
$translate = new Translation($adapter, $locale->getLocaleCode(), TEXT_FILE_PATTERN);
?>
```

最后，创建显示逻辑，使用`$translate`实例。

```php
<!DOCTYPE html>
<head>
  <title>PHP 7 Cookbook</title>
  <meta http-equiv="content-type" content="text/html;charset=utf-8" />
  <link rel="stylesheet" type="text/css" href="php7cookbook_html_table.css">
</head>
<body>
<table>
<tr>
  <th><h1 style="color:white;"><?= $translate('Welcome') ?></h1></th>
  <td>
    <div style="float:left;width:50%;vertical-align:middle;">
    <h3 style="font-size:24pt;"><i>Some Company, Inc.</i></h3>
    </div>
    <div style="float:right;width:50%;">
    <img src="jcartier-city.png" width="300px"/>
    </div>
  </td>
</tr>
<tr>
  <th>
    <ul>
      <li><?= $translate('About Us') ?></li>
      <li><?= $translate('Contact Us') ?></li>
      <li><?= $translate('Find Us') ?></li>
    </ul>
  </th>
  <td>
    <p>
    <?= $translate->text('main_page'); ?>
    </p>
    <p>
    <a href="#"><?= $translate('click') ?></a>
    </p>
  </td>
</tr>
</table>
</body>
</html>
```

然后，您可以执行其他类似的测试，替换一个新的locale来获得不同的语言，或者使用另一个适配器来测试不同的数据源。下面是一个使用`fr_FR`的locale和数据库翻译适配器进行输出的例子。

![](../../.gitbook/assets/image%20%28103%29.png)

## 更多...

* 有关Google翻译API的更多信息，请参见[https://cloud.google.com/translate/v2/translating-text-with-rest](https://cloud.google.com/translate/v2/translating-text-with-rest)。 
* 有关Amazon Mechanical Turk的更多信息，请参见[https://www.mturk.com/mturk/welcome](https://www.mturk.com/mturk/welcome)。有关`gettext`的更多信息，请参见[http://www.gnu.org/software/gettext/manual/gettext.html](http://www.gnu.org/software/gettext/manual/gettext.html)。

