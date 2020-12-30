# Symfony 概述

像Symfony这样的全栈框架通过提供从用户界面到数据存储的所有必要组件，帮助简化了构建模块化应用的过程。这使得随着应用的发展，可以快速循环地交付各个零碎的应用。稍后我们将通过将我们的应用程序分割成若干个小的模块，或者Symfony术语中的捆绑包来体验这一点。

接下来，我们将安装Symfony，创建一个空白项目，并开始研究构建模块化应用所必需的单个框架功能：

* 控制器 
* 路由
* 模板 
* 表单 
* 捆绑系统 
* 数据库和Doctrine 
* 测试 
* 验证

## 安装Symphony

安装Symfony是非常简单的。我们可以使用以下命令在Linux或Mac OS X上安装Symfony：

```bash
sudo curl -LsS https://symfony.com/installer -o /usr/local/bin/symfony
sudo chmod a+x /usr/local/bin/symfony
```

我们可以使用以下命令在Windows上安装Symfony：

```text
c:\> php -r "file_put_contents('symfony', file_get_contents('https://symfony.com/installer'));"
```

一旦命令被执行，我们可以简单地将新创建的`symfony`文件移动到我们的项目目录下，并进一步以`symfony`的形式执行，或者在Windows中以`php symfony`的形式执行。

这样就会产生如下所示的输出：

![](../../.gitbook/assets/image%20%28154%29.png)

前面的响应表明我们已经成功地设置了Symfony，现在可以开始创建新项目了。

