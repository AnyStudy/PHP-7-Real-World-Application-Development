# 前言

## PHP 7:真实世界的应用开发（中文翻译）

* 作者：[Doug Bierer](https://learning.oreilly.com/search/?query=author%3A%22Doug%20Bierer%22&sort=relevance&highlight=true), [Altaf Hussain](https://learning.oreilly.com/search/?query=author%3A%22Altaf%20Hussain%22&sort=relevance&highlight=true), [Branko Ajzele](https://learning.oreilly.com/search/?query=author%3A%22Branko%20Ajzele%22&sort=relevance&highlight=true)
* 原书名称：《[PHP 7: Real World Application Development](https://www.packtpub.com/php-7-real-world-application-development)》
* 译者：金弘扬（[ganymedenil@gmail.com](mailto:ganymedenil@gmail.com)）
* Gitbook地址：[PHP 7:真实世界的应用开发](https://ganymedenil.gitbook.io/php-7/)

推荐使用 Gitbook 以获取最佳阅读体验。

## 前言

PHP 7 在开源社区掀起了一场风暴，它打破了之前版本的速度记录，也重新引起了人们对它的关注。从最根本的意义上讲，核心工程团队已经对它进行了重大重写，但仍能保持高度的向后兼容性。PHP是一门开发Web应用的好语言。它本质上是一类服务器端脚本语言，也用于通用编程。PHP 7是最新的版本，提供了主要的向后兼容性突破，并专注于提高性能和速度。这意味着你可以通过多线程网络服务器，用低成本的硬件和服务器维持网站的高流量。

### 这条学习之路都涵盖了什么

模块1，PHP 7 编程指南，本模块以 PHP 7 为中心，展示了中高级的PHP技术。每个示例都是为了解决像您这样的 PHP 开发人员每天面临的实际问题。其中还介绍了只有在 PHP 7 中才有的，新的编写 PHP 代码的方法。此外，我们还讨论了向后兼容性中断的问题，并为您提供了大量指导，告诉您何时何地需要修改 PHP 5 代码，以便在 PHP 7 下运行时产生正确的结果。本模块还包含了最新的 PHP 7.x 特性。在本模块结束时，您将具备为您的网站和企业提供高效应用程序所需的工具和技能。

模块2，学习 PHP 7 高性能，该模块是 PHP 7 的快速入门，这将提高您的生产力和编码技能。所涉及的概念将使您作为一个PHP程序员，提高你的应用程序的性能标准。我们将向您介绍 PHP 7 中的新特性，然后介绍 PHP 7 中面向对象编程（OOP）的概念。接下来，我们将阐明如何提高 PHP 7 应用程序的性能和数据库性能。通过这个模块，您将能够使用模块中讨论的各种基准测试工具来提高程序的性能。最后，模块讨论了 PHP 编程中的一些最佳实践，以帮助你提高代码的质量。

模块3，用 PHP 7 更新旧版应用程序，此模块将向您展示如何通过提取和替换旧版组件，从实践和技术上而不是在使用框架和库之类的工具方面对应用程序进行升级。 我们将采用循序渐进的方法，有条不紊地缓慢前进，从根本上改善您的应用程序。我们将向您展示依赖注入是如何替换新的和全局依赖的。我们还将向您展示如何将表示逻辑改为视图文件，将动作逻辑改为控制器。此外，我们将使您的应用程序始终保持运行状态。在这个过程中，每一个完成的步骤都会让您的代码库以更高的质量完全正常运行。当我们完成后，您将能够像风一样轻而易举地通过您的代码。您的代码将是自动加载、依赖注入、单元测试、层级分离和前端控制。我们将添加到您的应用程序中的大多数非常有限的代码都是针对这个模块的。我们将以程序员的身份提高自己，并提高传统应用程序的质量。

### 你在这条学习之路上需要什么

#### 模块1

要成功地实现本模块中介绍的示例，你只需要一台计算机，100MB 的额外磁盘空间，和一个文本或代码编辑器（不是文字处理器！）。第一章将介绍如何设置 PHP 7 开发环境。拥有一个 Web 服务器是可选的，因为 PHP 7 包含一个开发 Web 服务器。不需 Internet 连接，但下载代码（如 PSR-7 接口集）和查看 PHP 7.x 文档可能会需要。

#### 模块2

任何符合运行以下软件最新版本的硬件规格，应该都足以通过本模块。

* 操作系统： Debian 或 Ubuntu
* 软件： NGINX、PHP 7、 MySQL、 PerconaDB、 Redis、 Memcached、 Xdebug、Apache JMeter、 ApacheBench、Siege 和 Git

#### 模块3

您需要参考本模块的“第二章，先决条件“来了解本模块所需的基本硬件和软件要求。本章将详细描述这些要求。

### 这条路是为谁而设

如果您是一个有抱负的Web开发人员，移动应用开发人员或后端程序员，并且具有PHP编程的基本经验并希望开发对性能至关重要的应用程序，那么这个课程是为你准备的。它将使您的PHP编程技能更上一层楼。

### 支持

课程的代码包也托管在github上  [https://GitHub.com/packtpublishing/php-7-be-pro-at-applications-development](https://GitHub.com/packtpublishing/php-7-be-pro-at-applications-development) 。

## 法律申明

译者纯粹出于**学习目的**与**个人兴趣**翻译本书，不追求任何经济利益。

译者保留对此版本译文的署名权，其他权利以原作者和出版社的主张为准。

本译文只供学习研究参考之用，不得公开传播发行或用于商业用途。有能力阅读英文书籍者请购买正版支持。

### LICENSE

CC-BY 4.0



