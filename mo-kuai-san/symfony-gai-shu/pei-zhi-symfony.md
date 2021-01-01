# 配置 Symfony

为了跟上现代的需求，今天的框架和应用程序需要一个灵活的配置系统。Symfony 通过其强大的配置文件和环境概念很好地完成了这个角色。

默认的 Symfony 配置文件 `config.yml` 位于`app/config/`目录下，（部分）内容划分如下：

```yaml
imports:
  - { resource: parameters.yml }
  - { resource: security.yml }
  - { resource: services.yml }

framework:
…

# Twig Configuration
twig:
…

# Doctrine Configuration
doctrine:
…

# Swiftmailer Configuration
swiftmailer:
…
```

像`framework`、`twig`、`doctrine`和`swiftmailer`这样的顶层条目定义了单个bundle的配置。

可以选择，配置文件可以是XML或PHP格式（`config.xml`或`config.php`）。虽然YAML简单易读，但XML功能更强大，而PHP功能强大但可读性较差。

我们可以使用控制台工具来转储整个配置，如图所示：

```bash
php bin/console config:dump-reference FrameworkBundle
```

前面的例子列出了core `FrameworkBundle`的配置文件。我们可以使用同样的命令来显示任何实现容器扩展的 bundle 的可能配置，这一点我们将在后面研究。

Symfony 有一个很好的环境概念的实现。在`app/config`目录下，我们可以看到默认的 Symfony 项目实际上是以三种不同的环境开始的：

* `config_dev.yml`
* `config_prod.yml`
* `config_test.yml`

每个应用程序可以在不同的环境中运行。每个环境共享相同的代码，但配置不同。开发环境可能会使用大量的日志，而prod环境可能会使用大量的缓存。

这些环境的触发方式是通过前台控制器文件，如下面的部分例子：

```php
# web/app.php
…
$kernel = new AppKernel('prod', false);
…

# web/app_dev.php
…
$kernel = new AppKernel('dev', true);
…
```

这里缺少测试环境，因为它只在运行自动测试时使用，不能直接通过浏览器访问。

`app/AppKernel.php`文件是实际加载配置的文件，无论是YAML，XML，还是PHP，如下面的代码片段所示：

```php
public function registerContainerConfiguration(LoaderInterface $loader)
{
  $loader->load($this->getRootDir().'/config/config_'.$this->getEnvironment().'.yml');
}
```

环境遵循同样的概念，而每个环境都会导入基础配置文件，然后修改其值以适应特定环境的需要。

