# 编写测试套件

你可能已经注意到了，在阅读完前面的事例后，手动运行phpunit并指定测试类和PHP文件名会很快变得乏味。尤其是在处理有几十个甚至上百个类和文件的应用程序时，这种情况更加明显。PHPUnit 项目有一个内置的功能，可以用一条命令处理多个测试。这样一组测试被称为测试套件。

## 如何做...

1.最简单的是，你需要做的就是把所有的测试移到一个单独的文件夹里。

```php
mkdir tests
cp *Test.php tests
```

2. 你需要调整包含或需要外部文件的命令，以说明新的位置。所示的例子（`SimpleTest`）是在前面的事例中开发的。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/../chap_13_unit_test_simple.php';

class SimpleTest extends TestCase
{
  // etc.
```

3. 然后你可以简单地以目录路径为参数运行`phpunit`。PHPUnit 会自动运行该目录下的所有测试。在这个例子中，我们假设有一个测试子目录。

```bash
phpunit tests
```

4. 你可以使用`--bootstrap`选项来指定一个在运行测试之前执行的文件。这个选项的一个典型用途是启动自动加载。

```bash
phpunit --boostrap tests_with_autoload/bootstrap.php tests
```

5. 这里是实现自动加载的示例`bootstrap.php`文件。

```php
<?php
require __DIR__ . '/../../Application/Autoload/Loader.php';
Application\Autoload\Loader::init([__DIR__]);
```

6. 另一种可能性是使用 XML 配置文件定义一个或多个测试集。下面是一个只运行Simple\*测试的例子。

```markup
<phpunit>
  <testsuites>
    <testsuite name="simple">
      <file>SimpleTest.php</file>
      <file>SimpleDbTest.php</file>
      <file>SimpleClassTest.php</file>
    </testsuite>
  </testsuites>
</phpunit>
```

7. 下面是另一个基于目录运行测试的例子，也指定了一个引导文件。

```markup
<phpunit bootstrap="bootstrap.php">
  <testsuites>
    <testsuite name="visitor">
      <directory>Simple</directory>
    </testsuite>
  </testsuites>
</phpunit>
```

## 如何运行...

确保所有的测试都已经定义好了，在前面的 "编写一个简单的测试 "中已经讨论过了，然后你可以创建一个测试文件夹，并将所有的`Test.php`_文件移动或复制到这个文件夹中。然后你可以创建一个测试文件夹，并将所有`Test.php`_文件移动或复制到这个文件夹中。然后你需要调整_\`_require\_once\`语句中的路径，如步骤2所示。

```bash
phpunit tests
```

你应该看到以下输出。

![](../../.gitbook/assets/image%20%28151%29.png)

为了演示通过`bootstrap`文件来使用自动加载，创建一个新的`test_with_autoload`目录。在这个文件夹中，定义一个`bootstrap.php`文件，代码如步骤5所示。在 `tests_with_autoload` 中创建两个目录。`Demo`和`Simple`。

从包含本章源代码的目录中，将该文件\(在前面的第12步中讨论过\)复制到`test_with_autoload/Demo/Demo.php`中。在开头的`<?php`标签后，添加这行。

```php
namespace Demo;
```

接下来，复制`SimpleTest.php`文件到 `tests_with_autoload/Simple/ClassTest.php`。\(注意文件名的变化!\). 你需要将前几行修改为以下内容:

```php
<?php
namespace Simple;
use Demo\Demo;
use PHPUnit\Framework\TestCase;

class ClassTest extends TestCase
{
  protected $demo;
  public function setup()
  {
    $this->demo = new Demo();
  }
// etc.
```

之后，创建一个 `tests_with_autoload/phpunit.xml` 文件，把所有的东西整合在一起。

```markup
<phpunit bootstrap="bootstrap.php">
  <testsuites>
    <testsuite name="visitor">
      <directory>Simple</directory>
    </testsuite>
  </testsuites>
</phpunit>
```

最后，改到包含本章代码的目录。现在你可以运行一个单元测试，其中包含一个bootstrap文件，以及自动加载和命名空间，如下所示。

```bash
phpunit -c tests_with_autoload/phpunit.xml
```

输出结果应如下所示：

![](../../.gitbook/assets/image%20%28170%29.png)

## 更多...

关于编写PHPUnit测试套件的更多信息，请看这个文档页：https://phpunit.de/manual/current/en/phpunit-book.html\#organizing-tests.xml-configuration。

