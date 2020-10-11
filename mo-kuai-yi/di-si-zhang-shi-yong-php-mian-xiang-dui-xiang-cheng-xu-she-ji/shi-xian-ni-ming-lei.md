# 实现匿名类

PHP 7引入了一个新功能，匿名类。 与匿名函数非常相似，可以将匿名类定义为表达式的一部分，从而创建一个没有名称的类。 匿名类用于需要动态创建对象的情况，使用该对象然后将其丢弃。

## 如何做...

1.`stdClass`的替代方法是定义一个匿名类。 

在定义中，您可以定义任何属性和方法（包括魔术方法）。 在此示例中，我们定义了一个具有两个属性和一个魔术方法 `__construct()` 的匿名类：

```php
$a = new class (123.45, 'TEST') {
  public $total = 0;
  public $test  = '';
  public function __construct($total, $test)
  {
    $this->total = $total;
    $this->test  = $test;
  }
};
```

2.匿名类可以扩展任何类。 

在此示例中，一个匿名类扩展了 `FilterIterator`，并覆盖了 `__construct()` 和 `accept()` 方法。 作为参数，它接受 `ArrayIterator $b`，它表示 10 到 100 的数组，以 10 为增量。第二个参数用作输出的限制：

```php
$b = new ArrayIterator(range(10,100,10));
$f = new class ($b, 50) extends FilterIterator {
  public $limit = 0;
  public function __construct($iterator, $limit)
  {
    $this->limit = $limit;
    parent::__construct($iterator);
  }
  public function accept()
  {
    return ($this->current() <= $this->limit);
  }
};
```

3.匿名类可以实现接口。 

在此示例中，匿名类用于生成 HTML 颜色代码图表。 该类实现内置的PHP `Countable` 接口。 定义了 `count()` 方法，当此类与需要 `Countable` 的方法或函数一起使用时，将调用此方法：

```php
define('MAX_COLORS', 256 ** 3);

$d = new class () implements Countable {
  public $current = 0;
  public $maxRows = 16;
  public $maxCols = 64;
  public function cycle()
  {
    $row = '';
    $max = $this->maxRows * $this->maxCols;
    for ($x = 0; $x < $this->maxRows; $x++) {
      $row .= '<tr>';
      for ($y = 0; $y < $this->maxCols; $y++) {
        $row .= sprintf(
          '<td style="background-color: #%06X;"', 
          $this->current);
        $row .= sprintf(
          'title="#%06X">&nbsp;</td>', 
          $this->current);
        $this->current++;
        $this->current = ($this->current >MAX_COLORS) ? 0 
             : $this->current;
      }
      $row .= '</tr>';
    }
    return $row;
  }
  public function count()
  {
    return MAX_COLORS;
  }
};
```

4. 匿名类可以使用特性。

5.最后这个例子是对前面定义的例子的修改。我们没有定义 `Test` 类，而是定义了匿名类：

```php
$a = new class() {
  use IdTrait, NameTrait {
    NameTrait::setKeyinsteadofIdTrait;
    IdTrait::setKey as setKeyDate;
  }
};
```

## 如何运行...

在匿名类中，你可以定义任何属性或方法。使用前面的例子，你可以定义一个匿名类，它可以接受构造函数的参数，并且可以访问属性。将步骤2中描述的代码放入测试脚本 `chap_04_oop_anonymous_class.php` 中。添加以下这些 `echo` 语句：

```php
echo "\nAnonymous Class\n";
echo $a->total .PHP_EOL;
echo $a->test . PHP_EOL;
```

下面是匿名类的输出。

![](../../.gitbook/assets/image%20%2867%29.png)

为了使用 `FilterIterator`，必须重写 `accept()` 方法。 在此方法中，您定义了将迭代元素包含为输出的条件。 现在继续，并将步骤4中的代码添加到测试脚本中。 然后，您可以添加以下 `echo` 语句以测试匿名类：

```php
echo "\nAnonymous Class Extends FilterIterator\n";
foreach ($f as $item) echo $item . '';
echo PHP_EOL;
```

在此示例中，限制为 50。 原始 `ArrayIterator` 包含一个 10 到 100 的值数组，以 10 为增量，如以下输出所示：

![](../../.gitbook/assets/image%20%2866%29.png)

要了解一个实现了接口的匿名类，请看步骤5和6所示的例子。将这段代码放在一个文件中，`chap_04_oop_anonymous_class_interfaces.php`。

接下来，添加使您可以在HTML颜色图表中进行分页的代码：

```php
$d->current = $_GET['current'] ?? 0;
$d->current = hexdec($d->current);
$factor = ($d->maxRows * $d->maxCols);
$next = $d->current + $factor;
$prev = $d->current - $factor;
$next = ($next <MAX_COLORS) ? $next : MAX_COLORS - $factor;
$prev = ($prev>= 0) ? $prev : 0;
$next = sprintf('%06X', $next);
$prev = sprintf('%06X', $prev);
?>
```

最后，继续把HTML彩图以网页的形式呈现出来：

```php
<h1>Total Possible Color Combinations: <?= count($d); ?></h1>
<hr>
<table>
<?= $d->cycle(); ?>
</table>	
<a href="?current=<?= $prev ?>"><<PREV</a>
<a href="?current=<?= $next ?>">NEXT >></a>
```

请注意，您可以通过将匿名类的实例传递到 `count()` 函数中来利用 `Countable` 接口（如标签之间所示）。下面是浏览器窗口中显示的输出：

![](../../.gitbook/assets/image%20%2863%29.png)

最后，为了说明特性在匿名类中的使用，请将上一节中提到的`chap_04_oop_trait_multiple.php` 文件复制到新文件 `chap_04_oop_trait_anonymous_class.php` 中。 删除 `Test` 类的定义，并将其替换为匿名类：

```php
$a = new class() {
  use IdTrait, NameTrait {
    NameTrait::setKeyinsteadofIdTrait;
    IdTrait::setKey as setKeyDate;
  }
};
```

删除此行：

```php
$a = new Test();
```

运行代码时，您将看到与前面的屏幕快照完全相同的输出，但类引用将是匿名的：

![](../../.gitbook/assets/image%20%2862%29.png)

