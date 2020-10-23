# 按地区处理货币

处理货币的技术与处理数字的技术类似。我们甚至会使用相同的`NumberFormatter`类。然而，有一个主要的区别，它是一个引人注目的问题：为了正确地格式化货币，你需要手头有货币代码。

## 如何做...

1.首先要做的是以某种格式提供货币代码。一种可能性是简单地将货币代码添加为`Application\I18n\Locale`类构造函数参数。

```php
const FALLBACK_CURRENCY = 'GBP';
protected $currencyCode;
public function __construct($localeString = NULL, $currencyCode = NULL)
{
  // add this to the existing code:
  $this->currencyCode = $currencyCode ?? self::FALLBACK_CURRENCY;
}
```

{% hint style="info" %}
这种方法虽然明显是稳妥可行的，但往往属于所谓的半途而废或易如反掌! 这种方法也倾向于消除完全自动化，因为货币代码无法从HTTP头中获得。正如你可能已经从本书中的其他事例中收集到的，我们并不回避更复杂的解决方案，所以，正如俗话说的那样，系上你的安全带吧！我们的解决方案是："你可以在你的网站上找到你想要的东西"。
{% endhint %}

2.我们首先需要建立某种查找机制，给定一个国家代码，我们就可以获得其主要的货币代码。在这个例子中，我们将使用适配器软件设计模式。根据这种模式，我们应该能够创建不同的类，这些类有可能以完全不同的方式进行操作，但却产生相同的结果。据此，我们需要定义所需的结果。为此，我们引入一个类，`Application\I18n\IsoCodes`。正如你所看到的，这个类有所有相关的属性，还有一个通用的构造函数。

```php
namespace Application\I18n;
class IsoCodes
{
  public $name;
  public $iso2;
  public $iso3;
  public $iso_numeric;
  public $iso_3166;
  public $currency_name;
  public $currency_code;
  public $currency_number;
  public function __construct(array $data)
  {
    $vars = get_object_vars($this);
    foreach ($vars as $key => $value) {
      $this->$key = $data[$key] ?? NULL;
    }
  }
}
```

3.接下来我们定义一个接口，它有我们需要的方法来执行国家代码到货币代码的查询。在这种情况下，我们引入`Application\I18n\IsoCodesInterface`。

```php
namespace Application\I18n;

interface IsoCodesInterface
{
  public function getCurrencyCodeFromIso2CountryCode($iso2) : IsoCodes;
}
```

4. ****现在我们准备建立一个查找适配器类，我们将把它称为`Application\I18n\IsoCodesDb`。它实现了上述接口，并接受一个`Application\Database\Connection`实例\(参见第1章，建立基础\)，用于执行查找。构造函数设置了所需的信息，包括连接、查找表名和代表ISO2代码的列。然后，接口所需的查找方法会发出一条SQL语句并返回一个数组，然后用这个数组来构建一个`IsoCodes`实例。

```php
namespace Application\I18n;

use PDO;
use Application\Database\Connection;

class IsoCodesDb implements IsoCodesInterface
{
  protected $isoTableName;
  protected $iso2FieldName;
  protected $connection;
  public function __construct(Connection $connection, $isoTableName, $iso2FieldName)
  {
    $this->connection = $connection;
    $this->isoTableName = $isoTableName;
    $this->iso2FieldName = $iso2FieldName;
  }
  public function getCurrencyCodeFromIso2CountryCode($iso2) : IsoCodes
  {
    $sql = sprintf('SELECT * FROM %s WHERE %s = ?', $this->isoTableName, $this->iso2FieldName);
    $stmt = $this->connection->pdo->prepare($sql);
    $stmt->execute([$iso2]);
    return new IsoCodes($stmt->fetch(PDO::FETCH_ASSOC);
  }
}
```

5. 现在我们把注意力转回`Application\I18n\Locale`类。我们首先添加一些新的属性和类常数。

```php
const ERROR_UNABLE_TO_PARSE = 'ERROR: Unable to parse';
const FALLBACK_CURRENCY = 'GBP';

protected $currencyFormatter;
protected $currencyLookup;
protected $currencyCode;
```

6. 我们添加新的方法，从locale字符串中获取国家代码。我们可以利用`getRegion()`方法，它来自于PHP `Locale`类（我们扩展了它）。为了以防万一，我们还添加了一个方法`getCurrencyCode()`。

```php
public function getCountryCode()
{
  return $this->getRegion($this->getLocaleCode());
}
public function getCurrencyCode()
{
  return $this->currencyCode;
}
```

7. 与格式化数字一样，我们定义一个`getCurrencyFormatter(I)`，就像我们定义`getNumberFormatter()`一样（如前所示）。注意，`$currencyFormatter`是用`NumberFormatter`定义的，但第二个参数不同。

```php
public function getCurrencyFormatter()
{
  if (!$this->currencyFormatter) {
    $this->currencyFormatter = new NumberFormatter($this->getLocaleCode(), NumberFormatter::CURRENCY);
  }
  return $this->currencyFormatter;
}
```

8. 然后，如果已经定义了查询类，我们就在类的构造函数中添加货币代码查询。

```php
public function __construct($localeString = NULL, IsoCodesInterface $currencyLookup = NULL)
{
  // add this to the existing code:
  $this->currencyLookup = $currencyLookup;
  if ($this->currencyLookup) {
    $this->currencyCode = $this->currencyLookup->getCurrencyCodeFromIso2CountryCode($this->getCountryCode())->currency_code;
  } else {
    $this->currencyCode = self::FALLBACK_CURRENCY;
  }
}
```

9. 然后添加适当的货币格式和解析方法。注意，解析货币与解析数字不同，如果解析操作不成功，将返回`FALSE`。

```php
public function formatCurrency($currency)
{
  return $this->getCurrencyFormatter()->formatCurrency($currency, $this->currencyCode);
}
public function parseCurrency($string)
{
  $result = $this->getCurrencyFormatter()->parseCurrency($string, $this->currencyCode);
  return ($result) ? $result : self::ERROR_UNABLE_TO_PARSE;
}
```

## 如何运行...

如前几个要点中所涉及的那样，创建以下类。

| Class | 讨论要点 |
| :--- | :--- |
| `Application\I18n\IsoCodes` | 3 |
| `Application\I18n\IsoCodesInterface` | 4 |
| `Application\I18n\IsoCodesDb` | 5 |

为便于说明，我们假设我们有一个已填充的MySQL数据库表\`iso\_country\_codes\`，其结构如下。

```php
CREATE TABLE `iso_country_codes` (
  `name` varchar(128) NOT NULL,
  `iso2` varchar(2) NOT NULL,
  `iso3` varchar(3) NOT NULL,
  `iso_numeric` int(11) NOT NULL AUTO_INCREMENT,
  `iso_3166` varchar(32) NOT NULL,
  `currency_name` varchar(32) DEFAULT NULL,
  `currency_code` char(3) DEFAULT NULL,
  `currency_number` int(4) DEFAULT NULL,
  PRIMARY KEY (`iso_numeric`)
) ENGINE=InnoDB AUTO_INCREMENT=895 DEFAULT CHARSET=utf8;
```

在`Application\I18n\Locale`类中进行添加，如前面第6至9点中所讨论的那样。然后你可以创建一个`chap_08_formatting_currency.php`文件，该文件设置了自动加载并使用了相应的类。

```php
<?php
define('DB_CONFIG_FILE', __DIR__ . '/../config/db.config.php');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\I18n\Locale;
use Application\I18n\IsoCodesDb;
use Application\Database\Connection;
use Application\I18n\Locale;
```

接下来，我们创建`Connection`和`IsoCodesDb`类的实例。

```php
$connection = new Connection(include DB_CONFIG_FILE);
$isoLookup = new IsoCodesDb($connection, 'iso_country_codes', 'iso2');
```

在这个例子中，创建两个`Locale`实例，一个用于英国，另一个用于法国。你也可以指定一个大的数量来用于测试。

```php
$localeFr = new Locale('fr-FR', $isoLookup);
$localeUk = new Locale('en_GB', $isoLookup);
$number   = 1234567.89;
?>
```

最后，你可以将`formatCurrency()`和`parseCurrency()`方法包在适当的HTML显示逻辑中，并查看结果。这里是最终的输出。

![](../../.gitbook/assets/image%20%28102%29.png)

## 更多...

最新的货币代码列表由ISO（国际标准组织）维护。你可以用XML或XLS\(即微软Excel电子表格格式\)获得这份清单。这里是可以找到这些清单的网页：[http://www.currency-iso.org/en/home/tables/table-a1.html](http://www.currency-iso.org/en/home/tables/table-a1.html)。

