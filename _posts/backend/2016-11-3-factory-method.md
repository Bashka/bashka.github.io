---
title: "Паттерн: фабричный метод"
description: "Логика создания объекта не столь тривиальна, как может показаться на первый взгляд. Хорошая программная архитектура накладывает определенные ограничения, которые стоит учитывать и о которых мы поговорим в этой статье..."
categories: backend
tags: ["pattern", "factory method", "фабричный метод", "паттерн", "инстанциация класса", "создания объекта"]
comments: true
---
В приложении с объектно-ориентированной архитектурой часто встает вопрос: кто должен отвечать за создание того или иного объекта? Многие современные решения используют _Локатор служб_ для инстанциации и хранения объектов инфраструктуры и даже некоторых элементов бизнес-модели, но этого часто не достаточно. Рассмотрим простой пример:

{% highlight php %}
<?php
class Message{
  // @var string Заголовок сообщения.

  private $title;

  //  @var string Имя шаблона сообщения.

  private $template;

  // Getters и Setters

  ...
}

class MailMessage extends Message{
  // @var string Отправитель сообщения.

  private $from;

  // @var string Получатель сообщения.

  private $to;

  // Отправляет сообщение получателю.
  
  public function send(){
    // Реализация

    ... 
  }
}
{% endhighlight %}

Предполагается, что использоваться эти классы будут в методах `Order::pay` (оплата заказа), `Auth::registerNewUser` (регистрация нового пользователя) и `Article::create` (создание новой статьи).

Какая часть системы должна отвечать за создание экземпляров класса `MailMessage` и следует ли вызывать метод `send` сразу после инстанциации?

# Фабричный метод
Паттерн _Фабричный метод_ достаточно прост в реализации и использовании. Он имеет следующую структуру:

![Структура паттерна Фабричный метод](http://www.plantuml.com/plantuml/png/Iyv9B2vMSCxFIovABKcjvghbuag621Mb9fRa5rLpAIYa9IO3MPM-gIKP-IaQcWfMSFKWvSuGXGfwUdPmSG00)

Его задача заключается в том, чтобы организовать специальный метод, который будет отвечать за создание объектов данного или другого класса и возвращать их.

Чем же фабричный метод лучше обычного конструктора? Тем что он позволяет сформировать конфигурацию (параметры) конструктора перед его вызовом. Так, для создания соединения с БД необходимо прежде получить конфигурацию для этого соединения, и фабричный метод позволит собрать всю логику инстанциации в одном месте:

{% highlight php %}
<?php
class Db{
  // Статичный фабричный метод

  public static function getInstance(){
    $config = include('config/database.php');
    $dsn = sprintf(
      '%s:host=%s;dbname=%s',
      isset($config['driver'])? $config['driver'] : 'mysql',
      isset($config['host'])? $config['host'] : 'localhost',
      $config['dbname']
    );

    return new PDO($dsn, $config['login'], $config['password']);
  }
}
{% endhighlight %}

Как видно из примера, фабричный метод может быть статичным. Часто это используется для инстанциации вызываемого класса. Не статичные методы больше применяются для создания экземпляров других классов.

Вернемся к нашей задаче из вводной. Где разместить фабричный метод, который будет создавать экземпляры класса `MailMessage`? Учитывая что для рассылки писем необходим адресат (свойство `$to`), логичным решением будет разместить его в классе `User` с именем `notifyFromEmail`. Метод будет создавать сообщение для вызываемого пользователя:

{% highlight php %}
<?php
class User{
  // Реализация

  ...

  public function notifyFromEmail($title, $template, $from = null){
    $message = new MailMessage;
    $message->setTitle($title);
    $message->setTemplate($template);
    // Заполнение адресата email текущего пользователя

    $message->setTo($this->getEmail());
    if(!is_null($from)){
      $message->setFrom($from);
    }

    return $message;
  }
}

// Использование

$message = $currentUser->notifyFromEmail('Заголовок', 'email/notifi.tpl');
$message->send();
{% endhighlight %}

Теперь имея ссылку на пользователя мы можем легко оповестить его с помощью письма об интересных событиях:

{% highlight php %}
<?php
class Auth{
  public static function registerNewUser($login, $password){
    $user = new User;
    $user->setEmail($login);
    $user->setPassword($password);

    // Оповещение

    $user->notifyFromEmail('Вы зарегистрированы', 'email/new_account.tpl', 'system@site.com')
      ->send();

    return $user;
  }
}
{% endhighlight %}

# Работа с объектом в фабричном методе
Из предыдущих примеров у вас может возникнуть желание вызывать метод `send` прямо в фабричном методе `notifyFromEmail`. Не делайте так! Запомните - фабричный метод должен только создавать и конфигурировать объекты, а не взаимодействовать с ними. За сохранение созданного объекта в базе данных или передачу созданного сообщения по электронной почте должен отвечать объект, вызвавший фабричный метод или инфраструктура системы. Запомнили? Хорошо. Теперь попробуем понять почему так.

Дело в том, что взаимодействие с объектом внутри фабричного метода делает решение менее гибким. Рассмотрим пример:

{% highlight php %}
<?php
class Auth{
  public static function registerNewUser($login, $password){
    $user = new User;
    $user->setEmail($login);
    $user->setPassword($password);

    // Оповещение

    $user->notifyFromEmail('Вы зарегистрированы', 'email/new_account.tpl', 'system@site.com')
      ->send();

    return $user;
  }
}
{% endhighlight %}

После вызова `registerNewUser` предполагается, что регистрируемый пользователь будет оповещен с помощью email-рассылки и появляется желание добавить эту логику прямо в метод. Но что делать, если мы не должны рассылать уведомления пользователям, зарегистрированным в системе посредством выгрузки из CSV файла? В этом случае нам придется повторить логику фабричного метода `registerNewUser` без оповещения. В данном случае проще будет отказаться от взаимодействия с создаваемым внутри метода объектом:

{% highlight php %}
<?php
class Auth{
  public static function registerNewUser($login, $password){
    $user = new User;
    $user->setEmail($login);
    $user->setPassword($password);

    return $user;
  }
}

class UserController{
  // Обычная регистрация пользователя с оповещением

  public function registerAction($email, $password){
    $user = Auth::registerNewUser($email, $password);

    // Оповещение

    $user->notifyFromEmail('Вы зарегистрированы', 'email/new_account.tpl', 'system@site.com')
      ->send();

    $db->save($user);
  }

  // Выгрузка пользователей из CSV файла без оповещения

  public function exportUsersAction($csvFile){
    ...

    while(...){
      $user = Auth::registerNewUser($data['email'], $data['password']);
      $db->save($user);
    }
  }
}
{% endhighlight %}

Как видите, сохранение зарегистрированного пользователя в базе данных так же выполняется не внутри фабричного метода `registerNewUser`, а в соответствующих методах контроллера.
