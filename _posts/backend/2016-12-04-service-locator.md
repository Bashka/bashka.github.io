---
title: "Паттерн: Локатор служб"
description: "О том, что такое и как лучше применять наиболее часто используемый (по мнению автора) паттерн..."
categories: backend
tags: ["pattern", "паттерн", "локатор служб", "service locator"]
comments: true
---
# Семантика
Локатор служб (или Service Locator) это паттерн, используемый для решения задачи хранения, создания и глобального доступа к сервисам приложения. Он часто применяется не осознанно путем создания "Божественного объекта", хранящего все функции программы и расположенного в глобальном пространстве имен (namespace). Естественно такая реализация сулит множество проблем, которые могут быть решены с помощью самого действенного в программировании совета - разделяй и властвуй. В результате данный патерн предлагает реализовывать все вспомогательные службы (классы, объекты, функции) в виде независимых компонентов и хранить их (либо создавать в) локаторе служб - объекте, доступном глобально или базовому элементу системы.

Семантика паттерна может быть представлена схематически следующим образом:

![service locator](https://www.plantuml.com/plantuml/png/oymhIIrAIqnELGXEBIhBJ4xroKzEBCalyeI9LtCfA6Ga5ciKGwJz4dDJDO52wQabg4ArN5p9EOd5nMZcWBJhAa1L5dC1UOafASWwTM2IciHRX6g5WcvfWMwD7IuF0000 "Локатор служб")

Как видите, все очень просто. Паттерн реализуется через объект, способный хранить множество служб (объектов) в себе и предоставлять их по требованию (при вызове метода `get`).

В качестве реальных примеров описания данного паттерна можно привести следующие:

1. [PSR-11](https://github.com/php-fig/fig-standards/blob/master/proposed/container.md#2-interfaces) - один из FIG-стандартнов, описывающих семантику интерфейса локатора служб
2. [ContainerInterop](https://github.com/container-interop/container-interop) - наиболее распространенная и поддерживаемая семантическая реализация этого паттерна

На первый взгляд все просто, не так ли? Осталось только обговорить некоторые детали и приступим к реализации:

Скалярные данные
: Локатор служб может хранить не только объекты, а, к примеру, конфигурацию приложения в виде массива или любые скалярные данные.

Фабрики
: Метод `get` локатора служб не обязует его предоставлять данный ему ранее на хранение объект (службу). На деле это более общий метод, который отвечает за предоставление службы по ее имени, даже если этой службы еще не существует (объект не создан). Делается это с помощью _Фабрик_. Фабрика это функция или целый класс, способный создавать объекты других классов и предоставлять их локатору служб по требованию.

Шаринг
: Если паттерн предполагает создание сервисов при их запросе, то как ограничить этот процесс, если это слишком затратная операция (как, на пример, соединение с базой данных)? Для этих целей локатор служб должен поддерживать _Карту шаринга_, которая разрешает или запрещает повторное создание сервиса при каждом его запросе.

Псевдонимы
: Иногда один и тот же сервис необходимо предоставлять под разными именами. К примеру, одно имя "Машинное", а другое "Человекочитаемое". Для этих целей служат псевдонимы, по сути являющиеся ссылками на уже имеющиеся в локаторе служб сервисы.

Иерархия
: Ничто не запрещает вложить один локатор служб в другой в виде сервиса. Такая организация позволит сформировать иерархию, которая, в свою очередь, поможет разделить ответственность за предоставление сервисов различных типов между несколькими хранилищами (еще одно название локатора служб).

# Реализация
Простейшей реализацией данного паттерна служит следующий класс:

{% highlight php %}
<?php
class ServiceLocator{
  private $services = [];

  public function add($serviceName, $service){
    $this->services[$serviceName] = $service;
  }

  public function has($serviceName){
    return isset($this->services[$serviceName]);
  }

  public function get($serviceName){
    return $this->services[$serviceName];
  }
}
{% endhighlight %}

Использоваться такой класс может следующим образом:

{% highlight php %}
<?php
class App{
  public static $serviceLocator;

  ...
}

App::$serviceLocator = new ServiceLocator;

// Регистрация сервисов

App::$serviceLocator->add('config', [
  'database' => [
    'dbname' => 'my_db',
    'user' => 'root',
    'pass' => '123',
  ],
]);
App::$serviceLocator->add('Log', new FileLog);

// Где то в другой части приложения

App::get('Log')->error('Все очень плохо!'); // Получение сервиса
{% endhighlight %}

## Фабрики
Для понимания работы фабрик достаточно рассмотреть пример с сервисом доступа к базе данных. Предположим нам необходимо работать с базой данных через интерфейс [PDO](https://php.net/manual/ru/book.pdo.php). Для этого необходимо предварительно зарегистрировать экземпляр данного класса в локаторе служб следующим образом:

{% highlight php %}
<?php
$dbConfig = $serviceLocator->get('config')['database'];
$serviceLocator->add('DataBase', new PDO(...));
{% endhighlight %}

Но это создает две проблемы:

1. Зачем нам выполнять подключение к базе данных в момент добавления сервиса в локатор?
2. Почему логика создания экземпляра класса `PDO` в ответственности локатора?

Обе эти проблемы решаются с использованием фабрик, которые представляют собой функции или объекты реализующие определенный интерфейс и создающие экземпляры сервисов при первой необходимости. Конечно для этого необходимо немного расширить класс `ServiceLocator`:

{% highlight php %}
<?php
class ServiceLocator{
  ...

  public function get($serviceName){
    // Если сервис является анонимной функцией

    if($this->serviceLocator[$serviceName] instanceof Closure){
      return $this->serviceLocator[$serviceName]($this);
    }

    // Или фабрикой

    elseif($this->serviceLocator[$serviceName] instanceof FactoryInterface){
      return $this->serviceLocator[$serviceName]->build($this);
    }
    else{
      return $this->services[$serviceName];
    }
  }
}
{% endhighlight %}

Семантика классов-фабрик может быть описана следующим интерфейсом:

{% highlight php %}
<?php
interface FactoryInterface{
  public function build(ServiceLocatorInterface $serviceLocator);
}
{% endhighlight %}

После эти небольших правок можно инкапсулировать логику создания экземпляра доступа к базе данных в фабрику:

{% highlight php %}
<?php
class DataBaseFactory implements FactoryInterface{
  public function build(ServiceLocatorInterface $serviceLocator){
    $config = $serviceLocator->get('config')['database'];
    return new PDO(...);
  }
}

$serviceLocator->add('DataBase', new DataBaseFactory);

$db = $serviceLocator->get('DataBase'); // Вызов фабрики
{% endhighlight %}

Конечно в реальном проекте не следует регистрировать фабрики с помощью метода `add`, так как это исключает возможность регистрировать фабрики как сервисы в локаторе служб, а это недопустимо.

## Шаринг
Раз мы обсудили пример с регистрацией сервиса доступа к базе данных в локаторе, затроним вопрос шаринга этого сервиса. Для этого так же следует немного расширить класс `ServiceLocator`:

{% highlight php %}
<?php
class ServiceLocator{
  private $services = [];

  private $sharingMap = [];

  public function add($serviceName, $service, $sharing){
    $this->services[$serviceName] = $service;
    $this->sharingMap[$serviceName] = $sharing;
  }

  ...

  public function get($serviceName){
    // Предоставление сервиса из карты шаринга, если он в ней зарегистрирован

    if(isset($this->sharingMap[$serviceName]) && !is_bool($this->sharingMap[$serviceName])){
      return $this->sharingMap[$serviceName];
    }

    if($this->serviceLocator[$serviceName] instanceof Closure){
      $service = $this->serviceLocator[$serviceName]($this);
    }
    elseif($this->serviceLocator[$serviceName] instanceof FactoryInterface){
      $service = $this->serviceLocator[$serviceName]->build($this);
    }
    else{
      $service = $this->services[$serviceName];
    }

    if($this->sharingMap[$serviceName] === true){
      $this->sharingMap[$serviceName] = $service;
    }

    return $service;
  }
}
{% endhighlight %}

После этих изменений служба доступа к базе данных будет создаваться только один раз при первом запросе:

{% highlight php %}
<?php
$serviceLocator->add('DataBase', new DataBaseFactory, true);

$db = $serviceLocator->get('DataBase'); // Создание службы

$db2 = $serviceLocator->get('DataBase'); // Получение уже созданной ранее службы

var_dump($db == $db2); // true
{% endhighlight %}

# Советы по использованию паттерна

Что нужно делать:

* Применяйте данный патерн для приведения в порядок вашей системы, ее инфраструктуры и зависимостей. Храните в локаторе служб все повторно используемые сервисы
* Используйте несколько локаторов служб, если это позволит более явно подчеркнуть различные компоненты системы. К примеру, все контроллеры вы можете хранить в одном локаторе, а сервисы уровня инфраструктуры в другом
* Храните сервисы в локаторе под именами их классов или реализуемых ими интерфейсов, если это возможно

Что не нужно делать:

* Не храните в локаторе элементы бизнес-модели
* Не предоставляйте всем элементам бизнес-модели прямой доступ к локатору служб, передавайте им из него только то, что им действительно нужно для работы
