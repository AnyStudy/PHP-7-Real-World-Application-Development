# 创建一个HTML国际日历生成器

创建一个显示日历的程序是你作为一个中学生最可能做的事情。一个嵌套的for\(\)循环，里面的循环生成一个七天的列表，一般就足够了。即使是一个月有多少天的问题，也可以用一个简单的数组的形式轻松解决。当你需要计算出任何一年的1月1日在一周中的哪一天时，问题就开始变得棘手了。另外，如果你想用一种特定地区可以接受的语言和格式来表示月份和星期几呢？你可能已经猜到了，我们将使用之前讨论过的`Application\I18n\Locale`类来构建一个解决方案。

## 如何做...

1.首先，我们需要创建一个通用类，用来保存单日的信息。最初，它将只保存一个整数值，`$dayOfMonth`。稍后，在下一个事例中，我们将扩展它以包含事件。由于这个类的主要目的是产生 `$dayOfMonth`，我们将在它的构造函数中加入这个值，并定义 `__invoke()` 来返回这个值。

```php
namespace Application\I18n;

class Day
{
  public $dayOfMonth;
  public function __construct($dayOfMonth)
  {
    $this->dayOfMonth = $dayOfMonth;
  }
  public function __invoke()
  {
    return $this->dayOfMonth ?? '';
  }
}
```

2. 创建一个新的类，它将持有适当的日历生成方法。它将接受一个`Application\I18n\Locale`的实例，并定义几个类常量和属性。格式代码，如`EEEEE`和`MMMM`，来自ICU日期格式。

```php
namespace Application\I18n;

use IntlCalendar;

class Calendar
{

  const DAY_1 = 'EEEEE';  // T
  const DAY_2 = 'EEEEEE'; // Tu
  const DAY_3 = 'EEE';   // Tue
  const DAY_FULL = 'EEEE'; // Tuesday
  const MONTH_1 = 'MMMMM'; // M
  const MONTH_3 = 'MMM';  // Mar
  const MONTH_FULL = 'MMMM';  // March
  const DEFAULT_ACROSS = 3;
  const HEIGHT_FULL = '150px';
  const HEIGHT_SMALL = '60px';

  protected $locale;
  protected $dateFormatter;
  protected $yearArray;
  protected $height;

  public function __construct(Locale $locale)
  {
    $this->locale = $locale;
  }

     // other methods are discussed in the following bullets

}
```

3. 然后我们定义一个方法，从我们的`locale`类中返回一个`ItlDateFormatter`实例。这将被存储在一个类属性中，因为它将被频繁使用。

```php
protected function getDateFormatter()
{
 if (!$this->dateFormatter) {
  $this->dateFormatter = $this->locale->getDateFormatter(Locale::DATE_TYPE_FULL);
 }
 return $this->dateFormatter;
}
```

4. 接下来我们定义了一个核心方法，`buildMonthArray()`，它创建了一个多维数组，其中外键是一年的星期，内数组是七个元素，代表一周的天数。我们接受年、月和可选的时区作为参数。注意，作为变量初始化的一部分，我们从月份中减去1。这是因为`IntlCalendar::set()`方法期望月份的值是基于0的，其中0代表1月，1代表2月，以此类推。

```php
public function buildMonthArray($year, $month, $timeZone = NULL)
{
$month -= 1; 
//IntlCalendar months are 0 based; Jan==0, Feb==1 and so on
  $day = 1;
  $first = TRUE;
  $value = 0;
  $monthArray = array();

```

5. 然后我们创建一个 `IntlCalendar` 实例，并使用它来确定这个月有多少天。

```php
$cal = IntlCalendar::createInstance($timeZone, $this->locale->getLocaleCode());
$cal->set($year, $month, $day);
$maxDaysInMonth = $cal->getActualMaximum(IntlCalendar::FIELD_DAY_OF_MONTH);
```

6. 之后，我们使用我们的`IntlDateFormatter`实例来确定一周中的哪一天相当于这个月的1号。之后，我们将模式设置为`w`，随后就会得到周号。

```php
$formatter = $this->getDateFormatter();
$formatter->setPattern('e');
$firstDayIsWhatDow = $formatter->format($cal);
```

7. 现在，我们已经准备好用嵌套循环来循环这个月的所有日子。外层的 `while()` 循环确保我们不会超过月末。内部循环表示一周中的日子。你会注意到，我们利用了 `IntlCalendar::get()`，它允许我们从广泛的预定义字段中获取值。如果年的星期值超过52，我们还将其调整为0。

```php
while ($day <= $maxDaysInMonth) {
  for ($dow = 1; $dow <= 7; $dow++) {
    $cal->set($year, $month, $day);
    $weekOfYear = $cal->get(IntlCalendar::FIELD_WEEK_OF_YEAR);
    if ($weekOfYear > 52) $weekOfYear = 0;
```

8.然后我们检查`$first`是否仍然设置为`TRUE`。如果是的话，我们开始向数组中添加日数。否则，数组的值将被设置为`NULL`。然后我们关闭所有打开的语句并返回数组。请注意，我们还需要确保内部循环不会超过这个月的天数，因此在外层的 `else` 子句中多了一条 `if()` 语句。

{% hint style="info" %}
请注意，我们使用新定义的`Application\I18n\Day`类，而不是仅仅存储本月某一天的值。
{% endhint %}

```php
   if ($first) {
        if ($dow == $firstDayIsWhatDow) {
          $first = FALSE;
          $value = $day++;
        } else {
          $value = NULL;
        }
      } else {
        if ($day <= $maxDaysInMonth) {
          $value = $day++;
        } else {
          $value = NULL;
        }
      }
      $monthArray[$weekOfYear][$dow] = new Day($value);
    }
  }
  return $monthArray;
}
```

### 国际化输出的定义

1.首先是一系列小方法，首先是根据类型提取国际格式化的日子。类型决定了我们是提供当天的全称、缩写还是只提供一个字母，所有这些都适合该地区。

```php
protected function getDay($type, $cal)
{
  $formatter = $this->getDateFormatter();
  $formatter->setPattern($type);
  return $formatter->format($cal);
}
```

2. 接下来我们需要一个方法来返回HTML中的日名行，调用新定义的`getDay()`方法。如前所述，类型决定了日期的外观。

```php
protected function getWeekHeaderRow($type, $cal, $year, $month, $week)
{
  $output = '<tr>';
  $width  = (int) (100/7);
  foreach ($week as $day) {
    $cal->set($year, $month, $day());
    $output .= '<th style="vertical-align:top;" width="' . $width . '%">' . $this->getDay($type, $cal) . '</th>';
  }
  $output .= '</tr>' . PHP_EOL;
  return $output;
}
```

3. 之后，我们定义一个非常简单的方法来返回一行周日期。注意，我们利用`Day::__invoke()`使用：`$day()`。

```php
protected function getWeekDaysRow($week)
{
  $output = '<tr style="height:' . $this->height . ';">';
  $width  = (int) (100/7);
  foreach ($week as $day) {
    $output .= '<td style="vertical-align:top;" width="' . $width . '%">' . $day() .  '</td>';
  }
  $output .= '</tr>' . PHP_EOL;
  return $output;
}
```

4. 最后，一个将小方法组合在一起的方法来生成单个月份的日历。首先，我们建立月份数组，但前提是`$yearArray`还没有可用。

```php
public function calendarForMonth($year, 
    $month, 
    $timeZone = NULL, 
    $dayType = self::DAY_3, 
    $monthType = self::MONTH_FULL, 
    $monthArray = NULL)
{
  $first = 0;
  if (!$monthArray) 
    $monthArray = $this->yearArray[$year][$month]
    ?? $this->buildMonthArray($year, $month, $timeZone);
```

5. 由于 `IntlCalendar` 月份是以 0 为基础的，所以需要将月份递减为 1。Jan = 0，Feb = 1，以此类推。然后，我们使用时区（如果有的话）和`locale`建立一个`InnalCalendar`实例。接下来，我们创建一个 `IntlDateFormatter` 实例，以根据 `locale` 检索月份名称和其他信息。

```php
  $month--;
  $cal = IntlCalendar::createInstance($timeZone, $this->locale->getLocaleCode());
  $cal->set($year, $month, 1);
  $formatter = $this->getDateFormatter();
  $formatter->setPattern($monthType);
```

6. 然后我们在月份数组中循环，并调用刚才提到的较小的方法来建立最终的输出。

```php
 $this->height = ($dayType == self::DAY_FULL) 
     ? self::HEIGHT_FULL : self::HEIGHT_SMALL;
  $html = '<h1>' . $formatter->format($cal) . '</h1>';
  $header = '';
  $body   = '';
  foreach ($monthArray as $weekNum => $week) {
    if ($first++ == 1) {
      $header .= $this->getWeekHeaderRow($dayType, $cal, $year, $month, $week);
    }
    $body .= $this->getWeekDaysRow($dayType, $week);
  }
  $html .= '<table>' . $header . $body . '</table>' . PHP_EOL;
  return $html;
}
```

7. 为了生成一整年的日历，只需要简单的循环1到12个月就可以了。为了方便外部访问，我们首先定义了一个建立年份数组的方法。

```php
public function buildYearArray($year, $timeZone = NULL)
{
  $this->yearArray = array();
  for ($month = 1; $month <= 12; $month++) {
    $this->yearArray[$year][$month] = $this->buildMonthArray($year, $month, $timeZone);
  }
  return $this->yearArray;
}

public function getYearArray()
{
  return $this->yearArray;
}
```

8. 为了生成一个年份的日历，我们定义了一个方法， `calendarForYear()`。如果年份数组还没有建立，我们调用`buildYearArray()`。我们考虑到我们希望跨月显示多少个月历，然后调用 `calendarForMonth()`。

```php
public function calendarForYear($year, 
  $timeZone = NULL, 
  $dayType = self::DAY_1, 
  $monthType = self::MONTH_3, 
  $across = self::DEFAULT_ACROSS)
{
  if (!$this->yearArray) $this->buildYearArray($year, $timeZone);
  $yMax = (int) (12 / $across);
  $width = (int) (100 / $across);
  $output = '<table>' . PHP_EOL;
  $month = 1;
  for ($y = 1; $y <= $yMax; $y++) {
    $output .= '<tr>';
    for ($x = 1; $x <= $across; $x++) {
      $output .= '<td style="vertical-align:top;" width="' . $width . '%">' . $this->calendarForMonth($year, $month, $timeZone, $dayType, $monthType, $this->yearArray[$year][$month++]) . '</td>';
    }
    $output .= '</tr>' . PHP_EOL;
  }
  $output .= '</table>';
  return $output;
}
```

## 如何运行...

首先，确保你在前面的事例中定义了`Application\I18n\Locale`类，然后，在`Application\I18n`文件夹中创建一个新的文件，`Calendar.php`，其中包含本事例中描述的所有方法。

接下来，定义一个调用程序，`chap_08_html_calendar.php`，设置自动加载并创建`Locale`和`Calendar`实例。另外，一定要定义年份和月份。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\I18n\Locale;
use Application\I18n\Calendar;

$localeFr = new Locale('fr-FR');
$localeUs = new Locale('en_US');
$localeTh = new Locale('th_TH');
$calendarFr = new Calendar($localeFr);
$calendarUs = new Calendar($localeUs);
$calendarTh = new Calendar($localeTh);
$year = 2016;
$month = 1;
?>
```

然后，您可以开发适当的视图逻辑来显示不同的日历。例如，您可以包含参数来显示完整的月份和日期名称。

```php
<!DOCTYPE html>
<html>
  <head>
  <title>PHP 7 Cookbook</title>
  <meta http-equiv="content-type" content="text/html;charset=utf-8" />
  <link rel="stylesheet" type="text/css" href="php7cookbook_html_table.css">
  </head>
  <body>
    <h3>Year: <?= $year ?></h3>
    <?= $calendarFr->calendarForMonth($year, $month, NULL, Calendar::DAY_FULL); ?>
    <?= $calendarUs->calendarForMonth($year, $month, NULL, Calendar::DAY_FULL); ?>
    <?= $calendarTh->calendarForMonth($year, $month, NULL, Calendar::DAY_FULL); ?>
  </body>
</html>
```

![](../../.gitbook/assets/image%20%28106%29.png)

经过几次修改，你也可以显示全年的日历。

```php
$localeTh = new Locale('th_TH');
$localeEs = new Locale('es_ES');
$calendarTh = new Calendar($localeTh);
$calendarEs = new Calendar($localeEs);
$year = 2016;
echo $calendarTh->calendarForYear($year);
echo $calendarEs->calendarForYear($year);
```

以下是浏览器输出的西班牙语全年日历。

![](../../.gitbook/assets/image%20%2896%29.png)

## 更多...

有关`IntlDateFormatter::setPattern()`使用的代码的更多信息，请参见本文：[http://userguide.icu-project.org/formatparse/datetime](http://userguide.icu-project.org/formatparse/datetime)。

