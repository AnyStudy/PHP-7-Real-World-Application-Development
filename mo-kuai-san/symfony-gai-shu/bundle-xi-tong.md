# bundle 系统

大多数流行的框架和平台都支持某种形式的模块、插件、扩展或 bundles。在大多数时候，两者的区别其实只是在命名上，而可扩展性和模块化的概念是一样的。在 Symfony 中，这些模块化模块被称为 bundles。

bundles 是 Symfony 中的一等公民，因为它们支持其他组件的所有操作。Symfony 中的所有东西都是一个bundle，甚至是核心框架。bundles 使我们能够构建模块化的应用程序，而一个特定功能的全部代码都包含在一个目录中。

一个bundle 将所有的PHP文件，模板，样式表，JavaScript文件，测试和其他任何东西都放在一个根目录中。

当我们第一次设置测试应用程序时，它为我们创建了一个`AppBundle`，在`src`目录下。随着我们推进自动生成的CRUD，我们看到我们的 bundle 得到了各种各样的目录和文件。

要想让一个 bundle 被 Symfony 注意到，需要将它添加到`app/AppKernel.php`文件中，使用`registerBundles`方法，如图所示：

```php
public function registerBundles()
{
  $bundles = [
    new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
    new Symfony\Bundle\SecurityBundle\SecurityBundle(),
    new Symfony\Bundle\TwigBundle\TwigBundle(),
    new Symfony\Bundle\SwiftmailerBundle\SwiftmailerBundle(),
    new Doctrine\Bundle\DoctrineBundle\DoctrineBundle(),
    //…
    new AppBundle\AppBundle(),
  ];

  //…

  return $bundles;
}
```

创建一个新的 bundle 就像创建一个PHP文件一样简单。让我们继续创建一个`src/TestBundle/TestBundle.php`文件，内容如下：

```php
namespace TestBundle;

use Symfony\Component\HttpKernel\Bundle\Bundle;

class TestBundle extends Bundle
{
  …
}
```

一旦文件到位，我们需要做的就是通过`app/AppKernel.php`文件的`registerBundles`方法进行注册，如图所示：

```php
class AppKernel extends Kernel {
//…
  public function registerBundles() {
    $bundles = [
      // …
      new TestBundle\TestBundle(),
      // …
    ];
    return $bundles;
  }
  //…
}
```

创建一个 bundle 的更简单的方法是运行一个控制台命令，如下所示：

```bash
php bin/console generate:bundle --namespace=Foggyline/TestBundle
```

这将会触发一系列关于 bundle 的问题，最终导致 bundle 的创建，看起来像下面的截图：

![](../../.gitbook/assets/image%20%28214%29.png)

一旦完成这个过程，就会创建一个包含多个目录和文件的新 bundle，如下图所示：

![](../../.gitbook/assets/image%20%28221%29.png)

Bundle 生成器很友好地创建了控制器、依赖注入扩展、路由、准备服务配置、模板，甚至测试。由于我们选择共享我们的 bundle，Symfony 选择了 XML 作为默认的配置格式。依赖扩展简单来说就是我们可以通过使用 `foggyline_test`作为 Symfony 主 `config.yml`的根元素来访问我们的 bundle 配置。实际的`foggyline_test`元素是在`DependencyInjection/Configuration.php`文件中定义的。

