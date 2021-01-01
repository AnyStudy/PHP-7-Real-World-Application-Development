# 数据库和 Doctrine

数据库是几乎所有网络应用的支柱。每当我们需要存储或检索数据时，我们都会借助数据库来完成。现代OOP世界的挑战是抽象数据库，使我们的 PHP 代码与数据库无关。MySQL 可能是 PHP 世界中最知名的数据库。PHP本身对 MySQL 的工作有很大的支持，无论是通过`mysqli_*`扩展还是通过 PDO。但是，这两种方法都是MySQL特有的。Doctrine 通过引入一个抽象层次来解决这个问题，使我们能够使用PHP对象来表示 MySQL中的表、行和它们的关系。

Doctrine 与 Symfony 完全脱钩，所以使用它完全是可选的。然而，它的伟大之处在于，Symfony 控制台提供了基于 Doctrine ORM 的伟大的自动生成的 CRUD，正如我们在之前的例子中创建 Customer 实体时看到的那样。

我们一创建项目，Symfony 就为我们提供了一个自动生成的 `app/config/parameters.yml` 文件。在这个文件中，我们除了提供数据库访问信息外，还提供了如下示例所示的数据库访问信息：

```yaml
parameters:
database_host: 127.0.0.1
database_port: null
database_name: symfony
database_user: root
database_password: mysql
```

一旦我们配置了合适的参数，我们就可以使用控制台生成功能了。

值得注意的是，这个文件中的参数只是一个约定，因为 `app/config/config.yml` 正在按照 `doctrine dbal` 进行配置，如下所示:

```php
doctrine:
dbal:
  driver:   pdo_mysql
  host:     "%database_host%"
  port:     "%database_port%"
  dbname:   "%database_name%"
  user:     "%database_user%"
  password: "%database_password%"
  charset:  UTF8
```

Symfony 控制台工具允许我们删除和创建一个基于这个配置的数据库，这在开发过程中很方便，如下面的代码块所示：

```bash
php bin/console doctrine:database:drop --force
php bin/console doctrine:database:create
```

我们在前面看到了控制台工具如何使我们能够创建实体以及它们到数据库表的映射。这将满足我们整本书的需要。一旦我们创建了它们，我们就需要能够对它们执行 CRUD 操作。如果我们忽略自动生成的 CRUD 控制器 `src/AppBundle/Controller/CustomerController.php` 文件，我们可以按如下方式生成与 CRUD 相关的代码:

```php
// Fetch all entities
$customers = $em->getRepository('AppBundle:Customer')->findAll();

// Persist single entity (existing or new)
$em = $this->getDoctrine()->getManager();
$em->persist($customer);
$em->flush();

// Delete single entity
$em = $this->getDoctrine()->getManager();
$em->remove($customer);
$em->flush();
```

关于 Doctrine 还有很多要说的，这远远超出了本书的范围。更多信息可以在官方网页上找到\( [http://www.doctrine-project.org](http://www.doctrine-project.org) \)。

