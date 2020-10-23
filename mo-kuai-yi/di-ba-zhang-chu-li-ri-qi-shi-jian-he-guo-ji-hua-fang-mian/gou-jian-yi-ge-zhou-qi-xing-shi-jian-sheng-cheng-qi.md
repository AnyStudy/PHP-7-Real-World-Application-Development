# 构建一个周期性事件生成器

与生成日历相关的一个非常常见的需求是活动的安排。事件可以是一次性事件的形式，发生在一天或一个周末。然而，有一个更大的需求是跟踪重复发生的事件。我们需要说明开始日期、重复发生的时间间隔（每天、每周、每月），以及发生的次数或特定的结束日期。

## 如何做...

1.在做任何事情之前，创建一个代表事件的类是一个极好的主意。最终，你可能会将这样一个类中的数据存储在数据库中。然而，在这个例子中，我们将简单地定义这个类，并将数据库方面留给你想象。你会注意到，我们将使用一些包含在`DateTime`扩展中的类，它们非常适用于事件的生成。

```php
namespace Application\I18n;

use DateTime;
use DatePeriod;
use DateInterval;
use InvalidArgumentException;

class Event
{
  // code
}
```

2. 接下来，我们定义了一系列有用的类常量和属性。你会注意到，我们定义了大部分的属性都是公共的，以节省所需的`getter`和`setter`的数量。间隔被定义为`sprintf()`格式的字符串；`%d`将被替换为一个值。

```php
const INTERVAL_DAY = 'P%dD';
const INTERVAL_WEEK = 'P%dW';
const INTERVAL_MONTH = 'P%dM';
const FLAG_FIRST = 'FIRST';    // 1st of the month
const ERROR_INVALID_END  = 'Need to supply either # occurrences or an end date';
const ERROR_INVALID_DATE = 'String i.e. YYYY-mm-dd or DateTime instance only';
const ERROR_INVALID_INTERVAL = 'Interval must take the form "P\d+(D | W | M)"';

public $id;
public $flag;
public $value;
public $title;
public $locale;
public $interval;
public $description;
public $occurrences;
public $nextDate;
protected $endDate;
protected $startDate;
```

3. 接下来我们将注意力转移到构造函数上。我们需要收集和设置与事件相关的所有信息。变量的名称是不言自明的。

{% hint style="info" %}
`$value`就不太清楚了。这个参数最终将被替换为间隔格式字符串中的值。所以，举例来说，如果用户选择`$interva`l为`INTERVAL_DAY`，`$value`为2，那么产生的间隔字符串将是`P2D`，也就是每隔一天（或每隔2天）。
{% endhint %}

```php
public function __construct($title, 
    $description,
    $startDate,
    $interval,
    $value,
    $occurrences = NULL,
    $endDate = NULL,
    $flag = NULL)
{
```

4. 然后我们初始化变量。注意，ID是伪随机生成的，但最终可能会成为数据库事件表中的主键。在这里，我们使用`md5()`不是为了安全，而是为了快速生成一个哈希值，使ID有一个一致的外观。

```php
$this->id = md5($title . $interval . $value) . sprintf('%04d', rand(0,9999));
$this->flag = $flag;
$this->value = $value;
$this->title = $title;
$this->description = $description;
$this->occurrences = $occurrences;
```

5. 如前所述，`interval`参数是一个`sprintf()`模式，用来构造一个合适的`DateInterval`实例。

```php
try {
  $this->interval = new DateInterval(sprintf($interval, $value));
  } catch (Exception $e) {
  error_log($e->getMessage());
  throw new InvalidArgumentException(self::ERROR_INVALID_INTERVAL);
}
```

6. 为了初始化`$startDate`，我们调用`stringOrDate()`。然后我们尝试通过调用`stringOrDate()`或`calcEndDateFromOccurrences()`为`$endDate`生成一个值。如果我们既没有结束日期，也没有出现次数，就会抛出一个异常。

```php
  $this->startDate = $this->stringOrDate($startDate);
  if ($endDate) {
    $this->endDate = $this->stringOrDate($endDate);
  } elseif ($occurrences) {
    $this->endDate = $this->calcEndDateFromOccurrences();
  } else {
  throw new InvalidArgumentException(self::ERROR_INVALID_END);
  }
  $this->nextDate = $this->startDate;
}
```

7.`stringOrDate()`方法由几行代码组成，检查日期变量的数据类型，并返回一个`DateTime`实例或`NULL`。

```php
protected function stringOrDate($date)
{
  if ($date === NULL) { 
    $newDate = NULL;
  } elseif ($date instanceof DateTime) {
    $newDate = $date;
  } elseif (is_string($date)) {
    $newDate = new DateTime($date);
  } else {
    throw new InvalidArgumentException(self::ERROR_INVALID_END);
  }
  return $newDate;
}
```

8.如果设置了`$occurrences`，我们从构造函数中调用`calcEndDateFromOccurrences()`方法，这样我们就可以知道这个事件的结束日期。我们利用`DatePeriod`类，它提供了一个基于开始日期、`DateInterval`和出现次数的迭代。

```php
protected function calcEndDateFromOccurrences()
{
  $endDate = new DateTime('now');
  $period = new DatePeriod(
$this->startDate, $this->interval, $this->occurrences);
  foreach ($period as $date) {
    $endDate = $date;
  }
  return $endDate;
}
```

9. 接下来我们抛出一个`__toString()`的神奇方法，它简单地呼应了事件的标题。

```php
public function __toString()
{
  return $this->title;
}
```

10. 我们需要为`Event`类定义的最后一个方法是`getNextDate()`，它在生成日历时使用。

```php
public function  getNextDate(DateTime $today)
{
  if ($today > $this->endDate) {
    return FALSE;
  }
  $next = clone $today;
  $next->add($this->interval);
  return $next;
}
```

11. 接下来我们将注意力转移到前面的配方中描述的`Application/I18n/Calendar`类。经过一些小手术，我们已经准备好将新定义的`Event`类绑定到日历中。首先，我们添加一个新的属性`$event`s，以及一个以数组形式添加事件的方法。我们使用`Event::$id`属性来确保事件被合并而不被覆盖。

```php
protected $events = array();
public function addEvent(Event $event)
{
  $this->events[$event->id] = $event;
}
```

12. 接下来我们添加一个方法，`processEvents()`，在构建年历时，将一个`Event`实例添加到`Day`对象中。首先我们检查是否有任何事件，以及`Day`对象是否为`NULL`。大家可能还记得，很可能每月的第一天并不在一周的第一天，因此需要将`Day`对象的值设置为`NULL`。我们当然不希望将事件添加到一个非操作性的日子里! 然后我们调用`Event::getNextDate()`，看看日期是否匹配。如果是，我们将事件存储到`Day::$events[]`中，并在Event对象上设置下一个日期。

```php
protected function processEvents($dayObj, $cal)
{
  if ($this->events && $dayObj()) {
    $calDateTime = $cal->toDateTime();
    foreach ($this->events as $id => $eventObj) {
      $next = $eventObj->getNextDate($eventObj->nextDate);
      if ($next) {
        if ($calDateTime->format('Y-m-d') == 
            $eventObj->nextDate->format('Y-m-d')) {
          $dayObj->events[$eventObj->id] = $eventObj;
          $eventObj->nextDate = $next;
        }
      }
    }
  }
  return $dayObj;
}
```

{% hint style="info" %}
请注意，我们不对这两个对象进行直接比较。原因有二：首先，一个是`DateTime`实例，另一个是`IntlCalendar`实例。另一个更有说服力的原因是，有可能在获取`DateTime`实例时包含了小时:分钟:秒，导致两个对象的实际值不同。
{% endhint %}

13. 现在，我们需要在`buildMonthArray()`方法中添加对`processEvents()`的调用，使其看起来像这样。

```php
  while ($day <= $maxDaysInMonth) {
    for ($dow = 1; $dow <= 7; $dow++) {
      // add this to the existing code:
      $dayObj = $this->processEvents(new Day($value), $cal);
      $monthArray[$weekOfYear][$dow] = $dayObj;
    }
  }
```

14.最后，我们需要修改`getWeekDaysRow()`，添加必要的代码，在框内输出事件信息与日期。

```php
protected function getWeekDaysRow($type, $week)
{
  $output = '<tr style="height:' . $this->height . ';">';
  $width  = (int) (100/7);
  foreach ($week as $day) {
    $events = '';
    if ($day->events) {
      foreach ($day->events as $single) {
        $events .= '<br>' . $single->title;
        if ($type == self::DAY_FULL) {
          $events .= '<br><i>' . $single->description . '</i>';
        }
      }
    }
    $output .= '<td style="vertical-align:top;" width="' . $width . '%">' 
  . $day() . $events . '</td>';
  }
  $output .= '</tr>' . PHP_EOL;
  return $output;
}
```

## 如何运行...

要将事件与日历联系起来，首先对步骤1至10中描述的`Application\I18n\Event`类进行编码。接下来，修改`Application\I18n\Calendar`，如步骤11至14所述。然后你可以创建一个测试脚本，`chap_08_recurring_events.php`，设置自动加载并创建`Locale`和`Calendar`实例。为了便于说明，请使用`'es_ES'`作为locale。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\I18n\ { Locale, Calendar, Event };

try {
  $year = 2016;
  $localeEs = new Locale('es_ES');
  $calendarEs = new Calendar($localeEs);
```

现在我们可以开始定义并向日历添加事件了。第一个例子添加了一个持续3天的事件，从2016年1月8日开始。

```php
  // add event: 3 days
  $title = 'Conf';
  $description = 'Special 3 day symposium on eco-waste';
  $startDate = '2016-01-08';
  $event = new Event($title, $description, $startDate, 
                     Event::INTERVAL_DAY, 1, 2);
  $calendarEs->addEvent($event);
```

这里再举一个例子，在2017年9月之前，每个月的1号都会有一个活动。

```php
 $title = 'Pay Rent';
  $description = 'Sent rent check to landlord';
  $startDate = new DateTime('2016-02-01');
  $event = new Event($title, $description, $startDate, 
    Event::INTERVAL_MONTH, 1, '2017-09-01', NULL, Event::FLAG_FIRST);
  $calendarEs->addEvent($event);
```

然后，你可以根据需要添加每周、每两周、每月等样本事件。然后你可以关闭`try...catch`块，并产生合适的显示逻辑。

```php
} catch (Throwable $e) {
  $message = $e->getMessage();
}
?>
<!DOCTYPE html>
<head>
  <title>PHP 7 Cookbook</title>
  <meta http-equiv="content-type" content="text/html;charset=utf-8" />
  <link rel="stylesheet" type="text/css" href="php7cookbook_html_table.css">
</head>
<body>
<h3>Year: <?= $year ?></h3>
<?= $calendarEs->calendarForYear($year, 'Europe/Berlin', 
    Calendar::DAY_3, Calendar::MONTH_FULL, 2); ?>
<?= $calendarEs->calendarForMonth($year, 1  , 'Europe/Berlin', 
    Calendar::DAY_FULL); ?>
</body>
</html>
```

下面是显示今年前几个月的产出。

![](../../.gitbook/assets/image%20%2899%29.png)

## 更多...

有关可以与`get()`一起使用的`IntlCalendar`字段常量的更多信息，请参考本页面：[http://php.net/manual/en/class.intlcalendar.php\#intlcalendar.constants](http://php.net/manual/en/class.intlcalendar.php#intlcalendar.constants)。

