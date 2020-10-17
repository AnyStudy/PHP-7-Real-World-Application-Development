# 构建一个 OOP SQL 查询生成器

PHP 7实现了一种称为上下文相关的词法分析器。这意味着，如果上下文允许，通常保留的词可以被使用，因此，当构建面向对象的 SQL 构建器时，我们可以使用名为 `and`, `or`, `not` 等方法。

## 如何做...

1.我们定义了一个 `Application/Database/Finder` 类。在这个类中，我们定义了与我们最喜欢的SQL操作相匹配的方法。

```php
namespace Application\Database;
class Finder
{
  public static $sql      = '';
  public static $instance = NULL;
  public static $prefix   = '';
  public static $where    = array();
  public static $control  = ['', ''];
  
    // $a == name of table
    // $cols = column names
    public static function select($a, $cols = NULL)
    {
      self::$instance  = new Finder();
      if ($cols) {
           self::$prefix = 'SELECT ' . $cols . ' FROM ' . $a;
      } else {
        self::$prefix = 'SELECT * FROM ' . $a;
      }
      return self::$instance;
    }
    
    public static function where($a = NULL)
    {
        self::$where[0] = ' WHERE ' . $a;
        return self::$instance;
    }
    
    public static function like($a, $b)
    {
        self::$where[] = trim($a . ' LIKE ' . $b);
        return self::$instance;
    }
    
    public static function and($a = NULL)
    {
        self::$where[] = trim('AND ' . $a);
        return self::$instance;
    }
    
    public static function or($a = NULL)
    {
        self::$where[] = trim('OR ' . $a);
        return self::$instance;
    }
    
    public static function in(array $a)
    {
        self::$where[] = 'IN ( ' . implode(',', $a) . ' )';
        return self::$instance;
    }
    
    public static function not($a = NULL)
    {
        self::$where[] = trim('NOT ' . $a);
        return self::$instance;
    }
    
    public static function limit($limit)
    {
        self::$control[0] = 'LIMIT ' . $limit;
        return self::$instance;
    }
    
    public static function offset($offset)
    {
        self::$control[1] = 'OFFSET ' . $offset;
        return self::$instance;
    }

  public static function getSql()
  {
    self::$sql = self::$prefix
       . implode(' ', self::$where)
               . ' '
               . self::$control[0]
               . ' '
               . self::$control[1];
    preg_replace('/  /', ' ', self::$sql);
    return trim(self::$sql);
  }
}

```

2.每个用于生成SQL片段的函数都会返回相同的属性，即 `$instance`。这使得我们可以使用一个流畅的接口来表示代码，比如这样。

```php
$sql = Finder::select('project')->where('priority > 9') ... etc.
```

## 如何运行...

将前面定义的代码复制到 `Application/Database` 文件夹下的`Finder.php`文件中。然后您可以创建一个`chap_05_oop_query_builder.php`调用程序，它初始化了在第一章建立基础中定义的自动加载器。然后你可以运行`Finder::select()`来生成一个对象，从这个对象中可以呈现SQL字符串。

```php
<?php
require __DIR__ . '/../Application/Autoload/Loader.php';
Application\Autoload\Loader::init(__DIR__ . '/..');
use Application\Database\Finder;

$sql = Finder::select('project')
  ->where()
  ->like('name', '%secret%')
  ->and('priority > 9')
  ->or('code')->in(['4', '5', '7'])
  ->and()->not('created_at')
  ->limit(10)
  ->offset(20);

echo Finder::getSql();
```

这是预编码的结果：

![](../../.gitbook/assets/image%20%2872%29.png)

## 参考

有关上下文相关词法分析器的更多信息，请参见本文：

[https://wiki.php.net/rfc/context\_sensitive\_lexer](https://wiki.php.net/rfc/context_sensitive_lexer)

