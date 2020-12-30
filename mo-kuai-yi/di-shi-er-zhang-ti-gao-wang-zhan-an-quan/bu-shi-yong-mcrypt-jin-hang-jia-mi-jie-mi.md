# 不使用mcrypt进行加密/解密

在一般的 PHP 社区成员中，一个鲜为人知的事实是，作为大多数基于 PHP 的加密的核心的 `mcrypt` 扩展被认为是安全的，但它并不安全。从安全的角度来看，最大的问题之一是 `mcrypt` 扩展需要高级的密码学知识才能成功运行，而很少有程序员具备这些知识。这就导致了严重的误操作，最终导致数据损坏的几率为256分之一等问题。这可不是什么好机会。此外，开发人员对`libmcrypt`的支持，即`mcrypt`扩展所基于的核心库，已于2007年被放弃，这意味着代码库已经过时，错误百出，并且没有应用补丁的机制。因此，了解如何在不使用`mcrypt`的情况下进行强加密/解密是非常重要的。

## 如何做...

1.之前提出的问题的解决方案，如果你想知道，就是使用`openssl`。这个扩展维护得很好，并且具有现代的、非常强大的加密/解密功能。

{% hint style="danger" %}
为了使用任何`openssl*`函数，openssl PHP扩展必须被编译并启用！此外，你需要在你的Web服务器上安装最新的`OpenSSL`包。
{% endhint %}

2. 首先，你需要确定你的安装中哪些密码方法是可用的。为此，您可以使用 `openssl_get_cipher_methods()` 命令。例子包括基于高级加密标准（AES）、BlowFish（BF）、CAMELLIA、CAST5、数据加密标准（DES）、Rivest密码（RC）（也被亲切地称为Ron's Code）和SEED的算法。你会注意到，这个方法显示的密码方法是大写和小写重复的。

3. 接下来，你需要弄清楚哪种方法最适合你的需求。下面是一个表格，对各种方法进行了快速的总结。

<table>
  <thead>
    <tr>
      <th style="text-align:left">&#x65B9;&#x6CD5;</th>
      <th style="text-align:left">&#x53D1;&#x5E03;&#x65F6;&#x95F4;</th>
      <th style="text-align:left">&#x5BC6;&#x94A5;&#x5927;&#x5C0F;(bits)</th>
      <th style="text-align:left">&#x5BC6;&#x94A5;&#x5757;&#x5927;&#x5C0F; (bytes)</th>
      <th style="text-align:left">&#x5907;&#x6CE8;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left"><code>camellia</code>
      </td>
      <td style="text-align:left">2000</td>
      <td style="text-align:left">128, 192, 256</td>
      <td style="text-align:left">16</td>
      <td style="text-align:left">&#x7531;&#x4E09;&#x83F1;&#x548C;NTT&#x5171;&#x540C;&#x5F00;&#x53D1;</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>aes</code>
      </td>
      <td style="text-align:left">1998</td>
      <td style="text-align:left">128, 192, 256</td>
      <td style="text-align:left">16</td>
      <td style="text-align:left">&#x7531;Joan Daemen&#x548C;Vincent Rijmen&#x5F00;&#x53D1;&#x3002;&#x6700;&#x521D;&#x4EE5;Rijndael&#x7684;&#x540D;&#x4E49;&#x63D0;&#x4EA4;</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>seed</code>
      </td>
      <td style="text-align:left">1998</td>
      <td style="text-align:left">128</td>
      <td style="text-align:left">16</td>
      <td style="text-align:left">&#x7531;&#x97E9;&#x56FD;&#x4FE1;&#x606F;&#x5B89;&#x5168;&#x5C40;&#x5F00;&#x53D1;</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>cast5</code>
      </td>
      <td style="text-align:left">1996</td>
      <td style="text-align:left">40-128</td>
      <td style="text-align:left">8</td>
      <td style="text-align:left">&#x7531;Carlisle Adams&#x548C;Stafford Tavares&#x5F00;&#x53D1;</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>bf</code>
      </td>
      <td style="text-align:left">1993</td>
      <td style="text-align:left">1 - 448</td>
      <td style="text-align:left">8</td>
      <td style="text-align:left">&#x7531;Bruce Schneier&#x8BBE;&#x8BA1;</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>rc2</code>
      </td>
      <td style="text-align:left">1987</td>
      <td style="text-align:left">
        <p>8 - 1,024</p>
        <p>&#x9ED8;&#x8BA4; 64</p>
      </td>
      <td style="text-align:left">8</td>
      <td style="text-align:left">&#x7531;Ron Rivest&#xFF08;RSA&#x7684;&#x6838;&#x5FC3;&#x521B;&#x59CB;&#x4EBA;&#x4E4B;&#x4E00;&#xFF09;&#x8BBE;&#x8BA1;</td>
    </tr>
    <tr>
      <td style="text-align:left"><code>des</code>
      </td>
      <td style="text-align:left">1977</td>
      <td style="text-align:left">56 (+8 &#x5947;&#x5076;&#x6821;&#x9A8C;&#x4F4D;)</td>
      <td style="text-align:left">8</td>
      <td style="text-align:left">&#x7531;IBM&#x5F00;&#x53D1;&#xFF0C;&#x57FA;&#x4E8E;Horst Feistel&#x7684;&#x5DE5;&#x4F5C;</td>
    </tr>
  </tbody>
</table>

4. 另一个考虑因素是你喜欢的块密码操作模式是什么。常见的选择总结在这个表中。

| 模式 | 代表 | 备注 |
| :--- | :--- | :--- |
| ECB | Electronic Code Book | 不需要初始化向量\(IV\);加密和解密都支持并行化;简单快速;不隐藏数据模式;不推荐!!! |
| CBC | Cipher Block Chaining | 需要IV；后续的块，即使是相同的，也会与前一个块进行XOR编辑，从而获得更好的整体加密效果；如果IV是可预测的，第一个块可以被解密，剩下的信息就会暴露出来；信息必须被填充到密码块大小的倍数；只支持解密的并行化 |
| CFB | Cipher Feedback | CBC的近亲，只是加密方式是反向的 |
| OFB | Output Feedback | 非常对称：加密和解密相同；完全不支持并行化 |
| CTR | Counter | 与OFB的操作类似；支持加密和解密的并行化 |
| CCM | Counter with CBC-MAC | CTR的衍生物；仅设计为128位的块长；提供认证和保密性；CBC-MAC代表密码块链-信息认证码 |
| GCM | Galois/Counter Mode | 基于CTR模式；应该对每个流使用不同的IV进行加密；特别高的吞吐量（与其他模式相比）；支持加密和解密的并行化 |
| XTS | XEX-based Tweaked-codebook mode with ciphertext Stealing | 相对较新\(2010年\)且速度较快；使用两把钥匙；增加了可作为一个区块安全加密的数据量 |

5. 在选择加密方法和模式之前，你还需要确定加密的内容是否需要在你的PHP应用之外解密。例如，如果你是将数据库凭证加密存储到一个独立的文本文件中，你是否需要具备从命令行解密的能力？如果是这样，请确保你选择的加密方法和操作模式是目标操作系统所支持的。

6. 为IV提供的字节数根据选择的密码方法而变化。为了得到最好的结果，可以使用 `random_bytes()`（PHP 7 中新增），它返回一个真正的 **CSPRNG** 字节序列。IV的长度有很大的不同。试着从16开始。如果产生警告，将显示该算法所需提供的正确字节数，因此要相应调整大小。

```php
$iv  = random_bytes(16);
```

7. 要执行加密，使用`openssl_encrypt()`。以下是需要传递的参数。

| 参数 | 备注 |
| :--- | :--- |
| Data | 纯文本你需要加密的内容 |
| Method | 你用`openssl_get_cipher_methods()`确定的一个方法，确定如下。 方法 - key\_size - cipher\_mode 所以，举例来说，如果你想采用AES的方法，密钥大小为256，并采用GCM模式，你可以输入`aes-256-gcm`。 |
| Password | 虽然这个参数被记录为密码，但它可以被看作是一个密钥。使用`random_bytes()`来生成一个与所需密钥大小相匹配的字节数的密钥。 |
| Options | 在你获得更多`openssl`加密经验之前，建议你坚持使用默认值0。 |
| IV | 使用`random_bytes()`来生成一个具有与密码方法相匹配的字节数的IV。 |

8. 举个例子，假设你想选择AES加密方式，密钥大小为256，XTS模式。下面是用于加密的代码。

```php
$plainText = 'Super Secret Credentials';
$key = random_bytes(16);
$method = 'aes-256-xts';
$cipherText = openssl_encrypt($plainText, $method, $key, 0, $iv);
```

9. 要解密，对`$key`和`$iv`使用相同的值，以及`openssl_decrypt()`函数。

```php
$plainText = openssl_decrypt($cipherText, $method, $key, 0, $iv);
```

## 如何运行...

为了查看可用的密码方法，创建一个名为`chap_12_openssl_encryption.php`的PHP脚本，然后运行这个命令。

```php
<?php
echo implode(', ', openssl_get_cipher_methods());
```

输出应该是这样的。

![](../../.gitbook/assets/image%20%28189%29.png)

接下来，可以为要加密的纯文本、方法、密钥和IV添加值。举个例子，试试使用XTS操作模式下的AES，密钥大小为256。

```php
$plainText = 'Super Secret Credentials';
$method = 'aes-256-xts';
$key = random_bytes(16);
$iv  = random_bytes(16);
```

要加密，你可以使用`openssl_encrypt()`，指定之前配置的参数。

```php
$cipherText = openssl_encrypt($plainText, $method, $key, 0, $iv);
```

你可能还想对结果进行base64编码，以使其更加可用。

```php
$cipherText = base64_encode($cipherText);
```

解密时，使用相同的`$key`和`$iv`值。不要忘了先解密base64值。

```php
$plainText = openssl_decrypt(base64_decode($cipherText), 
$method, $key, 0, $iv);
```

这里的输出显示的是base64编码的密码文本，然后是解密后的明文。

![](../../.gitbook/assets/image%20%28149%29.png)

如果你为IV提供的字节数不正确，对于所选择的密码方法，将显示一条警告信息。

![](../../.gitbook/assets/image%20%28185%29.png)

## 更多...

在 PHP 7 中，当使用 `open_ssl_encrypt()` 和 `open_ssl_decrypt()` 以及所支持的认证数据加密 \(AEAD\) 模式时出现了一个问题——GCM 和 CCM。因此，在 PHP 7.1 中，这些函数增加了三个额外的参数，如下所示。

| 参数 | 备注 |
| :--- | :--- |
| `$tag` | 通过引用传递的认证标签；如果认证失败，变量值保持不变。 |
| `$aad` | 额外的认证数据 |
| `$tag_length` | GCM模式为4-16；CCM模式无限制；仅适用于open\_ssl\_encrypt\(\) |

更多信息，您可以参考https://wiki.php.net/rfc/openssl\_aead。

## 更多...

关于为什么在 PHP 7.1 中取消 mcrypt 扩展的优秀讨论，请参考 https://wiki.php.net/rfc/mcrypt-viking-funeral 的文章。关于构成各种加密方法基础的块加密的良好描述，请参考 https://en.wikipedia.org/wiki/Block\_cipher 的文章。关于AES的优秀描述，请参考https://en.wikipedia.org/wiki/Advanced\_Encryption\_Standard。关于描述加密操作模式的优秀文章，可以参考https://en.wikipedia.org/wiki/Block\_cipher\_mode\_of\_operation。

{% hint style="info" %}
对于一些较新的模式，如果要加密的数据小于块大小，`openssl_decrypt()`将不返回任何值。如果你把数据填充到至少是块大小，问题就会消失。大多数的模式都实现了内部填充，所以这不是问题。对于一些较新的模式\(即xts\)，你可能会看到这个问题。在将你的代码投入生产之前，一定要对小于8个字符的短数据串进行测试。
{% endhint %}

