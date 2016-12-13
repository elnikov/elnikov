---
layout: post
title:  "Генерация JSON с помощью PostgreSQL"
date:   2016-09-12 13:30:32 +0300
categories: postgre rails json
---

Одно из главных требований для любого онлайн бизнеса, это иметь бэкнд, который обеспечивает доступ к API интерфейсу. Создание статического HTML с jquery ajax в прошлом веке. В новой эре, рынком правят JavaScript фреймворки. Следовательно, JSON становится нитью, связывающий бэкенд и фронтэнд.

Rails имеет встроенную поддержку JSON. Этого достаточно, на определенном этапе. Вскоре, проиводительность проекта достигнет своего придела, и нужно либо докупать сервера, либо использовать более производительные языки, такие как Elixir, Go, и.т.п. До того как мы это сделаем, мы можем возложить генерацию сложны JSON на базу данных, что ускорит Rails примерно в 10 раз. 

Начиная c **PostgreSQL 9.2**, субд начинает поддержку JSON. Она состоит из:

* Сохранение ифнормациив формате JSON и JSONB
* Генерация JSON из запроса

В это статье будет говорить о генерации JSON из запроса.


#### Как генерировать JSON

Простейший способ это использовать функцию **row_to_json()**
Например, запрос который возвращает пользователя с id=1

{% highlight SQL %}
 select row_to_json(users) from users where id = 1;
{% endhighlight %}

Результат:

{% highlight JSON %}
{"id":1,"email":"hsps@redpanthers.co","encrypted_password":"iwillbecrazytodisplaythat",
"reset_password_token":null,"reset_password_sent_at":null,
"remember_created_at":"2016-11-06T08:39:47.983222",
"sign_in_count":11,"current_sign_in_at":"2016-11-18T11:47:01.946542",
"last_sign_in_at":"2016-11-16T20:46:31.110257",
"current_sign_in_ip":"::1","last_sign_in_ip":"::1",
"created_at":"2016-11-06T08:38:46.193417",
"updated_at":"2016-11-18T11:47:01.956152",
"first_name":"Super","last_name":"Admin","role":3}
{% endhighlight %}

Если нужно выбрать определенные поля:

{% highlight SQL %}
select row_to_json(results)
from (
  select id, email from users
) as results
{% endhighlight %}

Результат:

{% highlight JSON %}
{"id":1,"email":"hsps@redpanthers.co"}
{% endhighlight %}

Теперь разберем как генерировать JSON с вложенными структурами:

{% highlight SQL %}
select row_to_json(result)
from (
  select id, email,
    (
      select array_to_json(array_agg(row_to_json(user_projects)))
      from (
        select id, name
        from projects
        where user_id=users.id
        order by created_at asc
      ) user_projects
    ) as projects
  from users
  where id = 1
) result
{% endhighlight %}

Результат:

{% highlight JSON %}
{"id":1,"email":"hsps@redpanthers.co", "project":["id": 3, "name": "CSnipp"]}
{% endhighlight %}

Проблема в том, у приведенного выше кода низкая читаемость, по сравненеию с Ruby кодом. В этом случае приходится жертвовать удобством. Поэтому его нужно использовать только на узких местах проекта. 

Метод **array_agg** , мы можем заменить на **json_agg**.

{% highlight SQL %}
array_to_json(array_agg(row_to_json(user_projects)))
{% endhighlight %}

Можно сократить до:

{% highlight SQL %}
json_agg(user_projects)
{% endhighlight %}

**PostgreSQL 9.4**, ввели новый метод json_build_object. Пример использования

{% highlight SQL %}
json_build_object('foo',1,'bar',2)
{% endhighlight %}

Результат:

{% highlight JSON %}
{"foo": 1, "bar": 2}
{% endhighlight %}

Так же мы можем использовать встроенные функции СУБД для генерации JSON. Конечно, делая это, мы перемещаем логику из кода в базу данных и нам потребуется делать миграции при каждом изменении логики.  Поэтому этот способ рекоммендуется использовать только на сложных участках проекта.

Ссылки:

https://www.postgresql.org/docs/current/static/functions-json.html

http://bytefish.de/blog/postgresql_json/
