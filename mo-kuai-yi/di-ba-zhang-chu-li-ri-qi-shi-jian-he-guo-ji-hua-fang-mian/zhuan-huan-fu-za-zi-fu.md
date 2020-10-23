# 转换复杂字符

访问整个Unicode字符集的能力为渲染复杂的字符提供了许多新的可能性，特别是拉丁字母以外的字符。

## 如何做...

1.有些语言的读法是从右到左，而不是从左到右。例如希伯来语和阿拉伯语。在这个例子中，我们向您展示如何使用`U+202E` Unicode字符来呈现从右到左覆盖的反向文本。下面这行代码打印了`txet desreveR`。

```php
echo "\u{202E}Reversed text";
echo "\u{202D}";    // returns output to left-to-right
```

{% hint style="info" %}
完成后别忘了调用从左到右的覆盖字符`U+202D`!
{% endhint %}

2. 另一个考虑因素是使用合成字符。其中一个例子是`ñ`（字母`n`，上面有一个tilde `~`）。这在诸如`mañana`（西班牙语中的早晨或明天，取决于上下文）这样的单词中使用。还有一个可用的组成字符，由Unicode代码`U+00F1`表示。下面是其使用的一个例子，与`mañana`相呼应。

```php
echo "ma\u{00F1}ana"; // shows mañana
```

3. 然而，这可能会潜在地影响搜索的可能性。想象一下，你的客户没有一个带有这个组成字符的键盘。如果他们开始输入`man`试图搜索`mañana`，他们将不成功。

4. 使用完整的Unicode集提供了其他的可能性。你可以不使用组成的字符，而是使用原始字母`n`与Unicode组合码的组合，将浮动的tilde置于字母之上。在这个`echo`命令中，输出的结果和之前一样。只是组成单词的方式不同。

```php
echo "man\u{0303}ana"; // also shows mañana
```

5. 对于重音也可以有类似的应用。考虑一下法语单词`élève`（学生）。您可以使用合成字符来呈现它，或者使用组合代码将重音浮在字母上方。考虑以下两个例子。这两个例子产生了相同的输出，但呈现方式不同。

```php
echo "\u{00E9}l\u{00E8}ve";
echo "e\u{0301}le\u{0300}ve";
```

## 如何运行...

创建一个名为`chap_08_control_and_combining_unicode.php`的文件。确保包含元标签，向浏览器发出UTF-8字符编码的信号。

```markup
<!DOCTYPE html>
<html>
  <head>
    <title>PHP 7 Cookbook</title>
    <meta http-equiv="content-type" content="text/html;charset=utf-8" />
  </head>
```

接下来，设置基本的PHP和HTML来显示前面讨论的例子。

```php
  <body>
    <pre>
      <?php
        echo "\u{202E}Reversed text"; // reversed
        //echo "\u{202D}"; // stops reverse
        echo "mañana";  // using pre-composed characters
        echo "ma\u{00F1}ana"; // pre-composed character
        echo "man\u{0303}ana"; // "n" with combining ~ character (U+0303)
        echo "élève";
        echo "\u{00E9}l\u{00E8}ve"; // pre-composed characters
        echo "e\u{0301}le\u{0300}ve"; // e + combining characters
      ?>
    </pre>
</body>
</html>
```

下面是浏览器的输出。

![](../../.gitbook/assets/image%20%28104%29.png)

