# 使用命名空间

对高级 PHP 开发至关重要的一个方面是命名空间的使用。 任意定义的命名空间成为类名称的前缀，从而避免了因意外导致的类重复问题，并为您提供了极大的开发自由。 假设命名空间与目录结构匹配，使用命名空间的另一个好处是，它有助于自动加载，如第一章，建立基础中所述。

## 如何做...

1.要在命名空间中定义一个类，只需在代码文件的顶部添加关键字 `namespace` 即可：

```php
namespace Application\Entity;
```

{% hint style="info" %}
**最佳实践**

与建议每个文件只有一个类一样，每个文件也应该只有一个名称空间。
{% endhint %}

2.关键字 `namespace` 之前唯一的 PHP 代码只能是注释、/ 或关键字 \`declare\`：

```php
<?php
declare(strict_types=1);
namespace Application\Entity;
/**
 * Address
 *
 */
class Address
{
  // some code
}
```

3.在 PHP 5 中，如果需要访问一个外部命名空间中的类，可以在 `use` 语句前加上一个只包含命名空间的语句。然后你需要在这个命名空间中的任何类引用前加上命名空间的最后一部分：

```php
use Application\Entity;
$name = new Entity\Name();
$addr = new Entity\Address();
$prof = new Entity\Profile();
```

4.另外，您可以明确指定所有三个类：

```php
use Application\Entity\Name;
use Application\Entity\Address;
use Application\Entity\Profile;
$name = new Name();
$addr = new Address();
$prof = new Profile();
```

5.PHP 7 引入了一个被称为 **group use** 的语法改进，极大地提高了代码的可读性：

```php
use Application\Entity\ {
  Name,
  Address,
  Profile
};
$name = new Name();
$addr = new Address();
$prof = new Profile();
```

6.正如在第一章 《建立基础》中提到的，命名空间是自动加载过程中不可或缺的一部分。这个例子展示了一个示范性的自动加载器，它打印了所传递的参数，然后试图根据命名空间和类名来包含一个文件。这假定目录结构与命名空间相匹配。

```php
function __autoload($class)
{
  echo "Argument Passed to Autoloader = $class\n";
  include __DIR__ . '/../' . str_replace('\\', DIRECTORY_SEPARATOR, $class) . '.php';
}
```

## 如何运行...

为了便于说明，请定义与 `Application \*` 名称空间匹配的目录结构。 创建一个基本文件夹 `Application` 和一个子文件夹 `Entity`。 您还可以根据需要包括其他章节中使用的任何子文件夹，例如 `Database` 和 `Generic` ：

![](../../.gitbook/assets/image%20%2853%29.png)

接下来，在 `Application/Entity` 文件夹下创建三个实体类，每个实体类在各自的文件中：`Name.php`，`Address.php`和`Profile.php`。 我们仅在此处显示 `Application\Entity\Name`。 `Application\Entity\Address`和 `Application\Entity\Profile`将相同，除了 `Address` 具有`$address` 属性，`Profile`具有 `$profile` 属性，每个属性都具有适当的 `get` 和 `set`方法：

```php
<?php
declare(strict_types=1);
namespace Application\Entity;
/**
 * Name
 *
 */
class Name
{

  protected $name = '';

  /**
   * 此方法返回 $name 的当前值
   *
   * @return string $name
   */
  public function getName() : string
  {
    return $this->name;
  }

  /**
   * 此方法设置 $name 的值
   *
   * @param string $name
   * @return name $this
   */
  public function setName(string $name)
  {
    $this->name = $name;
    return $this;
  }
}
```

然后，您可以使用第一章 《建立基础》中定义的自动加载器，也可以使用前面提到的简单自动加载器。 将用于设置自动加载的命令放置在文件 `chap_04_oop_namespace_example_1.php` 中。 然后，在此文件中，您可以指定一个 `use` 语句，该语句仅引用名称空间，而不引用类名称。 通过使用名称空间的最后一部分 `Entity` 作为前缀，创建三个实体类 `Name`，`Address`和`Profile`的实例：

```php
use Application\Entity;
$name = new Entity\Name();
$addr = new Entity\Address();
$prof = new Entity\Profile();

var_dump($name);
var_dump($addr);
var_dump($prof);
```

这是输出：

![](../../.gitbook/assets/image%20%2855%29.png)

接下来，使用**另存为**将文件复制到名为 `chap_04_oop_namespace_example_2.php` 的新文件中。 将 `use` 语句更改为以下内容：

```php
use Application\Entity\Name;
use Application\Entity\Address;
use Application\Entity\Profile;
```

现在你可以只使用类名来创建类实例：

```php
$name = new Name();
$addr = new Address();
$prof = new Profile();
```

当你运行这个脚本时，这里是输出：

![](../../.gitbook/assets/image%20%2843%29.png)

最后，再次运行**另存为**并创建一个新文件 `chap_04_oop_namespace_example_3.php`。 现在，您可以测试PHP 7中引入的 **group use** 功能：

```php
use Application\Entity\ {
  Name,
  Address,
  Profile
};
$name = new Name();
$addr = new Address();
$prof = new Profile();
```

同样，当你运行这段代码时，输出将与前面的输出相同：

![](../../.gitbook/assets/image%20%2852%29.png)



