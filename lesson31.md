# Урок 31. Django ORM. Объекты моделей и queryset, Meta моделей, прокси модели.

![](https://cs8.pikabu.ru/post_img/2016/09/12/5/og_og_1473660997242355939.jpg)

Мы уже знаем про то как хранить данные, и как связать таблицы между собой, давайте научимся, извлекать, модифицировать и удалять данные при помощи кода.

Допустим, что ваша модель выглятит так:

```python
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone
from django.utils.translation import gettext as _

GENRE_CHOICES = (
    (1, _("Not selected")),
    (2, _("Comedy")),
    (3, _("Action")),
    (4, _("Beauty")),
    (5, _("Other"))
)


class Author(models.Model):
    pseudonym = models.CharField(max_length=120, blank=True, null=True)
    name = models.CharField(max_length=120)

    def __str__(self):
        return self.name


class Article(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE, null=True)
    text = models.TextField(max_length=10000, null=True)
    created_at = models.DateTimeField(default=timezone.now)
    updated_at = models.DateTimeField(default=timezone.now)
    genre = models.IntegerField(choices=GENRE_CHOICES, default=1)

    def __str__(self):
        return "Author - {}, genre - {}, id - {}".format(self.author.name, self.genre, self.id)


class Comment(models.Model):
    text = models.CharField(max_length=1000)
    article = models.ForeignKey(Article, on_delete=models.DO_NOTHING)
    comment = models.ForeignKey('myapp.Comment', null=True, blank=True, on_delete=models.DO_NOTHING,
                                related_name='comments')
    user = models.ForeignKey(User, on_delete=models.DO_NOTHING)

    def __str__(self):
        return "{} by {}".format(self.text, self.user.username)


class Like(models.Model):
    user = models.ForeignKey(User, on_delete=models.DO_NOTHING)
    article = models.ForeignKey(Article, on_delete=models.DO_NOTHING)

    def __str__(self):
        return "By user {} to article {}".format(self.user.username, self.article.id)

```

Рассмотрим некоторые новые возможности

```python
from django.contrib.auth.models import User
```

Это модель встроенного в Django юзера, её мы рассмотрим немного позже.

```python
from django.utils.translation import gettext as _
```

Стандартная функция перевода языка для Django, допустим что ваш сайт имеет функцию переключения языка, с русского, украинского и английского, эта функция поможет нам в будущем указать значения для всех трех языков. Подробнейшая информация по переводам [Тут](https://docs.djangoproject.com/en/2.2/topics/i18n/translation/)

```python
GENRE_CHOICES = (
    (1, _("Not selected")),
    (2, _("Comedy")),
    (3, _("Action")),
    (4, _("Beauty")),
    (5, _("Other"))
)
```

Переменная состоящяя из тупла туплов, нужна для использования choices значений, используется для хранения выбора чего либо, в нашем случае жанра, в базе будет храниться только число, а пользователю будет выводиться уже текст.

Используем это вот тут:

```python
genre = models.IntegerField(choices=GENRE_CHOICES, default=1)
```

Рассмотрим вот эту строку

```python
return "Author - {}, genre - {}, id - {}".format(self.author.name, self.genre, self.id)
```

**self.author.name** - в базе по значению форейн кея хранится айди, но в коде мы можем получить доступ к значениям связанной модели, конкретно в этой ситуации, мы берем значение поля **name** из связанной модели **author**.


Рассмотрим вот эту строку:

```python
comment = models.ForeignKey('myapp.Comment', null=True, blank=True, on_delete=models.DO_NOTHING,
                                related_name='comments')
    
```

Модель можно передать не только как класс, но и по имени модели указав приложение `appname.Modelname` (Да мне было лень переименовывать приложение из myapp, во что-то читаемое)

При такой записи мы создаём связь один ко многим с самому себе, указав при этом black=True, null=True. При таком случае у нас первый коментарий, будет без ссылки на коментарий, и по этому принципу мы можем выбрать коментарии к сути, а если создать коментарий со ссылкой на другой, это будет коментарий к коментарию, причем это можно сделать любой вложенности. 

`related_name` - в этой записи нужен для того, что бы получить выборку всех вложенных объектов, мы рассмотрим их немного ниже.


## Meta моделей

В некоторых ситуациях нам необходимо иметь возможность задать определённые условия на уровне модели, например порядок сохранения объектов или другие особые условия, тут нам на помощь приходит встроенный класс `Meta`

```python
from django.db import models

class Ox(models.Model):
    horn_length = models.IntegerField()

    class Meta:
        ordering = ["horn_length"]
        verbose_name_plural = "oxen"
```

синтаксис такой конструкции нужно просто запомнить.

В мете может быть большое кол-во свойст моделей, давайте рассмотрим основные (полный список [тут](https://docs.djangoproject.com/en/3.1/ref/models/options/))

### ordering

Содержит список из строк соответусвующих названиям полей(атрибутов) могут быть указанны со знаком `-` что бы указать, обратный порядок, в каком порядке указанны такой приоритет полей и будет, например, если указан ``` ordering = ['name', '-age'] ``` то мы объекты будут рассположены в базе по полю `name` и в случае совпадения этого поля, по полю `age`, в обратном порядке.

Может быть указана при помощи `F` обхектов, о них позже.

### unique_together

принимает колекцию коллекций, например список списков, каждый список, должен содержать набор строк, с именами полей. При указании, этого набора, данные поля будут совместно уникальны (Если совместно уникальны имя и фамилия, то может быть сколько угодно объектов с именем `Мария`, и сколько угодно объектов с фамилией `Петрова`, но только один объект с такой комбинацией.)

```
unique_together = [['driver', 'restaurant'], ['driver', 'phone_number']]
```

Если есть только одно нужное значение может быть одним списком.

```
unique_together = ['driver', 'restaurant']
```

### verbose_name и verbose_name_plural

свойства содержащие строки и отвечающие за то какие имена будут описанны в админке, в единственном и множественно числе соответсвенно


## Абстрактные и прокси модели

### Абстрактные модели

Абстрактные классы моделей, это `заготовки` под дальнейшие модели которые не создают дополнительных таблиц. Например некоторые из ваших моделей должны содержать поле `created_at`, в котором будет сранится информация о том, когда обхект создан, для этого можно в каждой можели прописать это поле, или один раз описать абстрактную модель с одним полем, инаследоваться уже от неё.

Модель как абстрактная указывается в мете.

Синтаксис:

```python
from django.db import models

class CommonInfo(models.Model):
    name = models.CharField(max_length=100)
    age = models.PositiveIntegerField()

    class Meta:
        abstract = True

class Student(CommonInfo):
    home_group = models.CharField(max_length=5)
```

`Таблица для CommonInfo не будет созданна!!!`

`Meta` не наследуется !!

Для наследования меты нужно использовать вот какой синтаксис (явное наследование меты):

```python
from django.db import models

class CommonInfo(models.Model):
    name = models.CharField(max_length=100)
    age = models.PositiveIntegerField()

    class Meta:
        abstract = True
        ordering = ['name']

class Unmanaged(models.Model):
    class Meta:
        abstract = True
        managed = False

class Student(CommonInfo, Unmanaged):
    home_group = models.CharField(max_length=5)

    class Meta(CommonInfo.Meta, Unmanaged.Meta):
        pass
```

### Прокси модели

Модель которая создаётся на уровне языка программирования но не на уровне базы данных. Используется если нужно добавить метод, измениеть поведение менеджера итд.

Синтаксис:

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)

class MyPerson(Person):
    class Meta:
        proxy = True

    def do_something(self):
        # ...
        pass
```

В базе будет хранится одна таблица, в Django две.

```python
>>> p = Person.objects.create(first_name="foobar")
>>> MyPerson.objects.get(first_name="foobar")
<MyPerson: foobar>
```

Часто используется для отображении в админке нескольких таблиц для одного объекта.

## objects и shell

Для доступа или модификации любых данных, у каждой модели есть аттрибут `objects`, который позволяет производить любые манипуляции с данными.

Для интреактивного использования кода используется команда

```python
python manage.py shell
```

Эта команда открывает нам консоль с уже испортироваными всеми стандатрными но не самописными модулями Django

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson36/clean_shell.png)

Предварительно я создал несколько объектов через админку.

Для доступа к моделям, их нужно импортировать, я импортирую модель Comment

Рассмотрим весь CRUD и дополнительные особенности. Очень подробная информация по всем возможным операциям [Тут](https://docs.djangoproject.com/en/2.2/topics/db/queries/) 

### R - retrieve

Функции для получения объектов в Django могут возвращать, два типа данных, **объект модели** или **queryset**

Объект, это еденичный объект, queryset это по сути список объектов, со своими встроенными методами.

#### all

Для получения всех данных используется метод `all()`, возвращает queryset со всеми существующими объектами этой модели.

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson36/objects_all.png)

#### filter

Для получения отфильтрованных данных, мы используем метод `filter()`

Если указать фильтр без параметров, то он сделает тоже самое что и all.

Какие у фильтра могу быть парметры? Да практически любые, мы можем указать любые поля для фильтрации.
Например фильтр по полю текст.
```python
Comment.objects.filter(text='Hey everyone')
```

Фильтр по вложенным объектам, выполняется через двойное подчеркивание.

Фильтр по жанру статьи коментария.

```python
Comment.objects.filter(article__genre=3)
```

По превдониму автора.

```python
Comment.objects.filter(article__author__pseudonym='The king')
```

По превдониму автора и жанру.

```python
Comment.objects.filter(article__author__pseudonym='The king', article__genre=3)
```

Так же у кадого поля существуют встроенные системы лукапов, пишутся с таким же синтаксисом как и доступ к форейн кеям `field__lookuptype=value`

Стандартные лукапы:

`lte` - меньше или равно

`gte` - больше или равно

`lt` - меньше

`gt` - больше

`startswith` - Начинается с

`istartswith` - Начинается с, без учёта регистра

`endswith` - Заканчивается на

`iendswith` - Заканчивается на, без учёта регистра

`range` - находится в рамках

`week_day` - день недели (для дат)

`year` - год (для дат)

`isnull` - является наном

`contains` - частично содержит учитывая регистр ("Всем привет, я Влад" содержит слово "Влад", но не содержит "влад")

`icontains` - тоже самое, но регистро независимо, теперь найдется и второй вариант.

`exact` - совпадает (не обязательный лукап, делает тоже, что и знак равно)

`iexact` - совпадает регистро независимо (по запросу "привет" найдет и "Привет" и "прИвЕт")

`in` - содержится в каком то списке

Их намного больше, читать [ТУТ](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#field-lookups)

Примеры:

Псевдоним содердит слово 'king'
```python
Comment.objects.filter(article__author__pseudonym__icontains='king')
```

Коменты к статье созданной не познее чем вчера

```python
Comment.objects.filter(article__created_at__gte=date.today() - timedelta(days=1))
```

Коменты к статьям с жанрами под номерами 2 и 3

```python
Comment.objects.filter(article__genre__in=[2,3])
```

Существуют более сложные и комплексные фильтры почитать про них можно [Тут](https://docs.djangoproject.com/en/2.2/ref/models/expressions/#django.db.models.F) и [Тут](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#django.db.models.Q)

#### exclude

Функция обратная функции `filter` вытащит всё что не попадает ввыборку

Напиример все коменты к статьям у которых жанр не 2 и не 3

```python
Comment.objects.exclude(article__genre__in=[2,3])
```

Фильтр и эксклюд можно совмещать. К любому кверисету можно применить фильтр или ексклюд еще раз
Например все коменты к статьям созданым не позже чем вчера, с жанрами не 2 и не 3

```python
Comment.objects.filter(article__created_at__gte=date.today() - timedelta(days=1)).exclude(article__genre__in=[2,3])
```

#### order_by

По умолчанию все модели сортируются по айди, если явно не указанно иное, но часто нужно отсортировать данные специальным образом для этого используется order_by()

Как явно указать сортировку и какие еще тайны хранят модели, можно прочитать [Туть](https://docs.djangoproject.com/en/2.2/ref/models/options/)

Особенно внимательно читаем `ordering`, `unique_together`, `verbose_name`

```python
Comment.objects.filter(article__created_at__gte=date.today() - timedelta(days=1)).order_by('-article__created_at').all()
```

Все методы кверисетов читаем [Тут](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#methods-that-return-new-querysets)

Особое внимание на методы `distinct`, `reverse`,  `values`, `difference`

Так же во все фильтры можно вставлять целые объекты, например

```python
article = Article.objects.get(id=2)
comments = Comment.object.exclude(article=article)
```

### объектные методы

#### get

В отличии от filter и exclude получает сразу объект, работает только если можно определить объект однозначно и он существует.

Можно применять теже условия что и для фильтра и эксклюду

Например получения объекта по айди

```python
Comment.objects.get(id=3)
```

Если объект не найден или найдено больше одного объекта по заданым параметрам, вы получите исключение, которое желательно всегда обрабатывать
Исключения уже находятся в самой модели.

```python
try:
    Comment.objects.get(article__genre__in=[2,3])
except Comment.DoesNotExist:
    return "Can't find object"
except Comment.MultipleObjectsReturned:
    return "More than one object"
```

#### first и last

К кверисету можно применять методы first и last что бы получить первый или последний элемент кверисета

Например получить первый комент написанный за вчера

```python
Comment.objects.filter(article__created_at__gte=date.today() - timedelta(days=1)).first()
```

Информация по всем остальным методам [Тут](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#methods-that-do-not-return-querysets)


### C - Create

Для создания новых объектов используется два возможных варианта, через метод, `create` или метод `save`

Создадим двух новых авторов, при помощи разных методом

```
Author.objects.create(name='New author by create', pseudonym="Awesome author")

a = Author(name="Another new author", pseudonym="Gomer")
a.save()
```

В чём же разница? В том, что в первом случае, при выполнении этой строки запрос в базу будет отправлен сразу, во втором, только при вызове метода `save()`

Метод save так же является и методом для апдейта, если применяется для уже существующего объекта, рассмотрим его подробнее немного дальше.

### U - Update

Для апдейта используется метод `update()`

**Применяется только к кверисетам, к объекту применить нельзя**

Напиример обновим текст в коментарии с айди = 3

```python
Comment.objects.filter(id=3).update(text='bla-bla')

ИЛИ

c = Comment.objects.get(id=3)
c.text = 'bla-bla'
c.save()
```

### D - Delete

Как можно догадаться, выполняется методом `delete()` тоже применяется только к кверисетам.

Удалить все коменты от пользователя с айди 2
```python
Comment.objects.filter(user__id=2).delete()
```

### Совмещенные методы

#### get_or_create() update_or_create() bulk_create() bulk_update()

get_or_create Метод который попытается создать новый объект если не сможет найти нужный в базе
 
update_or_create Обновит если объект существует, создаст если не существует

bulk_create - массовое создание

bulk_update - массовые апдейт (отличие от обычного в том, что при обычном на каждый объект создается запрос, в этом случае запрос делается массово)

Подробно почитать про них [Тут](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#get-or-create)

## Подробнее о методе save

Метод save() применяется при любых изменениях или создании данных, но очень часто нужно что-бы при сохраннии данных выполнялись еще какие-либо действия, переписывание данных или запись логов итд. Для этого используется переписывание метода save()

Допустим мы хотим делать время создания статьи на один день раньше чем фактическая. Перепишем метод `save` для статьи

```python
def save(self, **kwargs):
    self.created_at = timezone.now() - timedelta(days=1)
    super().save(**kwargs)
```

Переопределяем значение, и вызываем оригинальный сейв, вуаля.

Для того, что бы переопределить логику при создании но не трогать при изменении, или наоборот, используется особенность данных, у уже созданного объекта `id` существует, у нового нет. Так что фактически наш код сейчас обновляет это поле всегда, и когда надо и когда не надо, допишем его.


```python
def save(self, **kwargs):
    if not self.id:
        self.created_at = timezone.now() - timedelta(days=1)
    super().save(**kwargs)
```

теперь поле будет переписываться только в момент создания, но не будет трогаться при обновлении.

## Сложные SQL конструкции

На самом деле мы не ограниченны стандартными конструкциями, мы можем применять предвычисления на уровне базы, добавлять логические конструкции итд., давайте рассмотри подробнее.

### Q объекты

Как вы могли заметить в случае фильтрации, мы можем выбрать объекты через логическое И, при помощи запятой.

```python
Comment.objects.filter(article__author__pseudonym='The king', article__genre=3)
```

В этом случае у нас есть кострукция типа выбрать объекты у которых псевдоним автора статьи это `The king` И жанр статьи это цифра 3

Но что же нам делать есть нам нужно использовать логическое ИЛИ.

В этом нам поможет использование Q объекта, на самом деле каждое из этих уловий мы могли завернуть в такой объект:

```python
from django.db.models import Q
q1 = Q(article__author__pseudonym='The king')
q2 = Q(article__genre=3)
``` 

Теперь мы можем явно использовать логические И и ИЛИ.

```
Comment.objects.filter(q1&q2) # И
Comment.objects.filter(q1|q2) # ИЛИ
```

### Aggregation

Агрегация в джанго это по сути предвычисления.

Допустим что у нас есть модели:

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    age = models.IntegerField()

class Publisher(models.Model):
    name = models.CharField(max_length=300)

class Book(models.Model):
    name = models.CharField(max_length=300)
    pages = models.IntegerField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    rating = models.FloatField()
    authors = models.ManyToManyField(Author)
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
    pubdate = models.DateField()

class Store(models.Model):
    name = models.CharField(max_length=300)
    books = models.ManyToManyField(Book)
```

Мы можем совершить предвычисления каких либо средних, минимальных, максимальных значени, вычислить сумму итд.

```
>>> from django.db.models import Avg
>>> Book.objects.all().aggregate(Avg('price'))
{'price__avg': 34.35}
```

на самом деле all() не несёт пользы в этом примере

```
>>> Book.objects.aggregate(Avg('price'))
{'price__avg': 34.35}
```

Значение можно именовать 

```python
>>> Book.objects.aggregate(average_price=Avg('price'))
{'average_price': 34.35}
```

Можно вносить больше одной агрегации за раз

```
>>> from django.db.models import Avg, Max, Min
>>> Book.objects.aggregate(Avg('price'), Max('price'), Min('price'))
{'price__avg': 34.35, 'price__max': Decimal('81.20'), 'price__min': Decimal('12.99')}
```

Если нам нужно, что бы подсчитаное значение было у каждого объекта модели, мы используем метод `annotate`

```python
# Build an annotated queryset
>>> from django.db.models import Count
>>> q = Book.objects.annotate(Count('authors'))
# Interrogate the first object in the queryset
>>> q[0]
<Book: The Definitive Guide to Django>
>>> q[0].authors__count
2
# Interrogate the second object in the queryset
>>> q[1]
<Book: Practical Django Projects>
>>> q[1].authors__count
1
```

Их тоже может быть больше одного

```python
>>> book = Book.objects.first()
>>> book.authors.count()
2
>>> book.store_set.count()
3
>>> q = Book.objects.annotate(Count('authors'), Count('store'))
>>> q[0].authors__count
6
>>> q[0].store__count
6
```

Все эти вещи можно комбинировать

```python
>>> highly_rated = Count('book', filter=Q(book__rating__gte=7))
>>> Author.objects.annotate(num_books=Count('book'), highly_rated_books=highly_rated)
```

C ордерингом

```
>>> Book.objects.annotate(num_authors=Count('authors')).order_by('num_authors')
```

## F -  выражения

F выражения нужны для получения значения поли, и оптимизации записи. [Дока](https://docs.djangoproject.com/en/3.1/ref/models/expressions/#f-expressions)

Допустим нам нужно увеличить опреденному объекту в базе значение какого либо поля на 1
```
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed += 1
reporter.save()
```

На самом деле в этот момент мы получаем значение из базы в панят обрабатываем, и записываем в базу

Есть другой путь:

```
from django.db.models import F

reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed = F('stories_filed') + 1
reporter.save()
```

Преимущества под капотом, но давайте предположим, что нам нужно сделать эту же операцию массово

```
reporter = Reporters.objects.filter(name='Tintin')
reporter.update(stories_filed=F('stories_filed') + 1)
```

Такие объекты можно использовать и в анотации и в фильтрах и во многих других местах.

## Методы и свойства модели User

## Модель User

Django предоставляет нам встроенную модель юзера, у которой уже реализовано много полей и методов, находится в ``

Подробнейшая инфа про юзера [Тут](https://docs.djangoproject.com/en/2.2/topics/auth/default/)

Стандартный юзер содержит в себе такие полезные поля как:

```python
username
password
email
first_name
last_name
```
Также содержит много системных полей, рассмотрим позже.
Также содержит базовый метод set_password и информацию о группах доступа.

Рассмотрим их дальше.

Для использования модели пользователя, которую нам нужно видоизменить используется наследование от базового абстрактного юзера

Выглядит примерно так:

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class MyUser(AbstractUser):
    birth_date = models.DateField()
    avatar = models.ImageField(blank=True, null=True)
```

Для того, что бы Django оценивала эту модель как пользователя, в `settings.py` нужно в любом месте указать:

```python
AUTH_USER_MODEL = 'myapp.MyUser' 
```
где myapp - название приложения, MyUser - название модели.

Все возможные подробности про модель юзера [Тут](https://docs.djangoproject.com/en/2.2/ref/contrib/auth/#django.contrib.auth.models.User)

Кроме стандарных полей юзер содержит в себе информацию о группах которых состоит пользователь, о пользовательских правах, содержит два поля статуса `is_staff` и `is_superuser` чаще всего стафф это сотрудника которым можно в админку, но у них ограниченные права, супербзеру можно всё, но всегда зависит от ситуации.

Также хранит инфо о последнем логине пользователя и дате согдания пользователя.

### Содержит кучу полезных методов

```python
get_username() - получить юзернейм

get_full_name() - получить имя и фамилию через пробел

set_password(raw_password) - установить хешированый и подсоленый пароль

check_password(raw_password) - проверить пароль на правильность

set_unusable_password() - разрешить использовать пустую строку как пароль

email_user(subject, message, from_email=None, **kwargs) - отправить пользователю имейл 
```

И другие методы, отвечающие за доступы, группы итд.

Содержит два метода create_user(username, email=None, password=None, **extra_fields) и create_superuser(username, email, password, **extra_fields)

Для создания новых пользователей другими пользователями.

Содержит поле `is_authenticated` необходимое для проверки, был ли пользователь авторизован.

Подробно разберём, когда будем делать систему логина.

# Домашнее задание:

Прочитать все ссылки из лекции!!! Буду спрашивать.

Выполнить всё через шел и залить на гит скрины в отдельной папке (Ну или в личку в телеграм):

Всё выполнять со своими моделями.

1. Придумать 10 разных фильтров или эксклюдов, с разными лукапами и заходом в связанные модели.

2. Получить 5 последних написанных коментариев (именно текст).

3. Создать 5 коментариев с разным текстом, Хотя бы один должен начинаться со слова "Start", хоть один в середине должен иметь слово "Middle", хоть один должен заканчиваться словом "Finish".

4. Переписать сейв коментария так, что бы при создании дата менялась бы на год назад (если сегодня 20 декабря 2019, должна выставляться 20 декабря 2018), изменение коментариев не затрагивать.

5. Изменить коментарии со спец словами "Start", "Middle", "Finish".

6. Удалить все коментарии у которых в тексте есть буква "k", но не удалять если есть буква "с".

7. Получить первые 2 коментария по дате создания к статье у которой имя автора последнее по алфавиту.