# 开闭原则

开闭原则根据维基百科上的定义，一个类的扩展应该是开放的，但修改应该是封闭的。

> _软件中的对象（类，模块，函数等等）应该对于扩展是开放的，但是对于修改是封闭的_

开放的扩展部分意味着我们应该设计我们的类，以便在需要时可以添加新的功能。封闭的修改部分是指这个新功能应该适合在不修改原类的情况下使用。只有在修复bug的情况下才应该修改类，而不是为了增加新功能。

下面是一个违反开闭原则的类的例子：

```php
class CsvExporter {
    public function export($data) {
        // Implementation...
    }
}

class XmlExporter {
    public function export($data) {
        // Implementation...
    }
}

class GenericExporter {
    public function exportToFormat($data, $format) {
        if ('csv' === $format) {
            $exporter = new CsvExporter();
        } elseif ('xml' === $format) {
            $exporter = new XmlExporter();
        } else {
            throw new \Exception('Unknown export format!');
        }
        return $exporter->export($data);
    }
}
```

在这里，我们有两个具体的类，`CsvExporter`和`XmlExporter`，每个类都有一个职责。然后我们有一个`GenericExporter`，它的`exportToFormat`方法实际上是在一个适当的实例类型上触发导出函数。这里的问题是，我们不能在不修改`GenericExporter`类的情况下，增加一个新的类型`exporter`。换句话说，`GenericExporter`不开放扩展，封闭修改。

下面是一个重构后的实现例子，它符合OCP的要求：

```php
interface ExporterFactoryInterface {
    public function buildForFormat($format);
}

interface ExporterInterface {
    public function export($data);
}

class CsvExporter implements ExporterInterface {
    public function export($data) {
        // Implementation...
    }
}

class XmlExporter implements ExporterInterface {
    public function export($data) {
        // Implementation...
    }
}

class ExporterFactory implements ExporterFactoryInterface {
    private $factories = array();

    public function addExporterFactory($format, callable $factory) {
          $this->factories[$format] = $factory;
    }

    public function buildForFormat($format) {
        $factory = $this->factories[$format];
        $exporter = $factory(); // the factory is a callable

        return $exporter;
    }
}

class GenericExporter {
    private $exporterFactory;

    public function __construct(ExporterFactoryInterface $exporterFactory) {
        $this->exporterFactory = $exporterFactory;
    }

    public function exportToFormat($data, $format) {
        $exporter = $this->exporterFactory->buildForFormat($format);
        return $exporter->export($data);
    }
}

// Client
$exporterFactory = new ExporterFactory();

$exporterFactory->addExporterFactory(
'xml',
    function () {
        return new XmlExporter();
    }
);

$exporterFactory->addExporterFactory(
'csv',
    function () {
        return new CsvExporter();
    }
);

$data = array(/* ... some export data ... */);
$genericExporter = new GenericExporter($exporterFactory);
$csvEncodedData = $genericExporter->exportToFormat($data, 'csv');
```

这里我们添加了两个接口，`ExporterFactoryInterface`和`ExporterInterface`。然后我们修改`CsvExporter`和`XmlExporter`，实现该接口。添加了`ExporterFactory`，实现了`ExporterFactoryInterface`。它的主要作用是由`buildForFormat`方法定义的，该方法作为回调函数返回`exporter`。最后，重写了`GenericExporter`，通过它的构造函数接受`ExporterFactoryInterface`，它的`exportToFormat`方法现在通过使用`exporter`工厂来构建`exporter`，并调用其上的`execute`方法。

现在，客户端本身已经发挥了更强大的作用，它首先实例化`ExporterFactory`，并向其添加两个导出器，然后再将其传递给`GenericExporter`。现在向`GenericExporte`r添加一个新的导出格式，不再需要修改它，因为它的扩展是开放的，而修改是封闭的。同样，这绝不是一个通用的公式，而是一个满足OCP的可能方法的概念。

