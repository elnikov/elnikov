---
layout: post
title:  "Наследование C#. Полиморфизм"
date:   2016-04-10 08:30:32 +0300
categories: csharp
---
Как применять на практики полиморфизм классов C#.

Можно понять на примере:
Есть массив из полигонов. Каждый элемент массива может быть кругом, квадратом, полигоном и неинициализированным объектом. 

Для начала нужно создать базовый класс Polygon.
{% highlight C# %}
	class Polygon
	{
		public int NumberOfSides { get; set; }
		public Polygon()
		{
			NumberOfSides = 0;
		}
		public Polygon(int numberOfSides)
		{
			NumberOfSides = numberOfSides;
		}
	}
{% endhighligh %}

Затем два дочерних класса Square и Circle, которые наследуют базовый класс Polygon
{% highlight C# %}
class Square : Polygon
	{
		public float Size { get; set; }
		public Square(float size)
		{
			Size = size;
			NumberOfSides = 4;
		}
	}

	class Circle : Polygon
	{
		public float Radius { get; set; }
		public Circle(float radius)
		{
			Radius = radius;
		}
	}
{% endhighlight %}

Теперь массив из полигонов можно получить следующим образом:
{% highlight C# %}
			Polygon[] polygons= new Polygon[100];
			polygons [1] = new Square (10f);
			polygons [2] = new Circle (4f);
			polygons [3] = new Polygon (2);
{% endhighlight %}



Так как изначальный тип - Polygon, мы не можем получить доступ к переменным float Radius и float Size, хотя они безусловно сохранились.
Самый простой способ получить доступ к этим переменным:

{% highlight C# %}
      Square square = (Square)polygon[1];
			Console.WriteLine (square.Size);
			Circle circle = (Circle)polygon[2];
			Console.WriteLine (circle.Size);
{% endhighlight %}

Примеры кода можно посмотреть тут: [DotNetFiddle.net][dotnetfiddle]


[dotnetfiddle]: https://dotnetfiddle.net/7ca3hK
