# 使用 Symfony 控制台

Symfony 框架自带了一个内置的控制台工具，我们只需在项目根目录下执行以下命令即可触发：

```bash
php bin/console
```

这样，屏幕上会显示大量可用命令列表，分为以下几组：

* `assets`
* `cache`
* `config`
* `debug`
* `doctrine`
* `generate`
* `lint`
* `orm`
* `router`
* `security`
* `server`
* `swiftmailer`
* `translation`

这些赋予我们各种功能。我们今后特别关注的是`doctrine`和`generate`命令。`doctrine`命令，更具体地说，是`doctrine:generate:crud`，它基于一个现有的`Doctrine`实体生成一个CRUD。此外，`doctrine:generate:entity`命令会在现有的bundle内生成一个新的Doctrine实体。对于我们想要快速、简单地创建实体以及围绕实体的整个CRUD的情况来说，这些命令是非常方便的。同样，`generate:doctrine:entity`和`generate:doctrine:crud`也做同样的事情。

在我们继续测试这些命令之前，我们需要确保我们的数据库配置参数已经到位，这样Symfony才能看到并与我们的数据库对话。为此，我们需要在`app/config/parameters.yml`文件中设置适当的值。

在本节中，让我们继续在默认的`AppBundle`包中创建一个简单的`Customer`实体，并围绕它进行整个CRUD，假设`Customer`实体有以下属性： `firstname、lastname`和`e-mail`。我们先在项目根目录下运行`php bin/console generate:doctrine:entity`命令，结果如下：

![](../../.gitbook/assets/image%20%28170%29.png)

在这里，我们首先提供`AppBundle:Customer`作为实体名称，并确认使用注释作为配置格式。

最后，我们被要求开始向我们的实体添加字段。输入第一个名字并按下回车键，我们将通过一系列关于我们的字段类型、长度、nullable和唯一状态的简短问题，如下面的截图所示：

![](../../.gitbook/assets/image%20%28163%29.png)

现在我们应该为我们的`Customer`实体生成两个类。通过Symfony和Doctrine的帮助，这些类被放在了对象关系映射器（ORM）的上下文中，因为它们将`Customer`实体和合适的数据库表联系起来。然而，我们还没有指示Symfony为我们的实体创建表。要做到这一点，我们执行以下命令：

```bash
php bin/console doctrine:schema:update --force
```

这将产生如下截图所示的输出:

![](../../.gitbook/assets/image%20%28164%29.png)

如果我们现在看一下数据库，我们应该看到一个客户表，上面有所有适当的列，用SQL创建dsyntax创建如下：

```sql
CREATE TABLE `customer` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `firstname` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `lastname` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `email` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UNIQ_81398E09E7927C74` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```

在这一点上，我们仍然没有一个实际的CRUD功能。我们只是有一个ORM授权的`Customer`实体类和它背后的相应数据库表。下面的命令将为我们生成实际的CRUD控制器和模板：

```bash
php bin/console generate:doctrine:crud
```

这将产生以下交互式输出：

![](../../.gitbook/assets/image%20%28150%29.png)

通过提供完全分类的实体名称`AppBundle:Customer`，生成器会进行一系列额外的输入，从生成写操作、读取的配置类型到路由的前缀，如下截图所示：

![](../../.gitbook/assets/image%20%28155%29.png)

一旦完成，我们应该能够通过简单地打开一个像 `http://test.app/customer/`（假设`test.app`是我们为我们的例子设置的主机）这样的URL来访问我们的Customer CRUD动作，如图所示：

![](../../.gitbook/assets/image%20%28206%29.png)

如果我们点击创建一个新的条目链接，我们将被重定向到`/customer/new/`URL，如下图所示：

![](../../.gitbook/assets/image%20%28167%29.png)

在这里，我们可以为我们的Customer实体输入实际值，并点击Create按钮，以便将其持久化到数据库`customer`表中。在添加了几个实体之后，现在初始的`/customer/` URL已经能够将它们全部列出，如下截图所示：

![](../../.gitbook/assets/image%20%28191%29.png)

在这里我们看到了显示和编辑操作的链接。显示操作是我们可以认为是面向客户的操作，而编辑操作是面向管理员的操作，点击编辑操作，我们会进入到`/customer/1/edit/`这个表单的URL。而这里的数字1是数据库中客户实体的ID。

![](../../.gitbook/assets/image%20%28153%29.png)

在这里，我们可以更改属性值，然后点击 "编辑 "将其持久化回数据库，也可以点击 "删除 "按钮将实体从数据库中删除。

如果我们要用一个已经存在的电子邮件创建一个新的实体，这个实体被标记为一个唯一的字段，系统会抛出一个通用的错误，比如下面这个错误：

![](../../.gitbook/assets/image%20%28199%29.png)

这只是系统默认的行为，随着我们的进一步发展，我们会研究如何让这个行为更加人性化。到现在，我们已经看到了Symfony的控制台是多么的强大。通过几个简单的命令，我们就能创建我们的实体和它的整个CRUD动作。控制台还有更多的功能，我们甚至可以创建自己的控制台命令，因为我们可以实现任何类型的逻辑。然而，就我们的需求而言，目前的实现暂时就足够了。

