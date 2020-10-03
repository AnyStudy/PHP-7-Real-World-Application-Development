# 使用 PHP 7 增强功能提高性能

一个趋势是，开发者开始使用匿名函数。当处理匿名函数时，一个典型的问题是，如何将它们写成任何对象都可以绑定到 `$this` 的形式，并且函数仍然可以工作。在 PHP 5 代码中使用的方法是使用  `bindTo()` 。在 PHP 7 中，增加了一个新的方法 `call()` ，它提供了类似的功能，但性能有很大的提升。

## 如何做...

为了利用 `call()` ，在一个冗长的循环中执行一个匿名函数。在这个例子中，我们将演示一个匿名函数，它可以扫描一个日志文件，根据出现的频率识别 IP 地址：

1.首先，我们定义一个 `Application\Web\Access` 类。 在构造函数中，我们接受文件名作为参数。日志文件通过 `SplFileObject` 打开，并分配给 `$this->log` ：

```php
Namespace Application\Web;

use Exception;
use SplFileObject;
class Access
{
  const ERROR_UNABLE = 'ERROR: unable to open file';
  protected $log;
  public $frequency = array();
  public function __construct($filename)
  {
    if (!file_exists($filename)) {
      $message = __METHOD__ . ' : ' . self::ERROR_UNABLE . PHP_EOL;
      $message .= strip_tags($filename) . PHP_EOL;
      throw new Exception($message);
    }
    $this->log = new SplFileObject($filename, 'r');
  }
```

2.接下来，我们定义一个生成器，逐行遍历文件：

```php
public function fileIteratorByLine()
{
  $count = 0;
  while (!$this->log->eof()) {
    yield $this->log->fgets();
    $count++;
  }
  return $count;
}
```

3. 最后，我们定义了一个寻找并提取一个IP地址作为子匹配的方法：

```php
public function getIp($line)
{
  preg_match('/(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})/', $line, $match);
  return $match[1] ?? '';
  }
}
```

## 如何运行...

首先，我们定义一个调用程序 `chap_02_performance_using_php7_enchancement_call.php` ，该程序利用第一章《建立基础》中定义的自动加载类来获取 `Application\Web\Access` 的实例：

```php
define('LOG_FILES', '/var/log/apache2/*access*.log');
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
```

接下来我们定义匿名函数，处理日志文件中的一行。如果检测到一个IP地址，它就会成为 `$frequency` 数组中的一个键，并且这个键的当前值会递增：

```php
// 定义方法
$freq = function ($line) {
  $ip = $this->getIp($line);
  if ($ip) {
    echo '.';
    $this->frequency[$ip] = 
    (isset($this->frequency[$ip])) ? $this->frequency[$ip] + 1 : 1;
  }
};
```

然后，我们在每个找到的日志文件中循环迭代行，处理IP地址：

```php
foreach (glob(LOG_FILES) as $filename) {
  echo PHP_EOL . $filename . PHP_EOL;
  // 存取类
  $access = new Application\Web\Access($filename);
  foreach ($access->fileIteratorByLine() as $line) {
    $freq->call($access, $line);
  }
}
```

> **小贴士**
>
> 实际上在 PHP 5 中也可以做同样的事情，但是需要两行代码。
>
> ```php
> $func = $freq->bindTo($access);
> $func($line);
> ```
>
> 性能比在 PHP 7 中使用 `call()` 慢20%到50%。

最后，我们对数组进行反向排序，但保留键。然后通过简单的 `foreach()` 循环输出：

```php
arsort($access->frequency);
foreach ($access->frequency as $key => $value) {
  printf('%16s : %6d' . PHP_EOL, $key, $value);
}
```

他的输出会根据你处理的 `access.log` 而有所不同。下面是一个例子：

![](../../.gitbook/assets/image%20%2822%29.png)

## 更多...

许多 PHP 7 的性能改进与新特性和功能无关。相反，它们采取的是内部改进的形式，在开始运行程序之前是看不见的。以下是属于这一类的改进的简短列表：

| 功能 | 更多信息 | 注释 |
| :--- | :--- | :--- |
| 快速参数解析 | [https://wiki.php.net/rfc/fast\_zpp](https://wiki.php.net/rfc/fast_zpp) | 在 PHP 5 中，每次调用函数时都要对提供给函数的参数进行解析。参数以字符串的形式传入，并以类似于 `scanf()` 函数的方式进行解析。在 PHP 7 中，这个过程得到了优化，变得更加高效，从而使性能得到了显著的提高。这种改进很难衡量，但似乎在6%左右。 |
| PHP NG | [https://wiki.php.net/rfc/phpng](https://wiki.php.net/rfc/phpng) | PHP NG（下一代）计划代表了PHP语言的大部分重写。它保留了现有的功能，但包含了所有可以想象到的节省时间和提高效率的措施。数据结构被压缩，内存被更有效地使用。仅仅是一个影响数组处理的变化，就使性能得到了显著的提高，同时大大减少了内存的使用。 |
| 删除无效的扩展 | [https://wiki.php.net/rfc/removal\_of\_dead\_sapis\_and\_exts](https://wiki.php.net/rfc/removal_of_dead_sapis_and_exts) | 大约有二十多个扩展属于这些类别中的一种：废弃的、不再维护的、未维护的依赖关系、或未移植到 PHP 7。经过核心开发人员的投票，决定删除 2/3 或 "短名单 "上的扩展。这样做的结果是减少了开销，加快了PHP语言未来的整体开发速度。 |







