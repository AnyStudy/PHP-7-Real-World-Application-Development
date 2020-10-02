# 递归目录迭代器

获取一个目录中的文件列表是非常容易的。传统上，开发人员使用 `glob()` 函数来实现这一目的。要从目录树中的一个特定点递归地获取所有文件和目录的列表就比较麻烦了。这个配方利用了\(**SPL标准PHP库**\)类 `RecursiveDirectoryIterator` ，它可以很好地实现这个目的。

此类的作用是解析目录树，找到第一个子代，然后跟随分支，直到没有更多子代，然后停止！ 不幸的是，这不是我们想要的。 我们需要某种方式让 `RecursiveDirectoryIterator` 从给定的起点继续解析每个树和分支，直到没有更多的文件或目录为止。 碰巧有一个很棒的类 `RecursiveIteratorIterator` 正是这样做的。 通过将 `RecursiveDirectoryIterator` 包装在 `RecursiveIteratorIterator` 中，我们可以完成所有目录树的完整遍历。

> **小贴士**
>
> **警告！**
>
> 在开始文件系统遍历时要非常小心。如果从根目录开始，可能会导致服务器崩溃，因为直到找到所有文件和目录，递归过程才会停止！

## 如何做...

1.首先，我们定义了一个 `Application\Iterator\Directory` 类，其中定义了相应的属性和常量，并使用了外部类：

```php
namespace Application\Iterator;

use Exception;
use RecursiveDirectoryIterator;
use RecursiveIteratorIterator;
use RecursiveRegexIterator;
use RegexIterator;

class Directory
{
        
  const ERROR_UNABLE = 'ERROR: Unable to read directory';
     
  protected $path;
  protected $rdi;
  // 递归目录迭代器
```

2.构造函数基于目录路径在 `RecursiveIteratorIterator` 内部创建一个 `RecursiveDirectoryIterator` 实例：

```php
public function __construct($path)
{
  try {
    $this->rdi = new RecursiveIteratorIterator(
      new RecursiveDirectoryIterator($path),
      RecursiveIteratorIterator::SELF_FIRST);
  } catch (\Throwable $e) {
    $message = __METHOD__ . ' : ' . self::ERROR_UNABLE . PHP_EOL;
    $message .= strip_tags($path) . PHP_EOL;
    echo $message;
    exit;
  }
}
```

3.接下来，我们决定如何处理迭代。一种可能是模仿 Linux 的 `ls -l -R` 命令的输出。请注意，我们使用了 `yield` 关键字，有效地将这个方法变成了一个**Generator**，然后可以从外部调用它。目录迭代产生的每个对象都是一个 `SPL FileInfo` 对象，它可以给我们提供关于文件的有用信息。下面是这个方法的样子：

```php
public function ls($pattern = NULL)
{
  $outerIterator = ($pattern) 
  ? $this->regex($this->rdi, $pattern) 
  : $this->rdi;
  foreach($outerIterator as $obj){
    if ($obj->isDir()) {
      if ($obj->getFileName() == '..') {
        continue;
      }
      $line = $obj->getPath() . PHP_EOL;
    } else {
      $line = sprintf('%4s %1d %4s %4s %10d %12s %-40s' . PHP_EOL,
      substr(sprintf('%o', $obj->getPerms()), -4),
      ($obj->getType() == 'file') ? 1 : 2,
      $obj->getOwner(),
      $obj->getGroup(),
      $obj->getSize(),
      date('M d Y H:i', $obj->getATime()),
      $obj->getFileName());
    }
    yield $line;
  }
}
```

4.你可能已经注意到，方法调用包含了一个文件模式。我们需要一种过滤递归的方法，使其只包含匹配的文件。SPL中有另一个迭代器可以完全满足这个需求： `RegexIterator`类。

```php
protected function regex($iterator, $pattern)
{
  $pattern = '!^.' . str_replace('.', '\\.', $pattern) . '$!';
  return new RegexIterator($iterator, $pattern);
}
```

5.最后，这里还有一种方法，但这次我们将模仿 `dir /s` 命令：

```php
public function dir($pattern = NULL)
{
  $outerIterator = ($pattern) 
  ? $this->regex($this->rdi, $pattern) 
  : $this->rdi;
  foreach($outerIterator as $name => $obj){
      yield $name . PHP_EOL;
    }        
  }
}
```

## 如何运行...

首先，我们利用第一章 《建立基础》中定义的自动加载类，获得 `Application\Iterator\Directory` 的实例，定义一个调用程序， `chap_02_recursive_directory_iterator.php` 。

```php
define('EXAMPLE_PATH', realpath(__DIR__ . '/../'));
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
$directory = new Application\Iterator\Directory(EXAMPLE_PATH);
```

然后，在 `try {...} catch () {...}` 块中，我们使用一个示例目录路径，对我们的两个方法进行调用：

```php
try {
  echo 'Mimics "ls -l -R" ' . PHP_EOL;
  foreach ($directory->ls('*.php') as $info) {
    echo $info;
  }

  echo 'Mimics "dir /s" ' . PHP_EOL;
  foreach ($directory->dir('*.php') as $info) {
    echo $info;
  }
        
} catch (Throwable $e) {
  echo $e->getMessage();
}
```

`ls()` 的输出将是这样的：

![](../../.gitbook/assets/image%20%287%29.png)

`dir()` 的输出将显示如下：

![](../../.gitbook/assets/image%20%288%29.png)

