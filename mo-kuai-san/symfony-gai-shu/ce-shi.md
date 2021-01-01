# 测试

如今，测试已经成为每个现代 web 应用程序不可或缺的一部分。通常测试这个词意味着单元测试和功能测试。单元测试是关于测试我们的 PHP 类。每个 PHP 类都被认为是一个单元，因此称为单元测试。另一方面，功能测试测试我们应用程序的各个层次，通常集中于测试整个功能，比如登录或注册过程。

PHP 生态系统有一个很棒的单元测试框架，叫做 PHPUnit，可以在 [https://PHPUnit.de](https://PHPUnit.de) 下载。它使我们能够编写基本的单元测试，也能够编写功能类型测试。Symfony 最棒的地方在于它内置了对 PHPUnit 的支持。

在我们开始运行 Symfony 的测试之前，我们需要确保我们已经安装了 PHPUnit，并且可以作为控制台命令使用。当执行时，`PHPUnit.xml` 或 `PHPUnit.xml.dist` 会自动尝试从当前工作目录中的 `PHPUnit.xml` 或 `PHPUnit.xml.dist` 获取和读取测试配置。默认情况下，Symfony 在其根文件夹中附带一个 `phpunit.xml.dist` 文件，因此 phpunit 命令可以选择其测试配置。

下面是默认 `phpunit.xml.dist` 文件的部分示例:

```markup
<phpunit … >
  <php>
    <ini name="error_reporting" value="-1" />
    <server name="KERNEL_DIR" value="app/" />
  </php>

  <testsuites>
    <testsuite name="Project Test Suite">
      <directory>tests</directory>
    </testsuite>
  </testsuites>

  <filter>
    <whitelist>
      <directory>src</directory>
      <exclude>
        <directory>src/*Bundle/Resources</directory>
        <directory>src/*/*Bundle/Resources</directory>
        <directory>src/*/Bundle/*Bundle/Resources</directory>
      </exclude>
    </whitelist>
  </filter>
</phpunit>
```

`Testsuites` 元素定义了目录测试，我们所有的测试都位于其中。带有其子元素的 `filter` 元素用于配置白名单以报告代码覆盖率。PHP 元素及其子元素用于配置 PHP 设置、常量和全局变量。

对我们这样的默认项目运行 `phpunit` 命令会得到如下输出:

![](../../.gitbook/assets/image%20%28207%29.png)

请注意，bundle 测试不会自动提取。所以没有执行我们的 `src/AppBundle/Tests/Controller/CustomerControllerTest.php` 文件，该文件是在我们使用自动生成的 CRUD 时自动为我们创建的。不是因为它的内容默认被注释掉了，而是因为 `bundle` 测试目录对于 `phpunit` 不可见。为了让它执行，我们需要扩展 `phpunit.xml.dist` 文件，将它添加到目录 `testsuite` 中，如下所示:

```markup
<testsuites>
  <testsuite name="Project Test Suite">
    <directory>tests</directory>
    <directory>src/AppBundle/Tests</directory>
  </testsuite>
</testsuites>
```

根据我们构建应用程序的方式，我们可能希望将所有 bundle 添加到 `testsuite` 列表中，即使我们计划独立发布 bundle。

关于测试还有很多要说的。我们将在后面的章节中一点一点地完成，并讨论单个 bundle 的需求。目前，了解如何触发测试以及如何向测试配置添加新位置就足够了。

