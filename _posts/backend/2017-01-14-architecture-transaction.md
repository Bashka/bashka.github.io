---
title: "Архитектура: Сценарий транзакции"
description: "Рассмотрим идеальную программную архитектуру для небольших проектов..."
categories: backend
tags: ["architecture", "php", "архитектура", "сценарий транзакции", "transaction script"]
comments: true
---
> Данная статья является продолжением поста "[Программная архитектура](/posts/software-architecture)". Ознакомьтесь с ним перед тем, как приступить к изучению.

Простейшим архитектурным решением, является "Сценарий транзакции" (автор [Мартин Фаулер](https://martinfowler.com/eaaCatalog/transactionScript.html)). Это решение предлагает использовать несколько скриптовых сценариев, обрабатывающих запросы пользователя к системе и возвращающих ему результат этой обработки. Важным условием здесь является то, что сценарии отвечают за весь производственный цикл ответа, а не делегируют обработку другим частям приложения.

Рассмотрим несколько примеров:

{% highlight php %}
<?php
// Процедурный стиль

// controller/guestbook.php

...

switch($_REQUEST['action']){
  // Просмотр гостевой книги

  case 'index':
    $template = new Template('view/guestbook/index.php', [
      'posts' => Db::getInstance()->select('SELECT * FROM `guestbook` LIMIT 10 ORDER BY `added` DESC')
    ]);
    echo $template->render();
    break;

  // Добавление сообщения в гостевую книгу

  case 'add':
    $text = InputStringFilter($_POST['text']);
    Db::getInstance()->insert('guestbook', [
      'text' => $text,
      'added' => time(),
    ]);
    return header('Location: /guestbook.php?action=index');
    break;
}
{% endhighlight %}

{% highlight php %}
<?php
// Объектный стиль

// controller/guestbook.php

class GuestbookController{
  // Просмотр гостевой книги

  public function indexAction(){
    $template = new Template('view/guestbook/index.php', [
      'posts' => Db::getInstance()->select('SELECT * FROM `guestbook` LIMIT 10 ORDER BY `added` DESC')
    ]);

    return $template->render();
  }

  // Добавление сообщения в гостевую книгу

  public function addAction(){
    $text = InputStringFilter($_POST['text']);
    Db::getInstance()->insert('guestbook', [
      'text' => $text,
      'added' => time(),
    ]);

    return header('Location: /guestbook.php?action=index');
  }
}
{% endhighlight %}

Подобный подход чаще всего используется в небольших проектах и отличается следующими возможностями:

* Быстрая разработка, за счет исключения из архитектуры дополнительных компонентов, таких как модели, сервисы и т.д.
* Простота изучения (только до определенного момента), основанная на аккумуляции всей логики приложения в скрипты транзакций

К сожалению применение этой архитектуры в средних и крупных проектах влечет следующие проблемы:

* Дублирование кода, вызванное отсутствием гибкости архитектуры и невозможностью частичного вынесения логики из сценариев для повторного использования
* Раздувание сценариев (как следствие предыдущего пункта) и усложнение процесса их изучения
