# 按地区设置日期/时间格式

日期和时间的格式因地区而异。作为一个经典的例子，考虑2016年4月15日晚上的一个时间。美国居民喜欢的格式是 7:23 PM,4/15/2016，而在中国，你很可能会看到2016-04-15 19:23。如同前面提到的数字和货币格式，以你的网站访问者可以接受的格式显示（和解析）日期也很重要。

## 如何做...

1.首先，我们需要修改`Application\I18n\Locale`，添加语句使用日期格式类。

```php
use IntlCalendar;
use IntlDateFormatter;
```

2.接下来，我们添加一个属性来表示一个 `IntlDateFormatter` 实例，以及一系列预定义常量。

```php
const DATE_TYPE_FULL   = IntlDateFormatter::FULL;
const DATE_TYPE_LONG   = IntlDateFormatter::LONG;
const DATE_TYPE_MEDIUM = IntlDateFormatter::MEDIUM;
const DATE_TYPE_SHORT  = IntlDateFormatter::SHORT;

const ERROR_UNABLE_TO_PARSE = 'ERROR: Unable to parse';
const ERROR_UNABLE_TO_FORMAT = 'ERROR: Unable to format date';
const ERROR_ARGS_STRING_ARRAY = 'ERROR: Date must be string YYYY-mm-dd HH:ii:ss or array(y,m,d,h,i,s)';
const ERROR_CREATE_INTL_DATE_FMT = 'ERROR: Unable to create international date formatter';

protected $dateFormatter;
```

3. 之后，我们就可以定义一个方法`getDateFormatter()`，它将返回一个`IntlDateFormatter`实例。`$type` 的值与之前定义的 `DATE_TYPE_*` 常量中的一个相匹配。

```php
public function getDateFormatter($type)
{
  switch ($type) {
    case self::DATE_TYPE_SHORT :
      $formatter = new IntlDateFormatter($this->getLocaleCode(),
        IntlDateFormatter::SHORT, IntlDateFormatter::SHORT);
      break;
    case self::DATE_TYPE_MEDIUM :
      $formatter = new IntlDateFormatter($this->getLocaleCode(), IntlDateFormatter::MEDIUM, IntlDateFormatter::MEDIUM);
      break;
    case self::DATE_TYPE_LONG :
      $formatter = new IntlDateFormatter($this->getLocaleCode(), IntlDateFormatter::LONG, IntlDateFormatter::LONG);
      break;
    case self::DATE_TYPE_FULL :
      $formatter = new IntlDateFormatter($this->getLocaleCode(), IntlDateFormatter::FULL, IntlDateFormatter::FULL);
      break;
    default :
      throw new InvalidArgumentException(self::ERROR_CREATE_INTL_DATE_FMT);
  }
  $this->dateFormatter = $formatter;
  return $this->dateFormatter;
}
```

4. 接下来我们定义一个方法来产生一个locale格式的日期。定义传入的`$date`的格式是有点棘手的，它不能是特定于本地的，否则我们将需要根据locale规则来解析它，结果是不可预测的。一个更好的策略是接受一个代表年、月、日等整数的数组。作为后备方案，我们将接受一个字符串，但只接受这种格式。`YYYY-mm-dd HH:ii:ss` 时区是可选的，可以单独设置。首先我们初始化变量。

```php
public function formatDate($date, $type, $timeZone = NULL)
{
  $result   = NULL;
  $year     = date('Y');
  $month    = date('m');
  $day      = date('d');
  $hour     = 0;
  $minutes  = 0;
  $seconds  = 0;
```

5. 之后，我们会产生一个代表年、月、日等数值的明细表。

```php
if (is_string($date)) {
  list($dateParts, $timeParts) = explode(' ', $date);
  list($year,$month,$day) = explode('-',$dateParts);
  list($hour,$minutes,$seconds) = explode(':',$timeParts);
} elseif (is_array($date)) {
  list($year,$month,$day,$hour,$minutes,$seconds) = $date;
} else {
  throw new InvalidArgumentException(self::ERROR_ARGS_STRING_ARRAY);
}
```

6. 接下来我们创建一个 `IntlCalendar` 实例，它将作为运行 `format()` 时的参数。我们使用谨慎的整数值来设置日期。

```php
$intlDate = IntlCalendar::createInstance($timeZone, $this->getLocaleCode());
$intlDate->set($year,$month,$day,$hour,$minutes,$seconds);
```

7. 最后，我们获得日期格式化实例，并产生结果为：

```php
  $formatter = $this->getDateFormatter($type);
  if ($timeZone) {
    $formatter->setTimeZone($timeZone);
  }
  $result = $formatter->format($intlDate);
  return $result ?? self::ERROR_UNABLE_TO_FORMAT;
}
```

8. `parseDate()`方法实际上比格式化更简单。唯一的复杂之处在于，如果没有指定类型，该怎么办（这是最有可能发生的情况）。我们需要做的就是循环浏览所有可能的类型（其中只有四种），直到产生一个结果。

```php
public function parseDate($string, $type = NULL)
{
 if ($type) {
  $result = $this->getDateFormatter($type)->parse($string);
 } else {
  $tryThese = [self::DATE_TYPE_FULL,
    self::DATE_TYPE_LONG,
    self::DATE_TYPE_MEDIUM,
    self::DATE_TYPE_SHORT];
  foreach ($tryThese as $type) {
  $result = $this->getDateFormatter($type)->parse($string);
    if ($result) {
      break;
    }
  }
 }
 return ($result) ? $result : self::ERROR_UNABLE_TO_PARSE;
}
```

## 如何运行...

在`Application\I18n\Locale`中编写代码，前面已经讨论过了。然后可以创建一个测试文件，`chap_08_formatting_date.php`，它设置了自动加载，并创建了两个`Locale`类的实例，一个用于美国，另一个用于法国。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\I18n\Locale;

$localeFr = new Locale('fr-FR');
$localeUs = new Locale('en_US');
$date     = '2016-02-29 17:23:58';
?>
```

接下来，使用合适的样式，运行`formatDate()`和`parseDate()`的测试。

```php
echo $localeFr->formatDate($date, Locale::DATE_TYPE_FULL);
echo $localeUs->formatDate($date, Locale::DATE_TYPE_MEDIUM);
$localeUs->parseDate($localeFr->formatDate($date, Locale::DATE_TYPE_MEDIUM));
// etc.
```

这里是一个输出的例子。

![](../../.gitbook/assets/image%20%2898%29.png)

## 更多...

ISO 8601为日期和时间的所有方面给出了精确的定义。还有一个RFC讨论了ISO 8601对互联网的影响。参考资料，请参见[https://tools.ietf.org/html/rfc3339](https://tools.ietf.org/html/rfc3339)。关于各国的日期格式，请参见[https://en.wikipedia.org/wiki/Date\_format\_by\_country](https://en.wikipedia.org/wiki/Date_format_by_country)。

