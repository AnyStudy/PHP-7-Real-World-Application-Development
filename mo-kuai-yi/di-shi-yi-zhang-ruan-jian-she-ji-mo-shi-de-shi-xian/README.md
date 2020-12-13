# 第十一章、软件设计模式的实现

在本章中，我们将介绍以下主题：

* 创建数组到对象的转化器 
* 构建对象到数组到转化器
* 实施策略模式 
* 定义一个映射器 
* 实现对象关系映射 
* 实施发布/订阅设计模式

## 前言

将软件设计模式融入到面向对象编程（OOP）代码中的想法，最早是在1994年由著名的 "四人帮"（E.Gamma，R.Helm，R.Johnson和J.Vlissides）撰写的《设计模式：可复用面向对象软件设计》中提出。这项工作既没有定义标准，也没有定义协议，它确定了多年来证明有用的通用软件设计。本书所讨论的模式一般被认为可以分为三类：创造型、结构型和行为型。

本书中已经介绍了其中许多模式的例子。下面是一个简单的总结。



<table>
  <thead>
    <tr>
      <th style="text-align:left">&#x8BBE;&#x8BA1;&#x6A21;&#x5F0F;</th>
      <th style="text-align:left">&#x7AE0;&#x8282;</th>
      <th style="text-align:left">&#x6848;&#x4F8B;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">&#x5355;&#x4F8B;</td>
      <td style="text-align:left">2</td>
      <td style="text-align:left">&#x5B9A;&#x4E49;&#x53EF;&#x89C1;&#x6027;</td>
    </tr>
    <tr>
      <td style="text-align:left">&#x5DE5;&#x5382;</td>
      <td style="text-align:left">6</td>
      <td style="text-align:left">&#x5B9E;&#x73B0;&#x8868;&#x683C;&#x5DE5;&#x5382;</td>
    </tr>
    <tr>
      <td style="text-align:left">&#x9002;&#x914D;&#x5668;</td>
      <td style="text-align:left">8</td>
      <td style="text-align:left">&#x4E0D;&#x4F7F;&#x7528; <code>gettext()</code> &#x5904;&#x7406;&#x7FFB;&#x8BD1;</td>
    </tr>
    <tr>
      <td style="text-align:left">&#x4EE3;&#x7406;</td>
      <td style="text-align:left">7</td>
      <td style="text-align:left">
        <p>&#x521B;&#x5EFA;&#x4E00;&#x4E2A;&#x7B80;&#x5355;&#x7684;REST&#x5BA2;&#x6237;&#x7AEF;</p>
        <p>&#x521B;&#x5EFA;&#x4E00;&#x4E2A;&#x7B80;&#x5355;&#x7684;SOAP&#x5BA2;&#x6237;&#x7AEF;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">&#x8FED;&#x4EE3;&#x5668;</td>
      <td style="text-align:left">
        <p>2</p>
        <p>3</p>
      </td>
      <td style="text-align:left">
        <p>&#x9012;&#x5F52;&#x76EE;&#x5F55;&#x8FED;&#x4EE3;&#x5668;</p>
        <p>&#x4F7F;&#x7528;&#x8FED;&#x4EE3;&#x5668;</p>
      </td>
    </tr>
  </tbody>
</table>

在本章中，我们将研究许多其他设计模式，主要侧重于并发和体系结构模式。

