---
title: "ZendFramework: ServiceManager"
description: "Рассмотрим пакет реализации паттерна Локатор служб во фреймворке ZendFramework..."
categories: backend
tags: ["zend framework", "zf", "service manager", "локатор служб"]
comments: true
---
Пакет [Zend-ServiceManager](https://github.com/zendframework/zend-servicemanager) представляет реализацию паттерна [Локатор служб](/posts/service-locator) с некоторыми дополнительными элементами.

# ServiceManager и PluginManager
Базовым элементом пакета является класс `Zend\ServiceManager\ServiceManager`, реализующий интерфейс `Zend\ServiceManager\ServiceLocatorInterface` и включающий следующие основные методы:

* `get` - предоставляет сервис
* `build` - создает сервис с использованием опций
* `setService` - добавляет сервис
* `setFactory` - добавляет фабрику
* `setAlias` - добавляет псевдоним
* `setShared` - устанавливает шаринг
* `setInvokableClass` - добавляет Invokable-фабрику
* `addAbstractFactory` - добавляет абстрактную фабрику
* `addDelegator` - добавляет делегатор
* `addInitializer` - добавляет инициализатор
* `setAllowOverride` - определяет возможность перезаписывать уже установленные сервисы

> Учтите, что интерфейс `ServiceLocatorInterface` наследует [`Interop\Container\ContainerInterface`](https://github.com/container-interop/container-interop) и дополняет его методом `build`, все же остальные методы являются нестандартными расширениями _ZendFramework_.

Каждый конкретный метод данного класса более подробно обсуждается ниже, а пока рассмотрим постой пример использования:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;

$locator = new ServiceManager;
// Добавление сервиса

$locator->setService('MyService', new MyService);

$locator->setService('MyService', ...); // Error: повторная регистрация сервиса приведет к ошибке.


// Получение сервиса

$myService = $locator->get('MyService');
{% endhighlight %}

> Помните, что по умолчанию нельзя добавлять _Локатору служб_ сервис, который уже был зарегистрирован в нем ранее. Для обхода этого ограничения необходимо сконфигурировать локатор с помощью следующего вызова: `$locator->setAllowOverride(true)`.

Класс `Zend\ServiceManager\AbstractPluginManager` является абстрактной реализацией интерфейса `Zend\ServiceManager\PluginManagerInterface` и отличает от _Локатора служб_ только тем, что ограничивает свою область ответственности только сервисами конкретных классов. Это удобно, если вам необходимо использовать _Локатор служб_ для хранения и предоставления, к примеру, _Контроллеров_ или _Хелперов представления_.

Для реализации собственного _Менеджера плагинов_ можно использовать как собственную реализацию интерфейса `PluginManagerInterface`, так и расширение абстракции `AbstractPluginManager`.

Через реализацию интерфейса:

{% highlight php %}
<?php
use Zend\ServiceManager\PluginManagerInterface;
use Zend\ServiceManager\ServiceManager;
use Zend\ServiceManager\Exception\InvalidServiceException;

class MyPluginManager extends ServiceManager implements PluginManagerInterface{
  // Стандартные плагины менеджера.

  protected $factories = [
    'MyService' => MyServiceFactory::class,
  ];

  ...

  // Метод проверки сервиса на допустимость интерфейса.

  public function validate($instance){
    if(!$instance instanceof MyServiceInterface){
      throw new InvalidServiceException(
          sprintf('Недопустимый сервис "%s". Необходима реализация "%s"',
            get_class($instance),
            MyServiceInterface::class
          )
      );
    }
  }

  // Расширение метода get валидацией получаемых сущностей.

  public function get($name, array $options = null){
    $instance = empty($options)? parent::get($name) : parent::build($name, $options);
    $this->validate($instance);

    return $instance;
  }
}
{% endhighlight %}

Через расширение абстракции:

{% highlight php %}
<?php
use Zend\ServiceManager\AbstractPluginManager;

class MyPluginManager extends AbstractPluginManager{
  // Стандартные плагины менеджера.

  protected $factories = [
    'MyService' => MyServiceFactory::class,
  ];

  ...

  // Ограничитель семантики. Только классы, являющиеся или наследующие данный могут быть в данном локаторе.

  protected $instanceOf = MyServiceInterface::class;
}
{% endhighlight %}

Через переопределение абстракции:

{% highlight php %}
<?php
use Zend\ServiceManager\AbstractPluginManager;

class MyPluginManager extends AbstractPluginManager{
  // Стандартные плагины менеджера.

  protected $factories = [
    'MyService' => MyServiceFactory::class,
  ];

  ...

  // Метод проверки сервиса на допустимость интерфейса. Переопределен для определения более одного допустимого родителя.

  public function validate($instance){
    if(!$instance instanceof FirstServiceInterface && !$instance instanceof SecondServiceInterface){
      throw new InvalidServiceException(
          sprintf('Недопустимый сервис "%s". Необходима реализация "%s" или "%s"',
            get_class($instance),
            FirstServiceInterface::class,
            SecondServiceInterface::class
          )
      );
    }
  }
}
{% endhighlight %}

Как, возможно, вы уже заметили, `PluginManager` может быть проинициализирован сервисами в момент объявления, для этого достаточно проинициализировать следующие его свойства (в листинге приведены примеры инициализации):

{% highlight php %}
<?php
use Zend\ServiceManager\AbstractPluginManager;

class MyPluginManager extends AbstractPluginManager{
  // Стандартные плагины менеджера.

  protected $services = [
    // Сервисы

    'config' => [
      ...
    ],
  ];

  protected $factories = [
    // Фабрики

    'MyService' => MyServiceFactory::class,
  ];

  protected $aliases = [
    // Псевдонимы

    'MyService2' => 'MyService',
  ];

  protected $shared = [
    // Шаринг

    'MyService' => false,
  ];

  protected $abstractFactories = [
    // Абстрактные фабрики

    MyAbstractFactory::class,
  ];

  protected $delegators = [
    // Делегаторы

    'MyService' => MyServiceDelegator::class,
  ];

  protected $initializers = [
    // Инициализаторы

    MyInitializer::class,
  ];

  // Реализация ограничителя.

  ...
}
{% endhighlight %}

При инстанциации такого _Менеджера плагинов_, он будет заполнен указанными сервисами и готов к использованию. Часто это применяется для создания небольшого локатора, на пример так - [_Менеджер плагинов_ для _Хелперов представления_](https://github.com/zendframework/zend-view/blob/master/src/HelperPluginManager.php).

# Сервисы
Сервисами в _ZendFramework_ могут являться любые данные и объекты. Их отличает то, что они создаются в момент заполнения контейнера и всегда предоставляются шарингом:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;

$locator = new ServiceManager;

// Регистрация сервиса

$locator->setService('Configuration', include(__DIR__ . '/config/app.config.php'));

...

// Запрос сервиса

$conf = $locator->get('Configuration');
{% endhighlight %}

# Фабрики
Фабрики позволяют создавать сервисы в момент их запроса из контейнера. В качестве фабрики может выступать как анонимная функция, так и класс, реализующий интерфейс `Zend\ServiceManager\Factory\FactoryInterface`:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;
use Interop\Container\ContainerInterface;

$locator = new ServiceManager;

// Регистрация фабрики

$locator->setFactory('MyService', function(ContainerInterface $container, $requestedName, array $options = null){
    $conf = is_null($options)? $container->get('Configuration')['my_service'] : $options;

  return new MyService($conf);
});

// Вызов фабрики с передачей дополнительных опций

$myservice = $locator->build('MyService', [...]);
{% endhighlight %}

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;
use Interop\Container\ContainerInterface;
use Zend\ServiceManager\Factory\FactoryInterface;

// Класс фабрики

class MyServiceFactory implements FactoryInterface{
  public function __invoke(ContainerInterface $container, $requestedName, array $options = null){
    $conf = is_null($options)? $container->get('Configuration')['my_service'] : $options;

    return new MyService($conf);
  }
}

$locator = new ServiceManager;

// Регистрация фабрики без инстанциации

$locator->setFactory('MyService', MyServiceFactory::class);

// Вызов фабрики без передачи дополнительных опций

$myservice = $locator->build('MyService');
{% endhighlight %}

> Метод `build`, используемый в примерах, позволяет не только указать опции для фабрики, но и исключает шаринг, даже если он включен. В противном же случае можно использовать метод `get` для получения сервисов из фабрик.

> При регистрации класса-фабрики можно как указать имя ее класса (как это сделано в примере), так и передать экземпляр. Во втором случае можно проинициализировать фабрику через конструктор.

## Invokable-фабрики
Если вам необходима фабрика, создающая экземпляры сервиса без передачи констуктору аргументов или с заданным их набором, можно воспользоваться готовой реализацией в виде класса `Zend\ServiceManager\Factory\InvokableFactory`:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;
use Zend\ServiceManager\Factory\InvokableFactory;

class MyService{
  // Создаваемый с помощью InvokableFactory сервис должен либо принимать в конструкторе необязательный массив, либо не иметь аргументов вовсе

  public function __construct(array $options = []){
    ...
  }
}

$locator = new ServiceManager;

$locator->setFactory('MyService', InvokableFactory::class);

// Вызов фабрики без передачи аргументов в констуктор сервиса

$myservice = $locator->build('MyService');

// Вызов фабрики с передачей аргументов в констуктор сервиса

$myservice = $locator->build('MyService', [...]);
{% endhighlight %}

Метод `setInvokableClass` является фассадным и позволяет регистрировать `InvokableFactory` без прямого взаимодействия с этим классом:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;

$locator = new ServiceManager;

// Аналогично: $locator->setFactory(MyService::class, InvokableFactory::class);

$locator->setInvokableClass('MyService', MyService::class);

$myservice = $locator->get('MyService');
{% endhighlight %}

> Метод `setInvokableClass` помимо прочего, так же позволяет использовать любое имя сервиса, а не только полное имя инстанциируемого класса.

# Псевдонимы
Метод `setAlias` позволяет зарегистрировать любой сервис или фабрику под несколькими именами:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;

$locator = new ServiceManager;
$locator->setService('MyService', new MyService);

// Добавление псевдонима

$locator->setAlias('ms', 'MyService');
{% endhighlight %}

Чаще всего это применяется в _Менеджерах плагинов_ для сокращения имен часто используемых сервисов.

# Шаринг
Шаринг позволяет кешировать созданные фабриками сервисы, что исключает повторный вызов фабрики:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;

$locator = new ServiceManager;
$locator->setFactory('MyService', function(){
  return new MyService;
});

// Шаринг сервиса MyService

$locator->setShared('MyService', true);
echo $locator->get('MyService') === $locator->get('MyService'); // true


// Отключение шаринга сервиса MyService

$locator->setShared('MyService', false);
echo $locator->get('MyService') === $locator->get('MyService'); // false
{% endhighlight %}

> Помните, что при запросе сервиса с помощью метода `build`, настройки шаринга не действуют.

# Абстрактные фабрики
Абстрактные фабрики позволяют создавать сервисы, незарегистрированные ранее в _Локаторе служб_. Это может быть полезно в случае, если вам необходимо создание сервиса по умолчанию:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;
use Interop\Container\ContainerInterface;
use Zend\ServiceManager\Factory\AbstractFactoryInterface;

// Класс абстрактной фабрики

class DefaultControllerAbstractFactory implements AbstractFactoryInterface{
  public function canCreate(ContainerInterface $container, $requestedName){
    // Абстрактная фабрика создает контроллер по умолчанию, если класса запрашиваемого контроллера не существует

    return strpos($requestedName, 'Application\Controller') == 0 && !class_exists($requestedName);
  }

  public function __invoke(ContainerInterface $container, $requestedName, array $options = null){
    return new DefaultController($options);
  }
}

$locator = new ServiceManager;

$locator->addAbstractFactory(DefaultControllerAbstractFactory::class);

$locator->get('Application\Controller\Index');
{% endhighlight %}

В отличии от обычных фабрик, абстрактные вызываются только в том случае, если сервис с данным именем не зарегистрирован и метод `canCreate` абстрактной фабрики возвращает `true`. Это позволяет зарегистрировать несколько таких фабрик, обеспечивая тем самым гарантированный возврат некотого сервиса.

> Как и при регистрации обычных фабрик, метод `addAbstractFactory` может принимать имя класса абстрактной фабрики или его экземпляр.

# Делегаторы
Делегаторы это специальные классы или анонимные функции, способные расширять или изменять функционал зарегистрированных в _Локаторе служб_ сервисов. Они регистрируются на то же имя, что и имеющийся сервис и вызываются последовательно, взаимодействуя с сервисом по принципу "конвейера".

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;
use Zend\ServiceManager\Factory\DelegatorFactoryInterface;
use Application\Controller\Index as IndexController;

// Класс делегатора

class ControllerDependenciesInjectorDelegator implements DelegatorFactoryInterface{
  public function __invoke(ContainerInterface $container, $name, callable $callback, array $options = null){
    // Создание сервиса

    $controller = call_user_func($callback);

    // Внедрение зависимости сервиса

    $controller->setCurrentUser($container->get('CurrentUser'));

    return $controller;
  }
}

$locator = new ServiceManager;

$locator->setFactory(IndexController::class, InvokableFactory::class);
$locator->addDelegator(IndexController::class, ControllerDependenciesInjectorDelegator::class);
{% endhighlight %}

В данном примере делегатор используется для внедрения зависимости в контроллер.

# Инициализаторы
Инициализаторы похоже на делегаторы, но регистрируются не на конкретный сервис, а вызываются для всех доступных в _Локаторе служб_:

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;
use Zend\ServiceManager\Initializer\InitializerInterface;
use Interop\Container\ContainerInterface;
use Zend\Mvc\Controller\AbstractController;

// Класс инициализатора

class ControllerDependenciesInjectorInitializer implements InitializerInterface{
  public function __invoke(ContainerInterface $container, $instance){
    // Инициализация только контроллеров

    if(!$instance instanceof AbstractController){
      return;
    }

    $instance->setCurrentUser($container->get('CurrentUser'));
    ...
  }
}

$locator = new ServiceManager;

$locator->setFactory(IndexController::class, InvokableFactory::class);
$locator->addInitializer(ControllerDependenciesInjectorInitializer::class);
{% endhighlight %}

Приведенный в примере инициализатор способен работать с любыми контроллерами приложения, что упрощает конфигурацию _Локатора служб_ (нет необходимости регистрировать делегатор для каждого контроллера в отдельности), но так же увеличивает нагрузку за счет проверки всех запрашиваемых из локатора сервисов. Учитывая это, старайтесь использовать делегатор везде, где это возможно, а инициализаторы применять только в особых случаях.

# Конфигурация локатора
Класс `Zend\ServiceManager\ServiceManager` может быть сконфигурирован декларативно (с использованием массива PHP). Для этого используется конструктор или метод `configure`. При этом массив конфигурации может включать следующие ключи:

* `services` - массив сервисов
* `factories` - массив фабрик
* `invokables` - массив `InvokableFactory`
* `aliases` - массив псевдонимов
* `shared` - конфигурация шаринга
* `abstract_factories` - массив абстрактных фабрик
* `delegators` - массив делегаторов
* `initializers` - массив инициализаторов

{% highlight php %}
<?php
use Zend\ServiceManager\ServiceManager;

$locator = new ServiceManager([
  'services' => [
    'config' => [...],
  ],
  'factories' => [
    IndexController::class => IndexControllerFactory::class,
  ],
  'invokables' => [
    'orderController' => OrderController::class,
  ],
  'aliases' => [
    'indexController' => IndexController::class,
  ],
  'shared' => [
    IndexController::class => false,
  ],
  'abstract_factories' => [
    DefaultControllerAbstractFactory::class,
  ],
  'delegators' => [
    IndexController::class => ControllerDependenciesInjectorDelegator::class,
  ],
  'initializers' => [
    ControllerDependenciesInjectorInitializer::class,
  ],
]);
{% endhighlight %}
