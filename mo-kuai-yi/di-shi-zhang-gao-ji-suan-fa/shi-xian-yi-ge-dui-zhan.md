# 实现一个堆栈

堆栈是一种简单的算法，通常实现为Last In First Out（LIFO）。想象一下，图书馆的桌子上放着一摞书。当图书管理员去把书恢复原位时，最上面的书先被处理，依次类推，直到堆栈底部的书被替换掉。最上面的书是最后一本放在书堆上的，因此是最后进先出。

在编程中，堆栈是用来临时存储信息的。检索顺序便于先检索最近的项目。

## 如何做...

1.首先我们定义一个类，`Application\Generic\Stack`。核心逻辑被封装在一个SPL类，`SplStack` 中。

```php
namespace Application\Generic;
use SplStack;
class Stack
{
  // code
}
```

2. 接下来我们定义一个属性来表示栈，并设置一个 `SplStack` 实例。

```php
protected $stack;
public function __construct()
{
  $this->stack = new SplStack();
}
```

3. 之后我们定义了从堆栈中添加和删除的方法，即经典的 `push()` 和 `pop()` 方法。

```php
public function push($message)
{
  $this->stack->push($message);
}
public function pop()
{
  return $this->stack->pop();
}
```

4. 我们还抛出了一个`__invoke()`的实现，它可以返回栈属性的实例。这允许我们在直接函数调用中使用对象。

```php
public function __invoke()
{
  return $this->stack;
}
```

## 如何运行...

堆栈的一个可能用途是存储消息。在消息的情况下，通常希望先检索最新的消息，因此这是堆栈的一个完美用例。如本示例中所讨论的那样，定义 `Application\Generic\Stack` 类。接下来，定义一个调用程序，设置自动加载并创建一个栈的实例。

```php
<?php
// setup class autoloading
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Generic\Stack;
$stack = new Stack();
```

要对堆栈做一些事情，存储一系列的消息。由于你很可能会在应用程序的不同点存储消息，你可以使用`sleep()` 来模拟其他代码的运行。

```php
echo 'Do Something ... ' . PHP_EOL;
$stack->push('1st Message: ' . date('H:i:s'));
sleep(3);

echo 'Do Something Else ... ' . PHP_EOL;
$stack->push('2nd Message: ' . date('H:i:s'));
sleep(3);

echo 'Do Something Else Again ... ' . PHP_EOL;
$stack->push('3rd Message: ' . date('H:i:s'));
sleep(3);
```

最后，简单地在堆栈中迭代以检索消息。注意，你可以像调用函数一样调用栈对象，它将返回 `SplStack` 实例。

```php
echo 'What Time Is It?' . PHP_EOL;
foreach ($stack() as $item) {
  echo $item . PHP_EOL;
}
```

下面是预期的输出。

![](../../.gitbook/assets/image%20%28131%29.png)

