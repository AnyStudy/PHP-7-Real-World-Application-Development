# 第九章、开发中间件

在本章中，我们将涉及以下主题。

* 使用中间件进行认证 
* 使用中间件实现访问控制
*  使用高速缓存提高性能 
* 实施路由选择 
* 进行框架间的系统调用 
* 使用中间件来跨语言

## 前言

正如IT行业经常发生的那样，术语被发明出来，然后被使用和滥用。中间件这个词也不例外。可以说，这个术语的第一次使用是在2000年由互联网工程任务组（IETF）提出的。最初，这个术语适用于任何在传输层（即TCP/IP）和应用层之间操作的软件。最近，特别是随着PHP标准建议7（PSR-7）的接受，中间件，特别是PHP世界中的中间件，已经被应用于Web客户-服务器环境。

{% hint style="info" %}
本节中的事例将使用附录《定义PSR-7类》中定义的具体类。
{% endhint %}



