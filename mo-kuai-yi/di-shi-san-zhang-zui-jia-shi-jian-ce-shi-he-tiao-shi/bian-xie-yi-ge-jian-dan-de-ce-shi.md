# 编写一个简单的测试

测试PHP代码的主要手段是使用PHPUnit，它是基于一种叫做单元测试的方法论。单元测试背后的理念非常简单：将代码分解成尽可能小的逻辑单元。然后对每个单元进行隔离测试，以确认它的性能符合预期。这些预期被编纂成一系列的断言。如果所有的断言都返回 "true"，那么这个单元就通过了测试。

{% hint style="info" %}
在程序化PHP中，单元是一个函数。对于OOP PHP来说，单元是一个类中的方法。
{% endhint %}

## 如何做...

1.首先要做的是直接将 PHPUnit 安装到你的开发服务器上，或者下载源码，源码以单个 phar（PHP 档案）文件的形式存在。快速访问PHPUnit的官方网站\(https://phpunit.de/\)，我们可以直接从主页下载。

2. 然而，最好的做法是使用一个包管理器来安装和维护 PHPUnit。为此，我们将使用一个名为 Composer 的软件包管理程序。要安装 Composer，请访问主网站 https://getcomposer.org/，并按照下载页面的说明进行安装。目前的程序，在写这篇文章的时候，如下所示。注意，你需要用当前版本的哈希值代替`<hash>`。

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '<hash>') { 
    echo 'Installer verified'; 
} else { 
    echo 'Installer corrupt'; unlink('composer-setup.php'); 
} echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
```

{% hint style="info" %}
**最佳实践**

使用Composer这样的软件包管理程序的好处是，它不仅可以安装，还可以用来更新你的应用程序所使用的任何外部软件（如PHPUnit）。
{% endhint %}

3. 接下来，我们使用Composer来安装PHPUnit。这是通过创建一个`composer.json`文件来完成的，该文件包含一系列概述项目参数和依赖关系的指令。对这些指令的完整描述超出了本书的范围；然而，为了本实例的目的，我们使用关键参数 `require` 创建了一组最小的指令。你还会注意到，文件的内容是**JavaScript对象符号（JSON）**格式。

```javascript
{
  "require-dev": {
    "phpunit/phpunit": "*"
  }
}
```

4. 要从命令行进行安装，我们运行以下命令。后面的输出就会显示出来。

```bash
php composer.phar install
```

![](../../.gitbook/assets/image%20%28197%29.png)

5. PHPUnit 和它的依赖项被放置在一个 `vendor` 文件夹中，如果该文件夹不存在，Composer 将会创建它。然后，调用 PHPUnit 的主要命令被符号化地链接到 `vendor/bin` 文件夹中。如果你把这个文件夹放在你的 `PATH` 中，你所需要做的就是运行这个命令，它将检查版本并顺便确认安装。

```bash
phpunit --version
```

### 运行简单的测试

1.为了便于说明，我们假设我们有一个包含`add()`函数的`chap_13_unit_test_simple.php`文件。

```php
<?php
function add($a = NULL, $b = NULL)
{
  return $a + $b;
}
```

2. 然后将测试写成扩展`PHPUnit\Framework\TestCase`的类。如果你要测试一个函数库，在测试类的开头，包括包含函数定义的文件。然后，你会写出以`test`开头的方法，通常后面是你要测试的函数的名称，可能还有一些额外的`CamelCase`词来进一步描述测试。在本示例中，我们将定义一个`SimpleTest`测试类。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/chap_13_unit_test_simple.php';
class SimpleTest extends TestCase
{
  // testXXX() methods go here
}
```

3. 断言是任何测试集的核心。一个断言是一个PHPUnit方法，它将一个已知的值和你想测试的值进行比较。一个例子是 `assertEquals()`，它检查第一个参数是否等于第二个参数。下面的例子测试了一个名为 `add()` 的方法，并确认 2 是 `add(1,1)` 的返回值。

```php
public function testAdd()
{
  $this->assertEquals(2, add(1,1));
}
```

4. 你也可以测试一下某件事情是否不真实。这个例子断言1+1不等于3。

```php
$this->assertNotEquals(3, add(1,1));
```

5. `assertRegExp()`是一个在测试字符串时非常有用的断言。在这个例子中，假设我们正在测试一个从一个多维数组中生成一个HTML表格的函数。

```php
function table(array $a)
{
  $table = '<table>';
  foreach ($a as $row) {
    $table .= '<tr><td>';
    $table .= implode('</td><td>', $row);
    $table .= '</td></tr>';
  }
  $table .= '</table>';
  return $table;
}
```

6. 我们可以构造一个简单的测试，以确认输出包含`<table>`，一个或多个字符，然后是`</table>`。此外，我们希望确认元素`<td>B</td>`存在。在编写测试时，我们建立一个由三个子数组组成的测试数组，其中包含字母A-C、D-F和G-I。然后我们将测试数组传递给函数，并针对结果运行断言。

```php
public function testTable()
{
  $a = [range('A', 'C'),range('D', 'F'),range('G','I')];
  $table = table($a);
  $this->assertRegExp('!^<table>.+</table>$!', $table);
  $this->assertRegExp('!<td>B</td>!', $table);
}
```

7. 要测试一个类，不需要包含一个函数库，只需要包含定义要测试的类的文件。为了便于说明，让我们把前面显示的函数库移到Demo类中。

```php
<?php
class Demo
{
  public function add($a, $b)
  {
    return $a + $b;
  }

  public function sub($a, $b)
  {
    return $a - $b;
  }
  // etc.
}
```

8. 在我们的`SimpleClassTest`测试类中，我们不包含库文件，而是包含代表`Demo`类的文件。为了运行测试，我们需要一个`Demo`的实例。为此，我们使用了一个专门设计的`setup()`方法，它在每次测试之前都会运行。另外，你会注意到一个 `teardown()`方法，它是在每次测试后立即运行的。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/Demo.php';
class SimpleClassTest extends TestCase
{
  protected $demo;
  public function setup()
  {
    $this->demo = new Demo();
  }
  public function teardown()
  {
    unset($this->demo);
  }
  public function testAdd()
  {
    $this->assertEquals(2, $this->demo->add(1,1));
  }
  public function testSub()
  {
    $this->assertEquals(0, $this->demo->sub(1,1));
  }
  // etc.
}
```

{% hint style="info" %}
之所以在每次测试前后运行`setup()`和`trapdown()`，是为了保证一个新鲜的测试环境。这样，一个测试的结果就不会影响另一个测试的结果。
{% endhint %}

### 测试数据库模型类

1.当测试一个有数据库访问权限的类（如Model类）时，其他的考虑因素也在发挥作用。主要的考虑是，你应该针对测试数据库，而不是生产中使用的真实数据库来运行测试。最后一点是，通过使用测试数据库，你可以事先用适当的、受控的数据填充它，`setup()`和 `teardown()`也可以用来添加或删除测试数据。

2. 作为一个使用数据库的类的例子，我们将定义一个类 `VisitorOps`。这个新类将包括添加、删除和查找访客的方法。请注意，我们还添加了一个方法来返回最新执行的SQL语句。

```php
<?php
require __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
class VisitorOps
{

const TABLE_NAME = 'visitors';
protected $connection;
protected $sql;

public function __construct(array $config)
{
  $this->connection = new Connection($config);
}

public function getSql()
{
  return $this->sql;
}

public function findAll()
{
  $sql = 'SELECT * FROM ' . self::TABLE_NAME;
  $stmt = $this->runSql($sql);
  while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    yield $row;
  }
}

public function findById($id)
{
  $sql = 'SELECT * FROM ' . self::TABLE_NAME;
  $sql .= ' WHERE id = ?';
  $stmt = $this->runSql($sql, [$id]);
  return $stmt->fetch(PDO::FETCH_ASSOC);
}

public function removeById($id)
{
  $sql = 'DELETE FROM ' . self::TABLE_NAME;
  $sql .= ' WHERE id = ?';
  return $this->runSql($sql, [$id]);
}

public function addVisitor($data)
{
  $sql = 'INSERT INTO ' . self::TABLE_NAME;
  $sql .= ' (' . implode(',',array_keys($data)) . ') ';
  $sql .= ' VALUES ';
  $sql .= ' ( :' . implode(',:',array_keys($data)) . ') ';
  $this->runSql($sql, $data);
  return $this->connection->pdo->lastInsertId();
}

public function runSql($sql, $params = NULL)
{
  $this->sql = $sql;
  try {
      $stmt = $this->connection->pdo->prepare($sql);
      $result = $stmt->execute($params);
  } catch (Throwable $e) {
      error_log(__METHOD__ . ':' . $e->getMessage());
      return FALSE;
  }
  return $stmt;
}
}
```

3. 对于涉及数据库的测试，建议使用测试数据库而不是实时生产数据库。相应地，你将需要一组额外的数据库连接参数，可以用来在`setup()`方法中建立数据库连接。

4. 有可能你希望建立一个一致的样本数据块。这可以在`setup()`方法中插入到测试数据库中。

5. 最后，你可能希望在每次测试后重置测试数据库，这在 `teardown()` 方法中完成。

### 使用MOCK类

1.在某些情况下，测试将访问需要外部资源的复杂组件。一个例子是需要访问数据库的服务类。在测试套件中尽量减少数据库访问是一个最佳实践。另一个考虑因素是，我们不是在测试数据库访问；我们只是在测试一个特定类的功能。因此，有时有必要定义模拟类，模仿其父类的行为，但限制对外部资源的访问。

{% hint style="info" %}
**最佳实践**

在你的测试中，将实际的数据库访问限制在Model（或同等的）类中。否则，运行整套测试所需的时间可能会过长。
{% endhint %}

2. 在这种情况下，为了说明问题，定义一个服务类`VisitorService`，它使用了前面讨论的`VisitorOps`类。

```php
<?php
require_once __DIR__ . '/VisitorOps.php';
require_once __DIR__ . '/../Application/Database/Connection.php';
use Application\Database\Connection;
class VisitorService
{
  protected $visitorOps;
  public function __construct(array $config)
  {
    $this->visitorOps = new VisitorOps($config);
  }
  public function showAllVisitors()
  {
    $table = '<table>';
    foreach ($this->visitorOps->findAll() as $row) {
      $table .= '<tr><td>';
      $table .= implode('</td><td>', $row);
      $table .= '</td></tr>';
    }
    $table .= '</table>';
    return $table;
  }
```

3. 为了测试的目的，我们为`$visitorOps`属性添加一个getter和setter。这允许我们插入一个模拟类来代替真正的`VisitorOps`类。

```php
public function getVisitorOps()
{
  return $this->visitorOps;
}

public function setVisitorOps(VisitorOps $visitorOps)
{
  $this->visitorOps = $visitorOps;
}
} // closing brace for VisitorService
```

4. 接下来，我们定义一个`VisitorOpsMock`模拟类，模仿其父类的功能。类的常量和属性都是继承的。然后我们添加模拟测试数据，以及一个getter，以备以后需要访问测试数据时使用。

```php
<?php
require_once __DIR__ . '/VisitorOps.php';
class VisitorOpsMock extends VisitorOps
{
  protected $testData;
  public function __construct()
  {
    $data = array();
    for ($x = 1; $x <= 3; $x++) {
      $data[$x]['id'] = $x;
      $data[$x]['email'] = $x . 'test@unlikelysource.com';
      $data[$x]['visit_date'] = 
        '2000-0' . $x . '-0' . $x . ' 00:00:00';
      $data[$x]['comments'] = 'TEST ' . $x;
      $data[$x]['name'] = 'TEST ' . $x;
    }
    $this->testData = $data;
  }
  public function getTestData()
  {
    return $this->testData;
  }
```

5. 接下来，我们重写`findAll()`来使用`yield`返回测试数据，就像在父类中一样。请注意，我们仍然构建SQL字符串，因为这是父类的工作。

```php
public function findAll()
{
  $sql = 'SELECT * FROM ' . self::TABLE_NAME;
  foreach ($this->testData as $row) {
    yield $row;
  }
}
```

6. 为了模拟`findById()`，我们简单地从`$this->testData`返回数组键。对于`removeById()`，我们取消设置从`$this->testData`中提供的数组键作为参数。

```php
public function findById($id)
{
  $sql = 'SELECT * FROM ' . self::TABLE_NAME;
  $sql .= ' WHERE id = ?';
  return $this->testData[$id] ?? FALSE;
}
public function removeById($id)
{
  $sql = 'DELETE FROM ' . self::TABLE_NAME;
  $sql .= ' WHERE id = ?';
  if (empty($this->testData[$id])) {
      return 0;
  } else {
      unset($this->testData[$id]);
      return 1;
  }
}
```

7. 添加数据稍微复杂一些，因为我们需要模拟一个事实，即`id`参数可能没有被提供，因为数据库通常会自动为我们生成这个参数。为了解决这个问题，我们检查`id`参数。如果没有设置，我们找到最大的数组键，然后递增。

```php
public function addVisitor($data)
{
  $sql = 'INSERT INTO ' . self::TABLE_NAME;
  $sql .= ' (' . implode(',',array_keys($data)) . ') ';
  $sql .= ' VALUES ';
  $sql .= ' ( :' . implode(',:',array_keys($data)) . ') ';
  if (!empty($data['id'])) {
      $id = $data['id'];
  } else {
      $keys = array_keys($this->testData);
      sort($keys);
      $id = end($keys) + 1;
      $data['id'] = $id;
  }
    $this->testData[$id] = $data;
    return 1;
  }

} // ending brace for the class VisitorOpsMock
```

### 使用匿名类作为模拟对象

1.关于mock对象的一个很好的变化是使用新的PHP 7匿名类来代替创建一个定义mock功能的正式类。使用匿名类的好处是可以扩展一个现有的类，这使得对象看起来合法。如果你只需要覆盖一两个方法，这种方法特别有用。

2. 在这个例子中，我们将修改之前介绍的`VisitorServiceTest.php`，将其称为`VisitorServiceTestAnonClass.php`。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/VisitorService.php';
require_once __DIR__ . '/VisitorOps.php';
class VisitorServiceTestAnonClass extends TestCase
{
  protected $visitorService;
  protected $dbConfig = [
    'driver'   => 'mysql',
    'host'     => 'localhost',
    'dbname'   => 'php7cookbook_test',
    'user'     => 'cook',
    'password' => 'book',
    'errmode'  => PDO::ERRMODE_EXCEPTION,
  ];
    protected $testData;
```

3. 您会注意到，在`setup()`中，我们定义了一个匿名类，该类扩展了`VisitorOps`。我们只需要重写`findAll()`方法。

```php
public function setup()
{
  $data = array();
  for ($x = 1; $x <= 3; $x++) {
    $data[$x]['id'] = $x;
    $data[$x]['email'] = $x . 'test@unlikelysource.com';
    $data[$x]['visit_date'] = 
      '2000-0' . $x . '-0' . $x . ' 00:00:00';
    $data[$x]['comments'] = 'TEST ' . $x;
    $data[$x]['name'] = 'TEST ' . $x;
  }
  $this->testData = $data;
  $this->visitorService = 
    new VisitorService($this->dbConfig);
  $opsMock = 
    new class ($this->testData) extends VisitorOps {
      protected $testData;
      public function __construct($testData)
      {
        $this->testData = $testData;
      }
      public function findAll()
      {
        return $this->testData;
      }
    };
    $this->visitorService->setVisitorOps($opsMock);
}
```

4. 请注意，在`testShowAllVisitors()`中，当`$this->visitorService->showAllVisitors()`被执行时，匿名类会被访问者服务调用，而访问者服务又会调用重写的`findAll()`。

```php
public function teardown()
{
  unset($this->visitorService);
}
public function testShowAllVisitors()
{
  $result = $this->visitorService->showAllVisitors();
  $this->assertRegExp('!^<table>.+</table>$!', $result);
  foreach ($this->testData as $key => $value) {
    $dataWeWant = '!<td>' . $key . '</td>!';
    $this->assertRegExp($dataWeWant, $result);
  }
}
}
```

### 使用MOCK BUILDER

1.另一种技术是使用`getMockBuilder()`。虽然这种方法不允许对产生的mock对象进行大量的有限控制，但在你只需要确认返回某个类的对象，并且当运行指定的方法时，这个方法会返回一些预期的值的情况下，这种方法是非常有用的。

2. 在下面的示例中，我们复制了`VisitorServiceTestAnonClass`；唯一的区别在于如何在`setup()`中提供`VisitorOps`的实例，在本例中，使用`getMockBuilder()`。请注意，虽然我们在这个例子中没有使用`with()`，但它是用来将受控参数馈送到模拟方法的。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/VisitorService.php';
require_once __DIR__ . '/VisitorOps.php';
class VisitorServiceTestAnonMockBuilder extends TestCase
{
  // code is identical to VisitorServiceTestAnon
  public function setup()
  {
    $data = array();
    for ($x = 1; $x <= 3; $x++) {
      $data[$x]['id'] = $x;
      $data[$x]['email'] = $x . 'test@unlikelysource.com';
      $data[$x]['visit_date'] = 
        '2000-0' . $x . '-0' . $x . ' 00:00:00';
      $data[$x]['comments'] = 'TEST ' . $x;
      $data[$x]['name'] = 'TEST ' . $x;
  }
  $this->testData = $data;
    $this->visitorService = 
      new VisitorService($this->dbConfig);
    $opsMock = $this->getMockBuilder(VisitorOps::class)
                    ->setMethods(['findAll'])
                    ->disableOriginalConstructor()
                    ->getMock();
                    $opsMock->expects($this->once())
                    ->method('findAll')
                    ->with()
                    ->will($this->returnValue($this->testData));
                    $this->visitorService->setVisitorOps($opsMock);
  }
  // remaining code is the same
}
```

{% hint style="info" %}
我们已经展示了如何创建简单的一次性测试。然而，在大多数情况下，你会有许多需要测试的类，最好是一次全部测试。这可以通过开发一个测试套件来实现，在下一个事例中会有更详细的讨论。
{% endhint %}

## 如何运行...

首先，你需要安装 PHPUnit，如步骤 1 至 5 所述。 确保在 PATH 中包含 `vendor/bin`，这样你就可以从命令行运行 PHPUnit。

### 运行简单的测试

接下来，定义一个`chap_13_unit_test_simple.php`程序文件，其中包含一系列简单的函数，如步骤1中讨论的`add()`、`sub()`等。然后你可以定义一个简单的测试类，包含在`SimpleTest.php`中，如步骤2和步骤3中提到的。

假设`phpunit`在你的`PATH`中，从终端窗口，改变到包含为这个配方开发的代码的目录，并运行以下命令。

```bash
phpunit SimpleTest SimpleTest.php
```

你应该看到以下输出。

![](../../.gitbook/assets/image%20%28181%29.png)

在`SimpleTest.php`中进行修改，使测试失败（第4步）。

```php
public function testDiv()
{
  $this->assertEquals(2, div(4, 2));
  $this->assertEquals(99, div(4, 0));
}
```

这是修订后的产出。

![](../../.gitbook/assets/image%20%28194%29.png)

接下来，在`chap_13_unit_test_simple.php`中添加`table()`函数\(步骤5\)，在`SimpleTest.php`中添加`testTable()`\(步骤6\)。重新运行单元测试，观察结果。

要测试一个类，将在`chap_13_unit_test_simple.php`中开发的函数复制到Demo类中（步骤7）。在对步骤8中建议的`SimpleTest.php`进行修改后，重新运行简单测试并观察结果。

### 测试数据库模型类

首先，创建一个要测试的示例类，`VisitorOps`，如本小节步骤2所示。现在你可以定义一个类，我们将调用`SimpleDatabaseTest`来测试`VisitorOps`。首先，使用`require_once`来加载要测试的类。\(我们将在下一个实例中讨论如何加入自动加载!\)然后定义关键属性，包括测试数据库配置和测试数据。你可以使用`php7cookbook_test`作为测试数据库。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/VisitorOps.php';
class SimpleDatabaseTest extends TestCase
{
  protected $visitorOps;
  protected $dbConfig = [
    'driver'   => 'mysql',
    'host'     => 'localhost',
    'dbname'   => 'php7cookbook_test',
    'user'     => 'cook',
    'password' => 'book',
    'errmode'  => PDO::ERRMODE_EXCEPTION,
  ];
  protected $testData = [
    'id' => 1,
    'email' => 'test@unlikelysource.com',
    'visit_date' => '2000-01-01 00:00:00',
    'comments' => 'TEST',
    'name' => 'TEST'
  ];
}
```

接下来，定义`setup()`，插入测试数据，并确认最后一条SQL语句是`INSERT`。还应该检查返回值是否为正值。

```php
public function setup()
{
  $this->visitorOps = new VisitorOps($this->dbConfig);
  $this->visitorOps->addVisitor($this->testData);
  $this->assertRegExp('/INSERT/', $this->visitorOps->getSql());
}
```

之后，定义 `teardown()`，删除测试数据，并确认查询 `id = 1` 的结果为 `FALSE`。

```php
public function teardown()
{
  $result = $this->visitorOps->removeById(1);
  $result = $this->visitorOps->findById(1);
  $this->assertEquals(FALSE, $result);
  unset($this->visitorOps);
}
```

首先测试的是`findAll()`。首先，确认结果的数据类型。你可以用`current()`取最上面的元素。我们确认有五个元素，其中有一个是`name`，并且其值与测试数据中的值相同。

```php
public function testFindAll()
{
  $result = $this->visitorOps->findAll();
  $this->assertInstanceOf(Generator::class, $result);
  $top = $result->current();
  $this->assertCount(5, $top);
  $this->assertArrayHasKey('name', $top);
  $this->assertEquals($this->testData['name'], $top['name']);
```

下一个测试是针对`findById()`。它与`testFindAll()`几乎相同。

```php
public function testFindById()
{
  $result = $this->visitorOps->findById(1);
  $this->assertCount(5, $result);
  $this->assertArrayHasKey('name', $result);
  $this->assertEquals($this->testData['name'], $result['name']);
}
```

你不需要费心去测试 `removeById()`，因为这已经在 `teardown()`中完成了。同样，也不需要测试`runSql()`，因为这已经作为其他测试的一部分完成了。

### 使用MOCK类

首先，定义一个 `VisitorService` 服务类，如本小节步骤 2 和 3 所述。接下来，定义一个`VisitorOpsMock`模拟类，这将在步骤4至7中讨论。

您现在可以为服务类开发一个测试，即 `VisitorServiceTest`。请注意，您需要提供您自己的数据库配置，因为最好的做法是使用测试数据库而不是生产版本。

```php
<?php
use PHPUnit\Framework\TestCase;
require_once __DIR__ . '/VisitorService.php';
require_once __DIR__ . '/VisitorOpsMock.php';

class VisitorServiceTest extends TestCase
{
  protected $visitorService;
  protected $dbConfig = [
    'driver'   => 'mysql',
    'host'     => 'localhost',
    'dbname'   => 'php7cookbook_test',
    'user'     => 'cook',
    'password' => 'book',
    'errmode'  => PDO::ERRMODE_EXCEPTION,
  ];
}
```

在`setup()`中，创建一个服务的实例，并在原类的位置插入`VisitorOpsMock`。

```php
public function setup()
{
  $this->visitorService = new VisitorService($this->dbConfig);
  $this->visitorService->setVisitorOps(new VisitorOpsMock());
}
public function teardown()
{
  unset($this->visitorService);
}
```

在我们的测试中，从访问者列表中产生一个HTML表格，然后你可以寻找某些元素，提前知道会发生什么，因为你可以控制测试数据。

```php
public function testShowAllVisitors()
{
  $result = $this->visitorService->showAllVisitors();
  $this->assertRegExp('!^<table>.+</table>$!', $result);
  $testData = $this->visitorService->getVisitorOps()->getTestData();
  foreach ($testData as $key => $value) {
    $dataWeWant = '!<td>' . $key . '</td>!';
    $this->assertRegExp($dataWeWant, $result);
  }
}
}
```

然后，你可能会希望尝试最后两个小节中建议的变化，使用匿名类作为Mock对象，以及使用Mock Builder。

## 更多...

其他断言测试对数字、字符串、数组、对象、文件、JSON和XML的操作，如下表所示。

| 分类 | 断言 |
| :--- | :--- |
| General | `assertEquals()`, `assertFalse()`, `assertEmpty()`, `assertNull()`, `assertSame(), assertThat()`, `assertTrue()` |
| Numeric | `assertGreaterThan()`, `assertGreaterThanOrEqual()`, `assertLessThan()`, `assertLessThanOrEqual()`, `assertNan()`, `assertInfinite()` |
| String | `assertStringEndsWith()`, `assertStringEqualsFile()`, `assertStringStartsWith()`, `assertRegExp()`, `assertStringMatchesFormat()`, `assertStringMatchesFormatFile()` |
| Array/iterator | `assertArrayHasKey()`, `assertArraySubset()`, `assertContains()`, `assertContainsOnly()`, `assertContainsOnlyInstancesOf()`, `assertCount()` |
| File | `assertFileEquals()`, `assertFileExists()` |
| Objects | `assertClassHasAttribute()`, `assertClassHasStaticAttribute()`, `assertInstanceOf()`, `assertInternalType()`, `assertObjectHasAttribute()` |
| JSON | `assertJsonFileEqualsJsonFile()`, `assertJsonStringEqualsJsonFile()`, `assertJsonStringEqualsJsonString()` |
| XML | `assertEqualXMLStructure()`, `assertXmlFileEqualsXmlFile()`, `assertXmlStringEqualsXmlFile()`, `assertXmlStringEqualsXmlString()` |

关于单元测试的精彩讨论，请看这里：https://en.wikipedia.org/wiki/Unit\_testing。 

关于 composer.json 文件指令的更多信息，请看 https://getcomposer.org/doc/04-schema.md。 

关于完整的断言列表，请看一下PHPUnit文档页：https://phpunit.de/manual/current/en/phpunit-book.html\#appendixes.assertions。 

PHPUnit文档还在这里详细介绍了如何使用getMockBuilder\(\)：https://phpunit.de/manual/current/en/phpunit-book.html\#test-doubles.mock-objects。

