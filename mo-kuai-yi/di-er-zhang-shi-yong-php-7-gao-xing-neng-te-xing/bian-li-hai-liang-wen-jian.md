# 遍历海量文件

像 `file_get_contents()` 和 `file()` 这样的函数使用起来非常方便快捷，但是由于内存的限制，当处理大量的文件时，它们很快就会产生问题。`php.ini` 内存限制的默认设置是128兆字节。因此，任何大于这个值的文件都不会被加载。

在解析海量文件时，另一个需要考虑的问题是函数或类方法产出的速度有多快？ 例如，在生成用户输出时，尽管乍一看似乎可以将输出累加到数组中。 然后，您将立即输出所有内容以提高效率。 不幸的是，这可能会对用户体验产生不利影响。 创建一个**生成器**，并使用 `yield` 关键字产生即时结果可能会更好。

## 如何做...

如前文所述， `file*` 函数（即 `file_get_contents()` ），不适合大文件。原因很简单，这些函数，在某一点上，文件的全部内容都会写入到内存中。因此，本示例的重点是 `f*` 函数（即 `fopen()` ）。

然而，稍有变化的是，我们将不直接使用 `f*` 函数，而是使用**SPL**（**标准PHP库**）中包含的 `SplFileObject` 类。

1.首先，我们定义一个 `Application\Iterator\LargeFile` 类，并赋予相应的属性和常量：

```php
namespace Application\Iterator;

use Exception;
use InvalidArgumentException;
use SplFileObject;
use NoRewindIterator;

class LargeFile
{
  const ERROR_UNABLE = 'ERROR: Unable to open file';
  const ERROR_TYPE   = 'ERROR: Type must be "ByLength", "ByLine" or "Csv"';     
  protected $file;
  protected $allowedTypes = ['ByLine', 'ByLength', 'Csv'];
```

2.然后我们定义了一个 `__construct()` 方法，它接受一个文件名作为参数，并用一个 `SplFileObject` 实例填充 `$file` 属性。如果文件不存在，这也是一个抛出异常的好地方：

```php
public function __construct($filename, $mode = 'r')
{
  if (!file_exists($filename)) {
    $message = __METHOD__ . ' : ' . self::ERROR_UNABLE . PHP_EOL;
    $message .= strip_tags($filename) . PHP_EOL;
    throw new Exception($message);
  }
  $this->file = new SplFileObject($filename, $mode);
}
```

3.接下来我们定义一个 `fileIteratorByLine()` 方法，该方法使用 `fgets()` 一次读取文件的一行。另创建一个 `fileIteratorByLength()` 方法，它可以做同样的事情，但使用 `fread()` 来代替。使用 `fgets()` 的方法适用于包含换行的文本文件。如果解析一个大的二进制文件，可以使用另一个方法：

```php
protected function fileIteratorByLine()
{
  $count = 0;
  while (!$this->file->eof()) {
    yield $this->file->fgets();
    $count++;
  }
  return $count;
}
    
protected function fileIteratorByLength($numBytes = 1024)
{
  $count = 0;
  while (!$this->file->eof()) {
    yield $this->file->fread($numBytes);
    $count++;
  }
  return $count; 
}
```

4.最后，我们定义一个 `getIterator()` 方法，返回一个 `NoRewindIterator()` 实例。这个方法接受 `ByLine` 或 `ByLength` 作为参数，它们指的是上一步定义的两个方法。在调用 `ByLength` 的情况下，这个方法还需要接受 `$numBytes`。我们需要 `NoRewindIterator()` 实例的原因是，在这个例子中，我们只从一个方向读取文件：

```php
public function getIterator($type = 'ByLine', $numBytes = NULL)
{
  if(!in_array($type, $this->allowedTypes)) {
    $message = __METHOD__ . ' : ' . self::ERROR_TYPE . PHP_EOL;
    throw new InvalidArgumentException($message);
  }
  $iterator = 'fileIterator' . $type;
  return new NoRewindIterator($this->$iterator($numBytes));
}
```

## 如何运行...

首先，我们利用第一章《建立基础》中定义的自动加载类，在调用程序中获取 `Application\Iterator\LargeFile` 的实例， `chap_02_iterating_through_a_massive_file.php` ：

```php
define('MASSIVE_FILE', '/../data/files/war_and_peace.txt');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
```

接下来，在 `try {...} catch () {...}` 块里面，我们得到一个 `ByLine` 迭代器的实例：

```php
try {
  $largeFile = new Application\Iterator\LargeFile(__DIR__ . MASSIVE_FILE);
  $iterator = $largeFile->getIterator('ByLine');
```

然后，我们提供一个有用的示例，在这种情况下，定义每行平均单词数：

```php
$words = 0;
foreach ($iterator as $line) {
  echo $line;
  $words += str_word_count($line);
}
echo str_repeat('-', 52) . PHP_EOL;
printf("%-40s : %8d\n", 'Total Words', $words);
printf("%-40s : %8d\n", 'Average Words Per Line', 
($words / $iterator->getReturn()));
echo str_repeat('-', 52) . PHP_EOL;
```

然后，我们结束catch块：

```php
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

预期的输出（太大了，这里就不显示了！）在本示例中我们采用了古腾堡版的《战争与和平》其中有566,095个字。另外，我们发现每行的平均字数是8个。

