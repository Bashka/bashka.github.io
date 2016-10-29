---
title: "Виджет. Часть 4: элементы и иерархия"
description: "Виджет должен быть реализован в виде модуля с минимальными зависимостями, что позволило бы включать их друг в друга, организовывая более сложные структуры. Как это делается обсудим в данной статье..."
categories: js
tags: ["widget", "bem", "elements", "элементы", "иерархия", "gui"]
comments: true
---
После получения модели и рендеринга представления, виджету необходимо сформировать ссылки на элементы, из которых строится представление. Такие элементы можно разделить на две группы:

* Элементы - части представления, неотделимые от виджета

{% highlight html %}
<!-- Виджет search -->
<form class="search">
  <!-- Элемент input -->
  <input type="text" class="search__input"/>
  <!-- Элемент button -->
  <input type="button" class="search__button" value="Найти"/>
</form>
{% endhighlight %}

* Вложенные виджеты - другие виджеты, являющиеся частью данного

{% highlight html %}
<!-- Виджет search -->
<form class="search">
  <!-- Виджет input -->
  <input type="text" class="input"/>
  <!-- Элемент button -->
  <input type="button" class="search__button" value="Найти"/>
</form>
{% endhighlight %}

> Для идентификации элементов виджета я рекомендую использовать [БЭМ](https://ru.bem.info/methodology/) методологию, так как это стандартизирует именование виджетов и его элементов, а так же упрощает работу с ними.

# Формирование ссылок на элементы виджета

Часто виджету необходимо получить доступ к своим элементам (для добавления обработчиков событий, на пример). Сделать это можно двумя способами:

* Получать ссылку в месте ее использования

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета
}

SearchWidget.prototype.onButtonClick = function(){
  var $input = $(this.el).find('.search__input'); // Получение ссылки

  var search = $input.val(); // Использование ссылки
  ...
}
{% endhighlight %}

* Получить ссылку заранее

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета

  // Получение ссылок
  this.$input = $(this.el).find('.search__input');
}

SearchWidget.prototype.onButtonClick = function(){
  var search = this.$input.val(); // Использование ссылки
  ...
}
{% endhighlight %}

Какой подход выбрать зависит от ситуации. Первый вариант позволяет динамически изменять структуру виджета, без необходимости переформирования списка. Второй вариант позволяет декларативно формировать и использовать элементы, на пример так:

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета

  // Декларация зависимостей
  this.parts = {
    '$input': '.search__input'
  };
}

// Получение элементов виджета по декларации
SearchWidget.prototype.getParts = function(){
  for(var prop in this.parts){
    this[prop] = $(this.el).find(this.parts[prop]);
  }
};
{% endhighlight %}

В случае необходимости, можно использовать оба подхода, заменяя ссылки динамически:

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета

  // Декларация зависимостей
  this.parts = {
    '$input': '.search__input'
  };
}

// Метод заменяющий поле ввода виджета
SearchWidget.prototype.otherMethod = function(){
  this.$input.remove();
  this.$input = $('<input type="text" class="search__input"/>');
  $(this.el).append(this.$input);
};
{% endhighlight %}

> Корневой элемент виджета можно обернуть в JQuery, что упростит работу с ним в будущем:
{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета
  this.el = el;
  this.$el = $(el);
}
{% endhighlight %}

# Работа с вложенными виджетами

Если элементом виджета является другой виджет, следует рассматривать две основные задачи:

1. Как рендерить вложенный виджет при рендеринге основного?
2. Как взаимодействовать с вложенным виджетом?

Для сохранения независимости и переносимости виджетов, рекомендуется взаимодействовать с ними через открытые интерфейсы и события.

Так, для рендеринга вложенных виджетов лучше использовать их метод `render`:

{% highlight js %}
SearchWidget.prototype.render = function(){
  var $box = $('<form class="search">');
  
  // Рендеринг и вставка вложенного виджета
  var inputWidget = new InputWidget;
  $box.append(inputWidget.render());

  $box.append('<input type="button" class="search__button" value="Найти"/>');

  return $box;
};
{% endhighlight %}

А вот пример взаимодействия с вложенным виджетом:

{% highlight js %}
SearchWidget.prototype.onButtonClick = function(){
  // Получение ссылки на виджет
  var inputWidget = $(this.el).find('.input').data('widget');
  var search = inputWidget.getValue();

  // Использование данных, полученных из виджета
};
{% endhighlight %}
