# 线框设计

有了用户故事的铺垫，让我们把重点转移到实际的线框设计上来。出于我们稍后会讲到的原因，我们的线框工作将围绕客户的角度进行。

目前有很多线框工具，有免费的也有商业的。一些商业工具，如https://ninjamock.com，我们将在我们的例子中使用，仍然提供免费计划。这对于个人项目来说是非常方便的，因为它为我们节省了很多时间。

每一个网络应用的起点都是它的主页。下面的线框说明了我们网店应用的主页。

![](../../.gitbook/assets/image%20%28160%29.png)

在这里我们可以看到决定页面结构的几个部分。头部由logo、分类菜单和用户菜单组成。需求中没有提到任何关于类别结构的内容，而我们正在构建一个简单的网店应用，所以我们将坚持使用扁平化的类别结构，没有任何子类别。用户菜单最初将显示注册和登录链接，直到用户真正登录，在这种情况下，菜单将发生变化，如以下线框所示。内容区域充满了畅销商品和促销商品，每个商品都定义了图片、标题、价格和添加到购物车按钮。页脚区域包含了大部分静态内容页面的链接，以及一个联系我们页面。

下面的线框说明了我们网店应用的分类页面。

![](../../.gitbook/assets/image%20%28190%29.png)

整个网站的页眉和页脚区域在概念上保持不变。内容区域现在已改为列出任何给定类别内的产品。单个产品区域的呈现方式与主页上相同。类别名称和图片呈现在产品列表上方。类别图片的宽度给出了一些提示，说明我们应该准备什么类型的图片并上传到我们的类别中。

下面的线框说明了我们网店应用的产品页面。

![](../../.gitbook/assets/image%20%28205%29.png)

现在，这里的内容区域改变为列出单个产品信息。我们可以看到一个大的图片占位符、标题、sku、库存状态、价格、数量字段、添加到购物车按钮和产品描述正在呈现。当商品可以购买时，要显示IN STOCK信息，当商品不再有货时，要显示OUT OF STOCK信息。这要和产品数量属性相关。我们还需要牢记 "以大视角（缩放）查看产品图片 "的要求，点击图片会放大图片。

下面的线框说明了我们网店应用的注册页面。

![](../../.gitbook/assets/image%20%28157%29.png)

现在这里的内容区会改变，呈现一个注册表。我们有很多方法可以实现注册系统。更多的时候，在注册屏幕上询问的信息量是最小的，因为我们想让用户尽快进入。然而，让我们在注册屏幕上尝试获取更完整的用户信息。我们不仅要求用户提供电子邮件和密码，还要求用户提供整个地址信息。

下面的线框说明了我们网店应用的登录页面。

![](../../.gitbook/assets/image%20%28200%29.png)

现在这里的内容区域改变为呈现客户登录和忘记密码的表单。我们为用户提供登录时的Email和密码字段，或者在密码重置操作时只提供Email字段。

下面的线框说明了我们网店应用的客户账户页面。

![](../../.gitbook/assets/image%20%28165%29.png)

现在这里的内容区域变为呈现客户账户区域，只有登录的客户才能看到。在这里我们看到一个屏幕，主要有两个信息。客户信息是一个，订单历史是另一个。客户可以在这个屏幕上修改自己的邮箱、密码等地址信息。此外，客户还可以查看、取消、打印之前的所有订单。我的订单表从上到下，从最新到最旧列出订单。虽然用户故事没有指定，但订单取消应该只对待定订单有效。这一点我们会在后面详细介绍。

这也是用户登录时显示用户菜单状态的第一屏。我们可以看到一个下拉菜单，显示了用户的全名、我的账户和注销链接。就在它的旁边，我们有购物车（%s）链接，这是为了列出购物车中的确切数量。

下面的线框说明了我们的网店应用的结账购物车页面。

![](../../.gitbook/assets/image%20%28187%29.png)

现在这里的内容区域会改变，以呈现购物车的当前状态。如果客户在购物车中添加了任何产品，它们将在这里列出。每个项目应该列出产品的标题、单个价格、添加的数量和小计。顾客应该能够改变数量，并按更新购物车按钮来更新购物车的状态。如果提供的数量为0，点击更新购物车按钮将从购物车中删除该商品。购物车的数量应该在任何时候都反映出标题菜单购物车（%s）链接的状态。屏幕的右侧显示了当前订单总价值的快速摘要，旁边还有一个大而清晰的 "去结账 "按钮。

下面的线框说明了我们的网店应用的结账购物车发货页面。

![](../../.gitbook/assets/image%20%28184%29.png)

这里的内容区域现在改变为呈现结账过程的第一步，即收集运输信息。未登录的客户应该无法访问此屏幕。客户可以在这里向我们提供他们的地址详细信息，同时选择送货方式。送货方式区域列出了几种送货方式。在右侧，显示了可折叠的订单摘要部分，列出了当前购物车中的项目。在它的下方，我们有购物车的小计值和一个大大的清晰的下一步按钮。只有在提供了所有所需信息后，下一步按钮才会触发，在这种情况下，它应该把我们带到结账车付款页面的付款信息。

下面的线框说明了我们的网店应用的结账车支付页面。

![](../../.gitbook/assets/image%20%28203%29.png)

这里的内容区域现在改变为呈现结账流程的第二步，即付款信息收集。未登录的客户不应访问此屏幕。客户会看到一个可用的支付方式列表。为了应用的简单性，我们将只关注统一/固定的支付方式，而不是像PayPal或Stripe这样的强大支付方式。在屏幕的右侧，我们可以看到一个可折叠的订单摘要部分，列出购物车中的当前项目。在它的下方，我们有订单总数部分，分别列出了购物车小计、标准送货、订单总数，以及一个大大的清晰的下单按钮。只有在提供了所有所需信息的情况下，下单按钮才会触发，在这种情况下，它应该把我们带到结账成功的页面。

下面的线框说明了我们的网络商店应用程序的结账成功页面。

![](../../.gitbook/assets/image%20%28175%29.png)

现在这里的内容区域会改变，输出结账成功的信息。显然，这个页面只有刚刚完成结账过程的登录客户才能看到。订单号是可以点击的，并链接到 "我的账户 "区域，关注具体订单。到了这个页面，客户和店长都应该会收到一封通知邮件，按照订单下达后获得邮件通知和如果新的销售订单已经创建就会被通知的要求。

到此，我们面向客户的线框就结束了。

关于店长用户故事需求，我们暂时只定义一个登陆管理界面，如下截图所示。

![](../../.gitbook/assets/image%20%28179%29.png)

后面使用框架，我们将得到一个完整的自动生成的CRUD接口，用于多个Add New和List & Manage链接。对这个界面及其链接的访问将由框架的安全组件控制，因为这个用户不会是客户或数据库中的任何用户。

此外，在下面的章节中，我们将把我们的应用程序分成几个模块。在这样的设置中，每个模块将拥有单独的功能，负责客户、目录、结账和其他需求。

