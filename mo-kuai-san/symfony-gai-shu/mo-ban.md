# 模板

前面我们说过，控制器接受请求并返回响应。然而，响应通常可以是任何内容类型。实际内容的生产是控制器委托给模板引擎的事情。然后，模板引擎有能力将响应变成HTML、JSON、XML、CSV、LaTeX或任何其他基于文本的内容类型。

在以前，程序员将PHP与HTML混合成所谓的PHP模板（`.php`和`.phtml`）。虽然现在一些平台还在使用，但这种方式被认为是不安全的，而且在很多方面都有所欠缺。其中之一就是将业务逻辑塞进模板文件中。

为了解决这些缺点，Symfony 打包了自己的模板语言`Twig`。与PHP不同的是，`Twig`的目的是严格地表达表现形式，而不是思考程序逻辑。我们无法在`Twig`中执行任何PHP代码。而`Twig`的代码不过是一个HTML，加上一些特殊的语法类型。

`Twig` 定义了三种类型的特殊语法：

* `{{ ... }}`: 将变量或表达式的结果输出到模板中
* `{% ... %}`:这个标签控制模板的逻辑（`if`和`for`循环，以及其他）
* `{# ... #}`:它相当于PHP的`/* comment */`语法。`Comments` 内容不包含在渲染的页面中

过滤器是`Twig`的另一个不错的功能。它们就像链式方法调用一个变量值一样，在输出之前修改内容，如下所示：

```markup
<h1>{{ title|upper }}</h1>

{{ filter upper }}
<h1>{{ title }}</h1>
{% endfilter %}

<h1>{{ title|lower|escape }}</h1>

{% filter lower|escape %}
<h1>{{ title }}</h1>
{% endfilter %}
```

它还支持以下功能：

```markup
{{ random(['phone', 'tablet', 'laptop']) }}
```

前面的随机函数调用会从数组中返回一个随机值。有了所有内置的过滤器和函数列表，`Twig`还允许我们在需要的时候自己编写。

与PHP类的继承类似，`Twig`也支持模板和布局的继承。让我们快速回顾一下`app/Resources/views/customer/index.html.twig`文件，如下所示：

```markup
{% extends 'base.html.twig' %}

{% block body %}
<h1>Customer list</h1>
…
{% endblock %}
```

这里我们看到一个客户`index.html.twig`模板，使用`extends`标签扩展另一个模板，本例中`base.html.twig`在`app/Resources/views/`目录下找到，内容如下：

```markup
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>{% block title %}Welcome!{% endblock %}</title>
    {% block stylesheets%}{% endblock %}
    <link rel="icon" type="image/x-icon"href="{{ asset('favicon.ico') }}" />
  </head>
  <body>
    {% block body %}{% endblock %}
    {% block javascripts%}{% endblock %}
  </body>
</html>
```

在这里，我们看到了几个区块标签：`1title`、`styleheets`、`body`和`javascripts`。我们可以在这里声明任意多的区块，并以任何方式命名它们。这使得扩展标签成为模板继承的关键。它告诉`Twig`首先评估基础模板，它设置了布局和定义了块，之后像`customer/index.html.twig`这样的子模板就会填充这些块的内容。

模板在两个位置：

* `app/Resources/views/`
* `bundle-directory/Resources/views/`

这意味着，为了 `render/extend app/Resources/views/base.html.twig`，我们将在模板文件中使用`base.html.twig`，为了`render/extend app/Resources/views/customer/index.html.twig`，我们将使用`customer/index.html.twig`路径。

当使用驻留在 bundle 中的模板时，我们必须以稍微不同的方式引用它们。在这种情况下，我们使用`bundle:directory:filename`字符串语法。以`FoggylineCatalogBundle:Product:index.html.twig`路径为例。这将是使用bundle模板文件的一个完整路径。这里`FoggylineCatalogBundle`是一个bundle名称，`Product`是该bundle `Resources/views` 目录下的一个目录名称，`index.html.twig`是`Product`目录下实际模板的名称。

每个模板文件名都有两个扩展名，首先指定格式，然后指定该模板的引擎；如`*.html.twig、*.html.php`和`*.css.twig`。

当我们开始构建我们的应用程序时，我们将进入关于这些模板的更多细节。





