# 在视图脚本中使用 emoji

表情一词是情感和图标的综合体。表情符号（Emoji），起源于日本，是另一套规模较大、应用广泛的图标。这些图标是小笑脸，小忍者，和在地板上打滚笑的图标，这些图标在任何有社交网络的网站上都非常流行。然而，在 PHP 7 之前，制作这些小动物是一项令人沮丧的工作。

## 如何做...

1.首先，你需要知道你想呈现的图标的Unicode。在互联网上进行快速搜索，你可以找到任何一个优秀的图表。以下是三个 “不见”、“不闻”、“不言”的猴子图标的代码。

 `U+1F648`, `U+1F649`和`U+1F64A`

![](../../.gitbook/assets/image%20%2894%29.png)

2.任何向浏览器输出的Unicode都必须被正确识别。这通常是通过元标签的方式来完成的。你应该将字符集设置为UTF-8。下面是一个例子。

```markup
<head>
  <title>PHP 7 Cookbook</title>
  <meta http-equiv="content-type" content="text/html;charset=utf-8" />
</head>
```

3. 传统的方法是简单地使用HTML来显示图标。因此，你可以这样做。

```markup
<table>
  <tr>
    <td>&#x1F648;</td>
    <td>&#x1F649;</td>
    <td>&#x1F64A;</td>
  </tr>
</table>
```

4. 从 PHP 7 开始，现在可以使用这种语法来构造完整的 Unicode 字符。"`\u{xxx}`"。下面是一个例子，它的三个图标与上面的相同。

```php
<table>
  <tr>
    <td><?php echo "\u{1F648}"; ?></td>
    <td><?php echo "\u{1F649}"; ?></td>
    <td><?php echo "\u{1F64A}"; ?></td>
  </tr>
</table>
```

{% hint style="info" %}
你的操作系统和浏览器必须同时支持Unicode，并且必须有正确的字体集。例如，在Ubuntu Linux中，你需要安装`ttf-ancient-fonts`软件包才能在浏览器中看到表情符号。
{% endhint %}

## 如何运行...

在 PHP 7 中，引入了一种新的语法，可以渲染任何 Unicode 字符。与其他语言不同的是，新的 PHP 语法允许使用可变数量的十六进制数字。基本格式是这样的。

```php
\u{xxxx}
```

整个结构必须使用双引号\(或使用heredoc\)。 xxxx可以是任意的十六进制数字组合，2、4、6及以上。

创建一个名为`chap_08_emoji_using_html.php`的文件。确保包含元标签，向浏览器发出UTF-8字符编码的信号。

```markup
<!DOCTYPE html>
<html>
  <head>
    <title>PHP 7 Cookbook</title>
    <meta http-equiv="content-type" content="text/html;charset=utf-8" />
  </head>
```

接下来，建立一个基本的HTML表格，并显示一排表情。

```markup
  <body>
    <table>
      <tr>
        <td>&#x1F648;</td>
        <td>&#x1F649;</td>
        <td>&#x1F64A;</td>
      </tr>
    </table>
  </body>
</html>
```

现在，使用PHP添加一行来发出表情。

```php
 <tr>
    <td><?php echo "\u{1F648}"; ?></td>
    <td><?php echo "\u{1F649}"; ?></td>
    <td><?php echo "\u{1F64A}"; ?></td>
  </tr>
```

这是从Firefox看到的输出。

![](../../.gitbook/assets/image%20%28108%29.png)

## 更多...

表情代码列表，见[http://unicode.org/emoji/charts/full-emoji-list.html](http://unicode.org/emoji/charts/full-emoji-list.html)。

