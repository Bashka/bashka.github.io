---
title: "Архитектура: Модуль таблицы"
description: "Пример переходной архитектуры, основанной на объектном интерфейсе к реляционной базе данных..."
categories: backend
tags: ["architecture", "php", "архитектура", "модуль таблицы", "table module"]
comments: true
---
> Данная статья является продолжением поста "[Программная архитектура](/posts/software-architecture)". Ознакомьтесь с ним перед тем, как приступить к изучению.

Переходным звеном между "[Сценарием транзакции](/posts/architecture-transaction)" и "Моделью домена" является "Модуль таблицы" (автор [Мартин Фаулер](https://www.martinfowler.com/eaaCatalog/tableModule.html)). Это решение предлагает использовать по классу для каждой таблицы в реляционной базе данных, который включал бы все необходимые для работы методы. Такие классы можно было бы обернуть в легкие сценарии (на подобе сценариев транзакций), отвечающие за взаимодействие с ними, либо использовать напрямую в качестве контроллеров с инкапсулированной логикой доступа к данным.

Рассмотрим пример:

{% highlight php %}
<?php
class BlogTable{
  // Добавление статьи в блог

  public function addArticle($author, $title, $content, array $tags){
    ...
  }

  // Удаление статьи из блога

  public function removeArticle($arthicleId){
    ...
  }

  // Получение списка статей

  public function getArticles($limit = 10, $offer = 0){
    ...
  }

  // Получение конкретной статьи

  public function getArticle($arthicleId){
    ...
  }
}
{% endhighlight %}

> Чаще всего классы, представляющие таблицы базы данных, предоставляют сразу коллекции сущностей, хранящихся в этих таблицах, а не единичные экземпляры.

Поверх таких классов, может быть построен простой REST API, который перенесет основную логику генерации пользовательского интерфейса на уровень клиента. Более того, инкапсуляция логики работы с сущностями системы уменьшает объем повторяемого кода и делает архитектуру более гибкой и переносимой.

Пример слоя контроллеров, использующих "Модуль таблиц":

{% highlight php %}
<?php
class BlogController{
  private $userTable;
  private $blogTable;

  public function __construct(UserTable $userTable, BlogTable $blogTable){
    $this->userTable = $userTable;
    $this->blogTable = $blogTable;
  }

  // Страницы

  public function indexPage(){
    return new TemplateEngine('blog/index.html', [
      'articles' => $this->blogTable->getArticles(),
    ]);
  }

  // Действия

  public function addAction(){
    $currentUser = $this->userTable->getCurrentUser();
    $title = trim($_POST['title']);
    $content = trim($_POST['content']);

    $this->blogTable->addArticle($currentUser, $title, $content, []);

    return header('Location: /blog');
  }

  public function removeAction(){
    $currentUser = $this->userTable->getCurrentUser();
    $targetArticle = $this->blogTable->getArticle((int) $_POST['id']);

    // Удалить статью может только ее автор.

    // Данное ограничение может быть инкапсулированно в класс BlogTable, либо вынесено в контроллер, как в данном примере

    if($targetArticle->author != $currentUser){
      return header('Location: /error');
    }

    $this->blogTable->removeArticle($targetArticle->id);

    return header('Location: /blog');
  }
}
{% endhighlight %}

Данная архитектура все еще довольно проста в реализации, но все же не идеальна. Вот некоторые проблемы, которые не могут быть решены с ее помощью:

* Сильная зависимость от инфраструктуры, в виде реляционной базы данных
* Смешивание с инфраструктурой или полное отсутствие слоя модели с бизнес-ограничениями и бизнес-логикой, присущей ей
* Более сложная структура, по сравнению со "Сценарием транзакции", и недостаточно гибкая и переносимая, нежели "Модель домена"
