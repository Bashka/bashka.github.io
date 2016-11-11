---
title: "Валидация бизнес-модели"
description: "Как следует проверять данные пользователя и контролировать состояние бизнес-модели, чтобы не утонуть в логике фильтров и валидаторов..."
categories: backend
tags: ["validation", "валидация", "бизне-ограничения", "объект", "состояние", "верификация", "verification"]
comments: true
---
Наибольшую опасности для приложения представляют входные данные. Это ни для кого не секрет и часто большая часть усилий направленно именно на их контроль и преобразование. Такой подход называется "Инвариантом" и предполагает наличие одного допустимого состояния для каждого объекта системы. Рассмотрим пример:

{% highlight php %}
<?php
class User{
  private $login;
  private $password;

  public static function registerUser($login, $password){
    $user = new self;
    $user->setLogin($login);
    $user->setPassword(md5($password));

    return $user;
  }

  public function setLogin($login){
    $this->login = $login;
  }

  public function getLogin(){
    return $this->login;
  }

  public function setPassword($password){
    $this->password = $password;
  }

  public function getPassword(){
    return $this->password;
  }
}
{% endhighlight %}

Этот простой класс бизнес-модели описывает пользователей системы, которые могут быть зарегистрированы с помощью метода `registerUser`. Зададимся вопросом - кто и где должен проверять входные данные пользователя, такие как его `$login` и `$password`, а так же какие ограничения накладываются на эти данные?

Предположим что бизнес требования для данных о пользователе системы у нас следующие:

* Логин пользователя должен содержать только латинские буквы в любом регистре, цифры и символ подчеркивания, а так же иметь длину от 1 до 10 байт
* Пароль пользователя должен содержать только латинские буквы в любом регистре, цифры, символ подчеркивания и тире, а так же иметь длину от 5 до 20 байт

Одним из решений для валидации входных данных является включение проверяющей логики в бизнес-модель:

{% highlight php %}
<php
class User{
  ...

  public static function registerUser($login, $password){
    if(preg_match('/^[A-Za-z0-9_]{1,10}$/', $login) === false){
      return false;
    }
    if(preg_match('/^[A-Za-z0-9_-]{5,20}$/', $password) === false){
      return false;
    }

    $user = new self;
    ...

    return $user;
  }
}
{% endhighlight %}

Другим решением является использование стороннего (внешнего) валидатора вида:

{% highlight php %}
<?php
class UserValidator{
  public function isValid($login, $password){
    if(preg_match('/^[A-Za-z0-9_]{1,10}$/', $login) === false){
      return false;
    }
    if(preg_match('/^[A-Za-z0-9_-]{5,20}$/', $password) === false){
      return false;
    }

    return true;
  }
}
{% endhighlight %}

Использоваться такой класс может следующим образом:

{% highlight php %}
<?php
$login = $_GET['login'];
$password = $_GET['password'];

$validator = new UserValidator;
if(!$validator->isValid($login, $password)){
  throw new RuntimeException('Невалидные данные пользователя');
}

...
{% endhighlight %}

В большинстве случаев можно ограничиться первым вариантом решения, для предотвращения использования объектов, находящихся в недопустимом состоянии. Я предпочитаю смешивать эти решения, реализуя валидацию во внешнем классе и агреригуя его в валидируемом объекте:

{% highlight php %}
<?php
class User{
  ...

  public function isValid($login, $password){
    return $this->validator->isValid($login, $password);
  }
}
{% endhighlight %}

Это позволяет связать проверяемую и проверяющую логику, но при этом избежать "засорения" объектов бизнес-модели, а так же сделать решение более гибким, за счет возможности быстрой замены валидатора.

# Аккумуляция ошибок
Предложенные решения достаточно удобны в использовании, но они ничего не сообщают о причинах невалидности модели. Конечно мы могли бы использовать исключения с описательными сообщениями, вместо возврата булевого значения, но это ведет к двум проблемам:

1. Для верификации объекта придется использовать конструкцию `try/catch`, что не очень удобно
2. При невалидности модели мы получим информацию только о первом невалидном свойстве, так как выброс исключения остановит дальнейшую валидацию

Более подходящим решением является аккумуляция ошибок. Реализуется оно достаточно просто и одинаково как для внутреннего, так и для внешнего валидатора:

{% highlight php %}
<?php
class UserValidator{
  private $errors = [];

  public function validate($login, $password){
    $this->errors = [];

    if(preg_match('/^[A-Za-z0-9_]{1,10}$/', $login) === false){
      $this->errors[] = new RuntimeException('Invalid login');
    }
    if(preg_match('/^[A-Za-z0-9_-]{5,20}$/', $password) === false){
      $this->errors[] = new RuntimeException('Invalid password');
    }

    return $this;
  }

  public function getErrors(){
    return $this->errors;
  }
}
{% endhighlight %}

Пример использования:
{% highlight php %}
<?php
$login = $_GET['login'];
$password = $_GET['password'];

$validator = new UserValidator;
if(!count($errors = $validator->validate($login, $password)->getErrors())){
  // Получение сообщений валидатора

  $errorsMessages = array_map(function($error){
    return $error->getMessage();
  }, $errors);

  throw new RuntimeException('Невалидные данные пользователя: ' . implode('; ', $errorsMessages));
}

...
{% endhighlight %}

Как видно из примера, метод `isValid` был заменен на `validate`. Это связано с тем, что валидация с аккумуляцией ошибок не просто проверяет валидность данных бизнес-модели, но и определяет, что именно не соответствует бизнес-ограничениям. Валидны ли данные можно определить с помощью подсчета записей, возвращаемых методом `getErrors` валидатора.

# Контекстуальная валидация
Немного усложним нашу бизнес-модель, добавив следующее свойство классу `User`:

{% highlight php %}
<?php
class User{
  ...
  private $email;

  ...

  public function setEmail($email){
    $this->email = $email;
  }

  public function getEmail(){
    return $this->email;
  }
}
{% endhighlight %}

Электронный адрес пользователя является необязательным свойством, но он должен быть указан для вызова метода `Notificator::notifyFromEmail` (email-рассылка).

Такая валидация называется "Контекстуальной" или "Зависимой от потребителя". Другими словами логика валидации объекта может изменяться, в зависимости от того, как предполагается им воспользоваться. Часто используется минимум один контекстуальный валидатор, проверяющий состояние объекта перед записью его в базу данных. Иногда он дополняется вторым контекстуальным валидатором, проверяющим состояние объекта перед выводом его пользователю (для защиты от XSS, на пример).

Реализуется такая валидация довольно просто. Достаточно всего лишь предоставить по отдельной логике валидации для каждого из возможных контекстов. При использовании внешнего валидатора это может быть реализовано с помощью нескольких методов, по одному для каждой проверки, либо с использованием множества классов. Мы рассмотрим пример контекстуальной валидации с внутренней логикой:

{% highlight php %}
<?php
class User{
  ...
  private $errors;

  // Проверяет допустимость состояния объекта для сохранения в базу данных

  public function validatePersist(){
    $this->errors = [];
    if(preg_match('/^[A-Za-z0-9_]{1,10}$/', $this->getLogin()) === false){
      $this->errors[] = new RuntimeException('Invalid login');
    }
    if(empty($this->getPassword())){
      $this->errors[] = new RuntimeException('Invalid password');
    }

    return $this;
  }

  // Проверяет допустимость состояния объекта для отправки email-уведомления

  public function validateNotification(){
    $this->validatePersist();
    if(preg_match('/^[A-Za-z0-9_-]+@[A-Za-z0-9.]+$/', $this->getEmail()) === false){
      $this->errors[] = new RuntimeException('Invalid email');
    }

    return $this;
  }
}
{% endhighlight %}

Как видно, метод `validateNotification` предварительно вызывает метод `validatePersist`. Это сделано для демонстрации возможности смешивания логики валидации.

Применяется такое решение следующим образом:

{% highlight php %}
<?php
// Регистрация нового пользователя

$user = User::registerUser($_GET['login'], $_GET['password']);

if(!count($errors = $user->validatePersist()->getErrors())){
  ...

  throw new RuntimeException('Регистрация невозможна. Неверные данные пользователя: ' . $errorsMessages);
}

$userTable->save($user);

// Уведомление по электронной почте

$user->setEmail($_GET['email']);

if(!count($errors = $user->validateNotification()->getErrors())){
  ...

  throw new RuntimeException('Неверные данные пользователя для уведомления: ' . $errorsMessages);
}

$emailSender->send($user, ...);
{% endhighlight %}
