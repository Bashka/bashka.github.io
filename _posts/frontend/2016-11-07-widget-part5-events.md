---
title: "Виджет. Часть 5: взаимодействие с пользователем"
description: "В данной статье мы обсудим ту часть виджета, которую можно считать его контроллером, а именно реакцию на действия пользователя и генерацию событий самим виджетом..."
categories: frontend
tags: ["js", "widget", "event", "controller", "событие", "виджет", "контроллер", "gui"]
comments: true
---
# Обработка событий
При взаимодействии пользователя с виджетом или его компонентами, необходимо перехватить событие и вызвать соответствующий ему метод обработки. Для установки слушателей используется два основных подхода:

1. Установка слушателя непосредственно элементу, генерирующему событие

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета
}

SearchWidget.prototype.render = function(){
  // Рендеринг

  this.bind(); // Установка слушателей
};

SearchWidget.prototype.bind = function(){
  this.$button.on('click', this.onSearch.bind(this));
};

SearchWidget.prototype.onSearch = function(){
  // Обработка события
};
{% endhighlight %}

2. Установка слушателя корневому элементу виджета с проверкой источника события

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета
}

SearchWidget.prototype.render = function(){
  // Рендеринг

  this.bind(); // Установка слушателей
};

SearchWidget.prototype.bind = function(){
  this.$el.on('click', this.$button, this.onSearch.bind(this));
  // или
  this.$el.on('click', 'search__button', this.onSearch.bind(this));
};

SearchWidget.prototype.onSearch = function(){
  // Обработка события
};
{% endhighlight %}

Решения не являются взаимоисключающими и могут использовать совместно, но следует помнить о необходимости отключения обработчиков при уничтожении виджета, для недопущения дублирования:

{% highlight js %}
var search = new SearchWidget(undefined, $('.search'));
search.render();  // Первая установка обработчиков
search.destroy(); // Обработчики не отключены
search.render();  // Дублирующая установка обработчиков
{% endhighlight %}

С использованием первого решения сделать это можно при декларативной реализации, путем отключения обработчиков на основе данных декларации:

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета

  this.events = {
    'click $button': this.onSearch.bind(this)
  };
}

SearchWidget.prototype.render = function(){
  // Рендеринг

  this.bind(); // Установка слушателей
};

SearchWidget.prototype.bind = function(){
  this._listeners = [];

  for(var i in this.events){
    var listener = this.events[i];

    i = i.split(' ');
    var event = i.shift(),
      $element = this[i.join(' ')];

    $element.on(event, listener);
    // Учет всех элементов виджета с обработчиками событий
    this._listeners.push({
      event: event,
      $element: $element,
      listener: listener
    });
  }
};

SearchWidget.prototype.destroy = function(){
  // Поочередное отключение обработчиков событий при уничтожении виджета
  for(var i in this._listeners){
    var config = this._listeners[i];
    config.$element.off(config.event, config.listener);
  }
};
{% endhighlight %}

При использовании второго решения, достаточно отключить обработчики, добавленные корневому элементу виджета:

{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета

  this.events = {
    'click .search__button': this.onSearch.bind(this)
  };
}

SearchWidget.prototype.render = function(){
  // Рендеринг

  this.bind(); // Установка слушателей
};

SearchWidget.prototype.bind = function(){
  this._listeners = [];

  for(var i in this.events){
    var listener = this.events[i];

    i = i.split(' ');
    var event = i.shift(),
      selector = this[i.join(' ')];

    this.$el.on(event, selector, listener);
    // Учет всех обработчиков событий корневого элемента
    this._listeners.push({
      event: event,
      selector: selector,
      listener: listener
    });
  }
};

SearchWidget.prototype.destroy = function(){
  // Поочередное отключение обработчиков событий при уничтожении виджета
  for(var i in this._listeners){
    var config = this._listeners[i];
    this.$el.off(config.event, config.selector, config.listener);
  }
};
{% endhighlight %}

# Генерация событий
Помимо обработки событий пользователя, виджет может генерировать собственные события, облегчающие процесс коммуникации GUI. Генерируемые виджетом события можно разделить на две группы:

1. Глобальные событие, генерируемые корневым элементов виджета и доступные любой части GUI
2. Локальные события, генерируемые компонентами виджета и доступные только его контроллеру

Такое разделение позволяет строго ограничить область ответственности виджета и инкапсулировать логику реакции на события.

## Глобальные события
Учитывая то, что глобальные события виджета генерируются его корневым элементом, не возникает необходимости использовать специальное пространство имен для имени события, так как достаточно имени класса корневого элемента, но это не запрещается:

{% highlight js %}
SearchWidget.prototype.onSearch = function(){
  this.$el.trigger('search.start'); // При наличии класса у корневого элемента
  // или
  this.$el.trigger('searchWidget.search.start'); // Без классификации корневого элемента

  // Выполнение поиска

  this.$el.trigger('search.done'); // При наличии класса у корневого элемента
  // или
  this.$el.trigger('searchWidget.search.done'); // Без классификации корневого элемента
};
{% endhighlight %}

## Локальные события
Как и при генерации глобальных событий, можно использовать классификацию элемента виджета в качестве пространства имен события. Важно лишь помнить, что этот тип событий должны генерироваться и обрабатываться только к контексте родительского виджета. Более того, недопустимо обрабатывать локальные события виджетом, включающим данный виджет в качестве элемента.

Пример правильной обработки локальных событий:
{% highlight js %}
function SearchWidget(model, el){
  // Инициализация виджета

  this.events = {
    'click search__button': this.onSearch.bind(this) // Обработка локальных событий
  };
}

SearchWidget.prototype.onSearch = function(){
  // Выполнение поиска
}
{% endhighlight %}

Пример неправильной обработки локальных событий:
{% highlight js %}
// Данный виджет включает SearchWidget в качестве элемента
function HeaderWidget(model, el){
  // Инициализация виджета

  this.events = {
    'click search__button': this.onListener.bind(this) // Неправильная обработка локальных событий. Реакция на локальные события стороннего виджета
  };
}
{% endhighlight %}
