---
layout: post
title:  "Rails 5 Action Cable и WebSockets"
date:   2016-12-14 10:00:32 +0300
categories: rails5 actioncable websocket
---

Недавно был выложен исходный код одной из самой ожидаемой библиотеки **ActionCable**, которая будет составе Rails 5. Для тех кто ещё ничего не слышал о библиотеке **ActionCable**, это websocket фреймворк. Он упростит разработку приложений реального времени на **Rails**.

В этой статье, мы сделаем очень простое приложение, которое использует **ActionCable**. Это будет простой чат, где люди смогут выбрать себе никнейм и отправлять сообщения в чат. Мы рассмотрим только как добавить realtime функционал в приложение, и не будем слишком углубляться в подробности.


### Gemfile

Создайте новое приложение **rails new chat** и добавьте **actioncable** и **puma** в gemfile.

{% highlight Ruby %}
  source 'https://rubygems.org'

  gem 'rails', '4.2.3'
  gem 'actioncable', github: 'rails/actioncable'

  gem 'sqlite3'
  gem 'coffee-rails', '~> 4.1.0'
  gem 'jquery-rails'
  gem 'turbolinks'
  gem 'puma'
  gem 'uglifier', '>= 1.3.0'

  group :development, :test do
    gem 'byebug'
    gem 'spring'
    gem 'web-console', '~> 2.0'
  end
{% endhighlight %}

Мы добавили **puma**, потому что **ActionCable** способен работать только в отдельном процессе и нуждается в веб-сервере, который способен это обеспечить. 

### Добавляем views

Мы будем отображать сообщения в *messages#index*. Прежде чем войти в комнату, мы попросим пользователя указать свой никнейм, который будет отображаться вместе с сообщением. Форма выбора ника будет находится в *sessions#new*. Добавим эти пути в routes.rb:

{% highlight Ruby %}
  Rails.application.routes.draw do
    resources :messages, only: [:index, :create]
    resources :sessions, only: [:new, :create]
    root 'sessions#new'
  end
{% endhighlight %}

Далее, нужно изменить *SessionsController*. Добавим в метод *#create* функционал, который записывает cookie.

{% highlight Ruby %}
  #app/controllers/sessions_controller.rb
  class SessionsController < ApplicationController
    def create
      cookies.signed[:username] = params[:session][:username]
      redirect_to messages_path
    end
  end
{% endhighlight %}

Так же добавим шаблон для *sessions#new*

{% highlight Ruby %}
#app/views/sessions/new.html.erb:
  <%= form_for :session, url: sessions_path do |f| %>
    <%= f.label :username, 'Enter a username' %><br/>
    <%= f.text_field :username %><br/>
    <%= f.submit 'Start chatting' %>
  <% end %>
{% endhighlight %}

Сейчас, метод *#create* просто отдает success code на каждый запрос. В будущем, в метод добавим фунционал actioncable.

{% highlight Ruby %}
  class MessagesController < ApplicationController
    def create
      head :ok
    end
  end
{% endhighlight %}

Шаблон messages/index выглядит так:

{% highlight Ruby %}
  Signed in as @<%= cookies.signed[:username] %>.
  <br/><br/>

  <div id='messages'></div>
  <br/><br/>

  <%= form_for :message, url: messages_path, remote: true, id: 'messages-form' do |f| %>
    <%= f.label :body, 'Enter a message:'  %><br/>
    <%= f.text_field :body %><br/>
    <%= f.submit 'Send message' %>
  <% end %>
{% endhighlight %}

Задав параметр **remote: true**, мы отправляем запрос без перезагрузки страницы.


### Настройка ActionCable

Нам нужно два класса *ApplicationCable::Connection* и *ApplicationCable::Channel*. Позже, мы добавим superclass, для нашего собственно класса channel.

{% highlight Ruby %}
  # app/channels/application_cable/connection.rb
  module ApplicationCable
    class Connection < ActionCable::Connection::Base
    end
  end

  # app/channels/application_cable/channel.rb
  module ApplicationCable
    class Channel < ActionCable::Channel::Base
    end
  end
{% endhighlight %}

ActionCable использует Redis для хранения информации, необходимо настроить Redis в config/redis/cable.yml. 

{% highlight Ruby %}
  local: &local
    :url: redis://localhost:6379
    :host: localhost
    :port: 6379
    :timeout: 1
    :inline: true
  development: *local
  test: *local
{% endhighlight %}

ActionCable запускается в отдельном процессе, так что ему требуется свой собственный *rackup* файл, который мы разместим в  cable/config.ru.

{% highlight Ruby %}
  # cable/config.ru
  require ::File.expand_path('../../config/environment',  __FILE__)
  Rails.application.eager_load!

  require 'action_cable/process/logging'

  run ActionCable.server
{% endhighlight %}

Мы запустим cable сервер испозльуя puma на порт 28080, дбавим скрипт в bin/cable.

{% highlight Ruby %}
  # /bin/bash
  bundle exec puma -p 28080 cable/config.ru

  # Добавляем наш MessagesChannel
  # Финальный шаг в настройке чата это настройка MessagesChannel:
  # app/channels/messages_channel.rb
  class MessagesChannel < ApplicationCable::Channel
    def subscribed
      stream_from 'messages'
    end
  end
{% endhighlight %}

Когда клиент подпишется на MessagesChannel, вызовется метод #subscribed, который начнет броадкастить все что находится в канале Messages.

### Отправляем сообщения в поток

Теперь мы можем изменить метод messages#create.

{% highlight Ruby %}
  class MessagesController < ApplicationController
    def create
      ActionCable.server.broadcast 'messages',
        message: params[:message][:body],
        username: cookies.signed[:username]
      head :ok
    end
  end
{% endhighlight %}

### Подписка на сообщения на клиентской стороне

Сначала, создадим соединение к серверу  app/assets/javascripts/channels/index.coffee:

{% highlight Ruby %}
  #= require cable
  #= require_self
  #= require_tree .

  @App = {}
  App.cable = Cable.createConsumer 'ws://127.0.0.1:28080'

  # We will subscribe to MessagesChannel by adding the following code to app/assets/javascripts/channels/messages.coffee:
  # Мы попишемся к MessagesChannel добавив следущий код в app/assets/javascripts/channels/messages.coffee:

  App.messages = App.cable.subscriptions.create 'MessagesChannel',
    received: (data) ->
      $('#messages').append @renderMessage(data)

    renderMessage: (data) ->
      "<p><b>[#{data.username}]:</b> #{data.message}</p>"
{% endhighlight %}

Функция *App.messages.received* вызвается когда клиент получает что либо через websocket. Входяшие данные находятся в формате JSON, то что мы можем получить доступ через data.username* и *data.message* в функции *renderMessage*.

Нужно удостовериться что наш код включен в application.js файл. Сразу после require_tree . в application.js, добавим эту строку.

{% highlight Ruby %}
  //= require channels
{% endhighlight %}

Чат, основанный на ActionCable готов. Убедитесь что вы запустили puma сервер. Не забывайте перезапускать сервер при каждом изменений кода.

Откройте два браузера с разными cookie. (можно открыть обычны хром и хром запущенный в режиме инкогнито) и попробуйте отправить сообщения в чат, используя странцу messages#index.

