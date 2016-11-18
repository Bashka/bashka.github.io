---
title: "ZendFramework: Config"
description: "Рассмотрим пакет для конфигурации приложения, а так же обсудим несколько полезных решений для организации данных..."
categories: backend
tags: ["zend framework", "zf", "config", "конфигурация"]
comments: true
---
Пакет [Zend-Config](https://github.com/zendframework/zend-config) представляет простой механизм для работы с конфигурацией приложения. Он включает класс `Config`, представляющий данные конфигурации и предоставляющий объектный интерфейс доступа к ним. Для начала рассмотрим несколько примеров:

{% highlight php %}
<?php
use Zend\Config\Config;

$config = new Config([
  'database' => [
    'dbname' => 'my_db',
    'user' => 'root',
    'password' => 123,
  ],
]);
echo $config->database->user; // "root"
{% endhighlight %}

При инстанциации этого класса ему необходимо передать массив конфигурации в качестве первого параметра. Конструктор класса так же принимает второй, булевый параметр, который определяет, является ли конфигурация неизменяемой (read only) или нет:

{% highlight php %}
<?php
use Zend\Config\Config;

$config = new Config([], true); // по умолчанию - false

$config->database = [];
$config->database->user = 'root';
$config->setReadOnly(); // Делаем конфигурацию неизменяемой

$config->database->password = 123; // Error
{% endhighlight %}

Получить доступ к данным конфигурации можно так же с помощью метода `get`, который принимает помимо имени целевой опции, так же значение по умолчанию, которое будет возвращено в случае, если опция отсутствует:

{% highlight php %}
<?php
use Zend\Config\Config;

$config = new Config([
  'database' => [
    'dbname' => 'my_db',
    'user' => 'root',
    'password' => 123,
  ],
]);

echo $config->database->get('host', 'localhost'); // "localhost" - так как данной опции конфигурации не задано
{% endhighlight %}

В том случае, если необходимо слить две конфигурации в одну, можно воспользоваться методом `merge`:

{% highlight php %}
<?php
use Zend\Config\Config;

$configA = new Config(['x' => 1, 'y' => 2]);
$configB = new Config(['x' => 0, 'z' => 3]);
$configA->merge($configB);

echo $configA->x; // 0

echo $configA->y; // 2

echo $configA->z; // 3
{% endhighlight %}

# Чтение и запись конфигураций
Для чтения или записи конфигураций, используются классы из пространств имен `Zend\Config\Reader` и `Zend\Config\Writer` соответственно. Семантика этих классов определяется интерфейсами `ReaderInterface` и `WriterInterface`, а именно следующими методами:

* `ReaderInterface`
  * `fromString` - чтение данных конфигурации из строки
  * `fromFile` - чтение данных конфигурации из файла
* `WriterInterface`
  * `toString` - запись конфигурации в строку
  * `toFile` - запись конфигурации в файл

Для начала рассмотрим простой пример реализации собственных классов для чтения конфигурации из PHP скрипта, содержащего массив конфигурации, а так же записи в этот файл:

{% highlight php %}
<?php
use Zend\Config\Config;
use Zend\Config\Reader\ReaderInterface;
use Zend\Config\Writer\WriterInterface;

class PhpArrayReader implements ReaderInterface{
  public function fromString($string){
    eval('$result = ' . $string . ';'); // Не безопасно! Не делайте так, код приведен в качестве примера!

    return $result;
  }

  public function fromFile($filename){
    if(!file_exists($filename)){
      return [];
    }

    return include($filename);
  }
}

class PhpArrayWriter implements WriterInterface{
  public function toString($config){
    if($config instanceof Config){
      $config = $config->toArray();
    }

    return var_export($config, true);
  }

  public function toFile($filename, $config, $exclusiveLock = true){
    $flags = 0;
    if($exclusiveLock){
      $flags |= LOCK_EX;
    } 

    file_put_contents($filename, $this->toString($config), $flags);
  }
}


$reader = new PhpArrayReader;
$config = new Config($reader->fromFile('conf.php'), true);

$writer = new PhpArrayWriter;
$config->database->password = 321;
$writer->toFile('conf.php', $config);
{% endhighlight %}

> Следует помнить, что все методы интерфейсов могут принимать как массив конфигурации, так и экземпляр класса `Config`, что требует правильной обработки. Для преобразования конфигурации в массив можно использовать метод `toArray`, как показано в примере выше.

Пакет Zend-Config включает множество готовых кассов для чтения и записи конфигураций, которые вы можете использовать вместо собственной реализации:

* `ReaderInterface`
  * `Ini` - чтение из формата INI
  * `Json` - чтение из формата JSON
  * `Xml` - чтение из формата XML
  * `Yaml` - чтение из формата YAML
  * `JavaProperties` - чтение из формата JavaProperties
* `WriterInterface`
  * `PhpArray` - запись в формат массива PHP
  * `Ini` - запись в формат INI
  * `Json` - запись в формат JSON
  * `Xml` - запись в формат XML
  * `Yaml` - запись в формат YAML

С использованием перечисленных классов наш код из предыдущего примера будет выглядить так:

{% highlight php %}
<?php
use Zend\Config\Config;
use Zend\Config\Writer\PhpArray;

$config = new Config(include('conf.php'), true);

$writer = new PhpArray;
$config->database->password = 321;
$writer->toFile('conf.php', $config);
{% endhighlight %}

# Процессоры
Иногда конфигурация приложения нуждается в дополнительной обработке перед использованием, к примеру, для замены всех вхождений констант на их значения. Для реализации этой логики используются классы пространства имен `Zend\Config\Processor`. Семантику этих классов описывает интерфейс `ProcessorInterface`, включающий два метода:

* `processValue` - обработка и преобразование значения конфигурации
* `process` - обработка конфигурации в целом

Попробуем реализовать собственный процессор с использованием этого интерфейса:

{% highlight php %}
<?php
use Zend\Config\Processor\ProcessorInterface;
use Zend\Config\Config;
use Zend\Config\Exception\InvalidArgumentException;

class UpperProcessor implementsuse ProcessorInterface{
  public function processValue($value){
    return strtoupper($value);
  }

  public function process(Config $value){
    if($value->isReadOnly()){
      throw new InvalidArgumentException('Cannot process config because it is read-only');
    }

    foreach($value as $key => $val){
      if($val instanceof Config){
        $value->$key = $this->process($val);
      }
      else{
        $value->$key = $this->processValue($val);
      }
    }
  }
}

$config = new Config([
  'foo' => 'bar',
], true);
$processor = new UpperProcessor;
$processor->process($config);
echo $config->foo; // "BAR"
{% endhighlight %}

В пакет Zend-Config входят следующие процессоры:

* `Token` - выполняет замену вхождений указанных токенов
* `Constant` - частный случай `Token`, выполняющий замену вхождений констант
* `Filter` - выполняет обработку значений конфигурации с помощью `Zend\Filter`
* `Translator` - выполняет локализацию значений конфигурации с помощью `Zend\I18n`
* `Queue` - объединяет несколько процессоров в один

{% highlight php %}
<?php
use Zend\Config\Config;
use Zend\Processor\Token;
use Zend\Processor\Constant;
use Zend\Processor\Filter;
Zend\Filter\StringToUpper;
use Zend\Processor\Queue;

$config = new Config([
  'token' => 'token',
  'constant' => 'CONST',
], true);

$queue = new Queue;

$token = new Token;
$token->addToken('token', 'foo');
$queue->insert($token);

define('CONST', 'bar');
$constant = new Constant;
$queue->insert($constant);

$filter = new StringToUpper;
$queue->insert($constant);

$queue->process($config);

echo $config->token; // "FOO"
echo $config->constant; // "BAR"
{% endhighlight %}

# Фабрика конфигураций
Класс `Factory` предоставляет удобный интерфейс доступа к конфигурациям с помощью следущих статичных методов:

* `fromFile` - чтение конфигурации из файла
* `fromFiles` - чтение конфигурации из нескольких файлов
* `toFile` - запись конфигурации в файл

Удобство данного класса в том, что он использует расширение файла для выбора подходящего `Reader` и `Writer` в каждом конкретном случае:

{% highlight php %}
<?php
use Zend\Config\Factory;

$config = Factory::fromFiles(['config.php', 'config.json', 'config.xml'], true);
Factory::toFile('config.yaml', $config);
{% endhighlight %}

Этот пример показывает, как можно с помощью класса `Factory`  объединить конфигурации разных форматов (PHP, JSON, XML) в один файл (YAML).

> Методы `fromFiles` и `fromFile` в качестве второго параметра принимают булевое значение, определяющее в каком виде должна быть возвращена конфигурация: true - в виде экземпляра класса `Config`, false - в виде массива.

# Организация конфигураций приложения
В приложении полезно разделять конфигурацию на несколько файлов, объединяя ее во время использования. Группируются конфигурации по следующим правилам:

* Глобальные и модульные конфигурации - общие настройки системы, относящиеся к инфраструктуре (на пример, конфигурация доступа к базе данных), должны располагаться в файлах общих конфигураций, а прикладные конфигурации в файлах модулей
* Конфигурации разработки и развертывания - часто конфигурации для разработки и развертывания требуют различных данных, в этом случае полезно использовать два объединяемых файла конфигурации `develop` и `production`

Рассмотрим пример организации конфигураций в модульном приложении:

{% highlight bash %}
application/
  config/ # глобальные конфигурации
    config.php # общие конфигурации
    production.php # конфигурации только для production
    develop.php # конфигурации для разработчика
  module/
    Album/
      config/ # модульные конфигурации
        config.php # общие конфигурации
        production.php # конфигурации только для production
        develop.php # конфигурации для разработчика
{% endhighlight %}

При старте приложения необходимо объединять глобальные и модульные конфигурации, исключая `production` и `develop` конфигурацию в зависимости от окружения:

{% highlight php %}
<?php
use Zend\Config\Factory;

// Для Production окружения

$config = Factory::fromFiles([
  'application/config/config.php',
  'application/config/production.php',
  'application/module/Album/config/config.php',
  'application/module/Album/config/production.php',
], true);

// Для Develop окружения

$config = Factory::fromFiles([
  'application/config/config.php',
  'application/config/develop.php',
  'application/module/Album/config/config.php',
  'application/module/Album/config/develop.php',
], true);
{% endhighlight %}

Для автоматизации процесса подключения конфигураций можно так же использовать переменные окружения:

{% highlight php %}
<?php
use Zend\Config\Factory;

$config = Factory::fromFiles([
  'application/config/config.php',
  'application/config/' . $_ENV['env'] . '.php',
  'application/module/Album/config/config.php',
  'application/module/Album/config/' . $_ENV['env'] . '.php',
], true);
{% endhighlight %}

В таком случае при запуске сервера с переменной окружения `env`, приложением будут загружены соответствующие ей конфигурации.
