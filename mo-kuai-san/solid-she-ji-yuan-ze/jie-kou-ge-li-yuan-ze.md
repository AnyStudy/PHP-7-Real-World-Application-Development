# 接口隔离原则

接口隔离原则指出，客户应该只实现他们实际使用的接口。他们不应该被强迫实现他们不使用的接口。根据维基百科上的定义：

> _多个客户端专用接口比一个通用接口要好_

这意味着，我们应该把大而肥的接口分成若干个小而轻的接口，把它隔离开来，让小的接口以方法组为基础，每个方法服务于一个特定的功能。

我们来看看下面这个违反ISP的例子：

```php
interface Appliance {
    public function powerOn();
    public function powerOff();
    public function bake();
    public function mix();
    public function wash();

}

class Oven implements Appliance {
    public function powerOn() { /* Implement ... */ }
    public function powerOff() { /* Implement ... */ }
    public function bake() { /* Implement... */ }
    public function mix() { /* Nothing to implement ... */ }
    public function wash() { /* Cannot implement... */ }
}

class Mixer implements Appliance {
    public function powerOn() { /* Implement... */ }
    public function powerOff() { /* Implement... */ }
    public function bake() { /* Cannot implement... */ }
    public function mix() { /* Implement... */ }
    public function wash() { /* Cannot implement... */ }
}

class WashingMachine implements Appliance {
    public function powerOn() { /* Implement... */ }
    public function powerOff() { /* Implement... */ }
    public function bake() { /* Cannot implement... */ }
    public function mix() { /* Cannot implement... */ }
    public function wash() { /* Implement... */ }
}
```

在这里，我们有一个接口为几个设备相关的方法设置需求。然后我们有几个类实现这个接口。问题很明显，不是所有的家电都能挤进同一个接口。对于一台洗衣机来说，强迫它实现烘烤和混合方法是没有意义的。这些方法需要各自拆分到自己的接口中。这样具体的家电类就只能实现真正有意义的方法。

