# drf-my-cheat-sheet
Custom cheat-sheet for DRF tools and features

мой Cheat Sheet по Django REST Framework. Здесь будут мои записи для скорейшего освоения drf.

## Содержание
1. [Сериализация (Serialization)](#сериализация-serialization)
   - [1.1. Serializer](#11-serializer)
     - [Создание сериализатора](#создание-сериализатора)
     - [Сериализация объектов](#сериализация-объектов)
     - [Десериализация объектов](#десериализация-объектов)
     - [Сохранение экземпляров модели](#сохранение-экземпляров-модели)
     - [Валидация](#валидация)
     - [Доступ к instance and initial data](#доступ-к-instance-and-initial-data)
     - [Частичное обновление](#частичное-обновление)
     - [Вложенные сложные объекты](#вложенные-сложные-объекты)
     - [Записываемые вложенные представления](#записываемые-вложенные-представления)
     - [Сериализация нескольких объектов](#сериализация-нескольких-объектов)
     - [Включение дополнительного контекста](#включение-дополнительного-контекста)
   - [1.2. ModelSerializer](#12-modelserializer)
     - [Выбор полей](#выбор-полей)
     - [Указание вложенной сериализации](#указание-вложенной-сериализации)
     - [Явный выбор полей](#явный-выбор-полей)
     - [Указание полей для чтения](#указание-полей-для-чтения)
     - [Дополнительные аргументы и изменение полей модели](#дополнительные-аргументы-и-изменение-полей-модели)
     - [Настройка сопоставления полей](#настройка-сопоставления-полей)
     - [Классы и аргументы полей](#классы-и-аргументы-полей)
   - [1.3. HyperlinkedModelSerializer](#13-hyperlinkedmodelserializer)
     - [Абсолютные и относительные ссылки](#абсолютные-и-относительные-ссылки)
     - [Как определяются представления для гиперссылок](#как-определяются-представления-для-гиперссылок)
   - [1.4. ListSerializer](#14-listserializer)
     - [Настройка поведения ListSerializer](#настройка-поведения-listserializer)
     - [Настройка инициализации ListSerializer при неявном объявлении](#настройка-инициализации-listserializer-при-неявном-объявлении)
   - [1.5. BaseSerializer](#15-baseserializer)
     - [Read-only сериализатор на основе BaseSerializer](#read-only-сериализатор-на-основе-baseserializer)
     - [Read-write BaseSerializer](#read-write-baseserializer)






2. [Виды (Views)](#виды-views)
3. [Аутентификация (Authentication)](#аутентификация-authentication)
4. [Разрешения (Permissions)](#разрешения-permissions)

---

## Сериализация (Serialization)
преобразуют сложные типы данных(наборы запросов, экз моделей) в типы данных питона(словарь) и обратно.
которые потом можно преобразовать в .json и отдать через api.
в них передают модель и определяют какие ее поля будут преобразовываться.

### 1.1. Serializer
Базовый класс для создания сериализаторов:
#### Создание сериализатора

```python

from rest_framework import serializers
from datetime import datetime

# имеется экземпляр модели:
class Comment:
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()

comment = Comment(email='leila@example.com', content='foo bar')


# объявление сериализатора для модели:
class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```   

#### Сериализация объектов

```python   
# передаем в сериализатор экземпляр модели, получаем словарь:
serializer = CommentSerializer(comment)
serializer.data
# {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}


# из словаря получаем .json:
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'
```

#### Десериализация объектов

```python
import io
from rest_framework.parsers import JSONParser

# преобразуем поток в типы данных python(json=>dict):
stream = io.BytesIO(json)  # BytesIO
data = JSONParser().parse(stream)  # dict

# словарь отдаем сериалайзеру(с инстансом - обновление, без инстанса - создание нового объекта)
serializer = CommentSerializer(data=data)
serializer.is_valid()  # if True => validated_data становится доступен
serializer.validated_data  # dict, проверенные данные которые сохраняет в себе сериалайзер 
# чтобы потом их использовать при создании экз модели.
# {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}
comment = serializer.save() # сохраняем джанго модель, см ниже
```
#### Сохранение экземпляров модели
чтобы использовать данные которые получаем (десериализация) нужно иметь возможность 
создавать или обновлять экземпляр модели, для этого методы create, update)
```python
class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()

    def create(self, validated_data):
        return Comment(**validated_data)

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        return instance

# для моделей джанги сохраняем данные в бд:
def create(self, validated_data):
        return Comment.objects.create(**validated_data)  
        # save data in db and return model's instance

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        instance.save()  # save data in db and return model's instance
        return instance

# .save() will create a new instance.
serializer = CommentSerializer(data=data)

# .save() will update the existing `comment` instance.
serializer = CommentSerializer(comment, data=data)

# в save() можно передать доп данные, они передадутся в validated_data:
serializer.save(owner=request.user)
```
[к содержанию](#содержание)
#### Валидация
Можно использовать простые валидаторы(в классе сериализатора) или reusable валидаторы(определены вне сериализатора).
Все они могут применяться на уровне поля или объекта.
 
```python
# в случае ошибки вернет словарь где ключ-поле, значение-список с ошибками для этого поля:
serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
serializer.is_valid()  # False
serializer.errors
# {'email': ['Enter a valid e-mail address.'], 'created': ['This field is required.']}

from rest_framework import serializers

class BlogPostSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()

    def validate_title(self, value):
        """
        Check that the blog post is about Django.
        """
        if 'django' not in value.lower():
            raise serializers.ValidationError("Blog post is not about Django")
        return value
```

#### Доступ к instance and initial data
```python
# .instance содержит объект (или queryset), который передан в сериализатор:

class UserSerializer(serializers.Serializer):
    username = serializers.CharField()
    email = serializers.EmailField()

user = {'username': 'john_doe', 'email': 'john@example.com'}

# Создаём экземпляр сериализатора с объектом
serializer = UserSerializer(instance=user)

# Доступ к атрибуту .instance
print(serializer.instance)
# Вывод: {'username': 'john_doe', 'email': 'john@example.com'}

# Если instance не передан
empty_serializer = UserSerializer()
print(empty_serializer.instance)
# Вывод: None

# .initial_data содержит необработанные данные, переданные в сериализатор
# через аргумент data. Эти данные ещё не прошли валидацию:
input_data = {'username': 'john_doe', 'email': 'john@example.com'}
serializer = UserSerializer(data=input_data)

# Доступ к атрибуту .initial_data
print(serializer.initial_data)
# {'username': 'john_doe', 'email': 'john@example.com'}

# Если данные не переданы
empty_serializer = UserSerializer()
print(hasattr(empty_serializer, 'initial_data'))  # Проверка существования атрибута
# False
```
#### Частичное обновление
```python
# Update `comment` with partial data - must include partial=True:
serializer = CommentSerializer(comment, data={'content': 'foo bar'}, partial=True)
```

#### Вложенные сложные объекты
поле сериализатора может быть не простым объектом, а тоже сериализатором, None или списком:  
```python
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()  # если поле есть сериалайзер
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
    
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)  # если поле None
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()

class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)
    edits = EditItemSerializer(many=True)  # если поле это список
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```
#### Записываемые вложенные представления
Если возникают ошибки во вложенных полях, то они отображаются с исходной вложенностью;
так же будет и с полем .validated_data:
```python
serializer = CommentSerializer(data={'user': {'email': 'foobar', 'username': 'doe'}, 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'user': {'email': ['Enter a valid e-mail address.']}, 'created': ['This field is required.']}
```
Если используются вложенные объекты, то методы .create() and update() нужно переопределить:
```python
# create:
class UserSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer()

    class Meta:
        model = User
        fields = ['username', 'email', 'profile']

    def create(self, validated_data):
        profile_data = validated_data.pop('profile')
        user = User.objects.create(**validated_data)
        Profile.objects.create(user=user, **profile_data)
        return user

# update:
def update(self, instance, validated_data):
        profile_data = validated_data.pop('profile')
        # Unless the application properly enforces that this field is
        # always set, the following could raise a `DoesNotExist`, which
        # would need to be handled.
        profile = instance.profile

        instance.username = validated_data.get('username', instance.username)
        instance.email = validated_data.get('email', instance.email)
        instance.save()

        profile.is_premium_member = profile_data.get(
            'is_premium_member',
            profile.is_premium_member
        )
        profile.has_support_contract = profile_data.get(
            'has_support_contract',
            profile.has_support_contract
         )
        profile.save()

        return instance
```
[к содержанию](#содержание)

#### Сериализация нескольких объектов
```python
queryset = Book.objects.all()
serializer = BookSerializer(queryset, many=True)
serializer.data
# [
#     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
#     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
#     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
# ]
```
Десериализация нескольких объектов поддерживается по умолчанию на создание, но не на обновление.

#### Включение дополнительного контекста
В дополнение к сериализуемому объекту бывает нужно передать дополнительный 
контекст (контекстный словарь), например если поле сериализатора включает гиперссылку:
```python
serializer = AccountSerializer(account, context={'request': request})
serializer.data
# {'owner': 'denvercoder9', 'details': 'http://example.com/accounts/6/details'}
```

### 1.2. ModelSerializer
Если есть модель джанго, то этот класс автоматически создаст сериализатор с полями из этой модели, встроенной валидацией, и встроенными методами create, update:
```python
# объявление сериализатора:
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
```
Все отношения модели(внешние ключи) передадутся в поле типа PrimaryKeyRelatedField.
Обратные отношения требуют явного включения.
```python
# посмотреть созданный сериализатор и его поля:
# python manage.py shell:
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print(repr(serializer))
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```
#### Выбор полей
Можно выбрать какие поля модели передавать в сериализатор:
```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
        # fields = '__all__'  # или все поля
```
```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        exclude = ['users']  # исключить поля
```

#### Указание вложенной сериализации
`depth` указывает уровень вложенности связанных объектов, которые будут сериализованы.
По умолчанию ModelSerializer использует первичные ключи для указания отношений, но
если поле связано с другой моделью или является тоже сериализатором, тогда в это поле будет включен не только id, а все поля той модели(сериализатора).
```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
        depth = 1
```
```json
{
    "id": 1,
    "account_name": "My Account",
    "users": [
        {
            "id": 10,
            "username": "johndoe",
            "email": "johndoe@example.com"
        }
    ],
    "created": "2024-12-28T12:34:56Z"
}

```
Однако, большая глубина замедляет процесс сериализации.
#### Явный выбор полей
Если нужно добавить поля которых нет в модели или переопределить существующие
```python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)
    groups = serializers.PrimaryKeyRelatedField(many=True)

    class Meta:
        model = Account
        fields = ['url', 'groups']
```
#### Указание полей для чтения
Идивидуально для поля при его объявлении указать `read_only=True`.
Или передать в списке несколько полей через `read_only_fields`
```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
        read_only_fields = ['account_name']
```
Полями для чтения по умолчанию будут поля типа `AutoField` и поля со свойством `editable=False`.
Если поле, которое должно быть только для чтения, является частью ограничение `unique_together`, то ему нужно установить `read_only=True` и задать значение по-умолчанию: 

```python
# В джанго модели поля, которые должны быть уникальны в своей комбинации, перечисляются в `unique_together`
class MyModel(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    identifier = models.CharField(max_length=100)

    class Meta:
        unique_together = ('user', 'identifier')
```
Все поля находящиеся в `unique_together` будут участвовать в валидации, тк drf должен проверить, что выполняются условия этого ограничения. 
Если просто запретить редактирование через `read_only=True`, то при создании нового объекта валидация провалится, тк значение поля окажется незаданным, поэтому полю нужно определить значение по умолчанию.
```python
class MySerializer(serializers.ModelSerializer):
    user = serializers.PrimaryKeyRelatedField(
        read_only=True,
        default=serializers.CurrentUserDefault()
        # в поле юзер передастся текущий юзер
    )

    class Meta:
        model = MyModel
        fields = ['user', 'identifier']
```
[к содержанию](#содержание)

#### Дополнительные аргументы и изменение полей модели
Чтобы изменить поля модели переданные в сериализатор, нужно их передать в `extra_kwargs`
```python
class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['email', 'username', 'password']
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```
`extra_kwargs` нужны для изменения существующих полей модели, поэтому
изменения применятся при неявном объявлении поля(передано из модели), и не применятся при явном объявлении(в сериализаторе)
```python
class MyModelSerializer(serializers.ModelSerializer):
    name = serializers.CharField(read_only=True)  # Поле объявлено явно.

    class Meta:
        model = MyModel
        fields = ['name', 'description']
        extra_kwargs = {
            'name': {'write_only': True},  # Настройки будут проигнорированы.
        }
```
#### Настройка сопоставления полей
`serializer_field_mapping` - это атрибут класса ModelSerializer, который представляет собой словарь, где ключи — это типы полей модели (models.Field), а значения — соответствующие классы полей сериализатора (serializers.Field). Используется для быстрого сопоставления и создания кастомных полей.
```python
from django.db import models
from rest_framework import serializers

serializer_field_mapping = {
    models.AutoField: serializers.IntegerField,
    models.CharField: serializers.CharField,
    models.TextField: serializers.CharField,
    models.IntegerField: serializers.IntegerField,
    models.FloatField: serializers.FloatField,
    models.BooleanField: serializers.BooleanField,
    models.DateField: serializers.DateField,
    models.DateTimeField: serializers.DateTimeField,
    models.ForeignKey: serializers.PrimaryKeyRelatedField,
    # и другие поля
}
```
создание кастомного поля:
```python
from rest_framework import serializers

class CustomCharField(serializers.CharField):
    def to_representation(self, value):
        return value.upper()  # Все значения переводятся в верхний регистр.
```
переопределение сопоставления:
```python
from rest_framework import serializers
from django.db import models

class MyModelSerializer(serializers.ModelSerializer):
    # Переопределяем маппинг, не изменяя оригинальный
    serializer_field_mapping = serializers.ModelSerializer.serializer_field_mapping.copy()
    serializer_field_mapping[models.CharField] = CustomCharField

    class Meta:
        model = MyModel
        fields = '__all__'
```
`serializer_related_field` - определяет какой тип поля сериализатора будет использован для отношений(ForeignKey, ManyToManyField, OneToOneField) по умолчанию.
для `ModelSerializer` используется `serializers.PrimaryKeyRelatedField` и означает, что связанные объекты будут представлены их первичным ключом.

For `HyperlinkedModelSerializer` this defaults to `serializers.HyperlinkedRelatedField` и означает что связанное поле будет представлено как гиперссылка на URL-адрес, связанный с этим объектом.

`serializer_url_field` - атрибут класса который используется для url-полей, его значение по умолчанию serializers.HyperlinkedIdentityField

`serializer_choice_field` - `Defaults to serializers.ChoiceField`

#### Классы и аргументы полей
В API Django REST Framework методы для генерации полей сериализатора автоматически определяют класс поля и аргументы для каждого поля модели. Они возвращают кортеж: (field_class, field_kwargs).

- `build_standard_field`(self, field_name, model_field)
Генерирует поле сериализатора для стандартного поля модели.
По умолчанию выбирает класс поля на основе атрибута `serializer_field_mapping`.
- `build_relational_field`(self, field_name, relation_info)
Создаёт поле для отношений (ForeignKey, ManyToManyField и т.д.).
Использует `serializer_related_field` для выбора класса поля.
Аргумент `relation_info` — именованный кортеж с информацией:
`model_field`: поле модели.
related_model: связанная модель.
to_many: булево значение (множественное отношение или нет).
has_through_model: булево значение (есть ли промежуточная модель, например, в ManyToManyField с through).
- `build_nested_field`(self, field_name, relation_info, nested_depth)
Генерирует поле для вложенного сериализатора, если установлена опция depth.
Динамически создаёт вложенный сериализатор (на основе `ModelSerializer` или `HyperlinkedModelSerializer`).
Аргумент `nested_depth`: текущая глубина вложенности (`depth` - 1).
Использует `relation_info`, аналогично `build_relational_field`.
- `build_property_field`(self, field_name, model_class)
Генерирует поле для свойства или метода модели без аргументов.
По умолчанию возвращает поле `ReadOnlyField`.
- `build_url_field`(self, field_name, model_class)
Создаёт поле для ссылки на сам объект (поле url).
По умолчанию возвращает `HyperlinkedIdentityField`.
- `build_unknown_field`(self, field_name, model_class)
Вызывается, если поле не связано с полем модели или свойством.
По умолчанию вызывает ошибку.
Можно переопределить, чтобы обрабатывать такие случаи.

[к содержанию](#содержание)

### 1.3. HyperlinkedModelSerializer
`HyperlinkedModelSerializer` похож на `ModelSerializer`, но для отношений использует гиперссылки, а не первичные ключи.
вместо поля с первичным ключом имеет поле с url по умолчанию:
поле url имеет тип `HyperlinkedIdentityField`, а отношения реализуются полем `HyperlinkedRelatedField`

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ['url', 'id', 'account_name', 'users', 'created']
        # так первичный ключ можно задать явно (id)
```
`HyperlinkedIdentityField` получает ссылку на сам объект(ссылку на конкретную книгу):
```python
class BookSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='book-detail')
    # view_name='book-detail' указывает на имя view, которое используется 
    # для отображения данных конкретной книги.
    
    class Meta:
        model = Book
        fields = ['url', 'title', 'author']

```
```json
{
    "url": "http://example.com/books/1/",
    "title": "The Great Gatsby",
    "author": "F. Scott Fitzgerald"
}
```
`HyperlinkedRelatedField` получает ссылку на связанный объект(ссылка на автора, связанного с книгой):
```python
class BookSerializer(serializers.HyperlinkedModelSerializer):
    author = serializers.HyperlinkedRelatedField(queryset=Author.objects.all(), view_name='author-detail')

    class Meta:
        model = Book
        fields = ['url', 'title', 'author']

```
```json
{
    "url": "http://example.com/books/1/",
    "title": "The Great Gatsby",
    "author": "http://example.com/authors/1/"
}
```

#### Абсолютные и относительные ссылки
При создании экземпляра класса `HyperlinkedModelSerializer` нужно передать контекстный словарь с запросом, чтобы гиперссылка была полным адресом url - содержала в себе имя хоста, а не была относительной ссылкой:
```python
serializer = AccountSerializer(queryset, context={'request': request})
# http://api.example.com/accounts/1/
# rather than: /accounts/1/
```
Если нужно использовать относительную ссылку, в контекстный словарь нужно передать `{'request': None}`

#### Как определяются представления для гиперссылок
Нужно определить какие представления использовать для гиперссылок на экземпляры модели. 
По умолчанию ожидается, что гиперссылка будет соответствовать стилю `{имя_модели}-detail` и искать экземпляр по первичному ключу.
Такое поведение можно изменить, переопределив параметры `view_name` и `lookup_field`:

объявив их неявно (используя `extra_kwargs`) :
```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ['account_url', 'account_name', 'users', 'created']
        extra_kwargs = {
            'url': {'view_name': 'accounts', 'lookup_field': 'account_name'},
            'users': {'lookup_field': 'username'}
        }
        # поле url будет генерировать гиперссылку на основе view_name='accounts' 
        # и значения поля account_name вместо дефолтного pk
        # http://api.example.com/accounts/my-account-name/
        # вместо: 
        # http://api.example.com/accounts/1/
```

или объявив явно:
```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='accounts',
        lookup_field='slug'
    )
    users = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        lookup_field='username',
        many=True,
        read_only=True
    )

    class Meta:
        model = Account
        fields = ['url', 'account_name', 'users', 'created']
```
`lookup_field` - атрибут, который обозначает какое поле модели использовать для поиска объекта, по-умолчанию pk.
`view_name` — это атрибут, с именем представления, которое будет использовано для формирования URL гиперссылки.
Чтобы проконтролировать правильную настройку имен представлений (view names) и полей для поиска (lookup fields) для гиперссылки, нужно использовать отладку с repr() для экземпляра `HyperlinkedModelSerializer`.

### 1.4. ListSerializer
`ListSerializer` позволяет сериализовать и валидировать несколько объектов одновременно. Обычно его не вызывают явно, но если установить `many=True` при создании экземпляра сериализатора, то будет вызван класс `ListSerializer` и создан его экземпляр.

еще аргументы `ListSerializer`:

`allow_empty` - разрешает пустой список, True by default

`max_length` - ограничивает максимальную длину списка, None by default

`min_length` - ограничивает минимальную длину списка, None by default

#### Настройка поведения `ListSerializer`
При множественном создании вызывается `create()` for each item in the list.
```python
# встроенный create():
def create(self, validated_data):
    return [self.child.create(attrs) for attrs in validated_data]
    # child.create() вызывает create() дочернего сериализатора.
    # для каждого элемента отдельный запрос в базу данных.
```
можно переопределить встроенный create():
```python
class BookListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        books = [Book(**item) for item in validated_data]
        return Book.objects.bulk_create(books)
        # массовое создание объектов с методом bulk_create():
        # используется всего один запрос к бд

class BookSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = BookListSerializer
```
[к содержанию](#содержание)

Множественное обновление не поддерживается классом `ListSerializer` by default.
Тк dfr не может сам решить что нужно создать, что обновить, что удалить. 
Это нужно указывать явно.

Для этого нужно явно объявить поле `id`, тк при неявном создании оно `read_only` и удаляется при обновлении. 

При явном объявлении поле `id` доступно в методе `update()`
```python
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        # Maps for id->instance and id->data item.
        book_mapping = {book.id: book for book in instance}  # объекты из бд
        data_mapping = {item['id']: item for item in validated_data}  # входящие данные

        # Perform creations and updates.
        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
                # если объекта нет в бд, он создается
            else:
                ret.append(self.child.update(book, data))
                # если есть, то обновляется

        # Perform deletions.
        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
            # если объекта базы данных нет во входящих данных, удаляем его из бд
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    # We need to identify elements in the list using their primary key,
    # so use a writable field here, rather than the default which would be read-only.
    id = serializers.IntegerField()
    ...

    class Meta:
        list_serializer_class = BookListSerializer
```

#### Настройка инициализации `ListSerializer` при неявном объявлении
нужно понять какие аргументы будут переданы в __init__() для существующего(дочернего) сериализатора и для будущего(родительского) `ListSerializer`.
По умолчанию передаются все аргументы кроме валидаторов и пользовательских именованных аргументов(они только для дочернего).

Можно это изменить и передать в родительский то, что нужно:
```python
@classmethod
    def many_init(cls, *args, **kwargs):
        # Instantiate the child serializer.
        kwargs['child'] = cls()
        # Instantiate the parent list serializer.
        return CustomListSerializer(*args, **kwargs)
```

### 1.5. BaseSerializer
`BaseSerializer` — базовый класс, который не предоставляет встроенной логики для работы с полями, валидацией или моделями. Требует явной реализации методов `to_representation` и `to_internal_value` для сериализации и десериализации данных. Имеет более низкий уровень абстракции чем `Serializer`. Используется для более кастомизированных случаев, когда остальные не подходят.

Методы которые нужно реализовать, тк по умолчанию вернут NotImplementedError:

`to_representation()` - реализовать метод, для сериализации (read only)
(преобразование объекта сериализации в читаемый формат).

`to_internal_value()` - реализовать метод, для десериализации (write)

`create()`, `update()` - реализовать методы, чтобы сохранять экземпляры модели.

#### Read-only сериализатор на основе BaseSerializer
для создания read-only сериализатора нужно реализовать метод `to_representation()`. 
Он отвечает за преобразование объекта модели в словарь данных, который будет возвращен клиенту.
```python
class HighScore(models.Model):
    # есть модель
    created = models.DateTimeField(auto_now_add=True)
    player_name = models.CharField(max_length=10)
    score = models.IntegerField()
```
```python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_representation(self, instance):
        # instance это экземпляр модели
        # преобразует объект модели в словарь и вернет его
        return {
            'score': instance.score,
            'player_name': instance.player_name
        }
```
представление для одного объекта модели:
```python
@api_view(['GET'])
def high_score(request, pk):
    instance = HighScore.objects.get(pk=pk)
    serializer = HighScoreSerializer(instance)
    return Response(serializer.data)
```

представление для всех объектов модели:
```python
@api_view(['GET'])
def all_high_scores(request):
    queryset = HighScore.objects.order_by('-score')
    serializer = HighScoreSerializer(queryset, many=True)
    return Response(serializer.data)
```
[к содержанию](#содержание)

#### Read-write BaseSerializer
Для создания read-write сериализатора нужно реализовать метод `to_internal_value()`.
Он преобразовывает данные, полученные из запроса, во внутренние данные, для сохранения в бд, создания экземпляров модели.
Метод `to_internal_value()` делает доступной валидацию с методами `is_valid()`, `validated_data`, `errors`. 

```python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_internal_value(self, data):
        score = data.get('score')
        player_name = data.get('player_name')

        # Perform the data validation.
        if not score:
            raise serializers.ValidationError({
                'score': 'This field is required.'
            })
        if not player_name:
            raise serializers.ValidationError({
                'player_name': 'This field is required.'
            })
        if len(player_name) > 10:
            raise serializers.ValidationError({
                'player_name': 'May not be more than 10 characters.'
            })

        # Return the validated values. This will be available as
        # the `.validated_data` property.
        return {
            'score': int(score),
            'player_name': player_name
        }

    def to_representation(self, instance):
        return {
            'score': instance.score,
            'player_name': instance.player_name
        }

    def create(self, validated_data):
        return HighScore.objects.create(**validated_data)
```

### 1.6. Advanced serializer usage

#### Переопределение поведения сериализации и десериализации
в классе Serializer определены методы `to_representation`(отвечает за сериализацию) и `to_internal_value`(отвечает за десериализацию), которые можно переопределить.

`to_representation(self, instance)` - принимает объект модели, возвращает питоновский тип данных(зависит от настроек рендеринга).

```python
def to_representation(self, instance):
    """Convert `username` to lowercase."""
    ret = super().to_representation(instance)
    ret['username'] = ret['username'].lower()
    return ret
```

`to_internal_value(self, data)` - принимает невалидированные данные из запроса, возвращает проверенные данные, которые доступны в `serializer.validated_data` и передаются в методы `create()` или `update()`, если вызван `save()`.

`data`является значением request.data, его тип данных зависит от настроек парсера.

#### Наследование сериализатора

Сериализаторы можно расширить и переиспользовать применяя наследование. Для этого у родительского класса нужно объявить общие поля и методы, которые понадобятся детям.
Мета-класс не наследуется неявно, чтобы он наследовался нужно сделать это явно(не рекомендуется):
```python
class AccountSerializer(MyBaseSerializer):
    class Meta(MyBaseSerializer.Meta):
    # наследоваться в Meta не рекомендуется
        model = Account
```
```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
    # лучше не наследоваться, а объявлять явно параметры Meta для каждого сериализатора:
        model = Book
        fields = ['title', 'author', 'published_date', 'isbn']
        read_only_fields = ['author']
```

Можно явно удалить поле, указав значение None, если оно было явно определено в родительском классе.
Если же поле было получено из модели, оно не удалится - нужно использовать [Выбор полей](#выбор-полей).
```python
class MyBaseSerializer(ModelSerializer):
    my_field = serializers.CharField()

class MySerializer(MyBaseSerializer):
    my_field = None
```

#### Динамическое изменение полей
Получение доступа к полям объекта сериализатора, используя аттрибут `.fields` - он позволяет менять поля во время выполнения.

Создадим сериализатор, позволяющий задать поля, которые будут созданы при его инициализации:
```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes an additional `fields` argument that
    controls which fields should be displayed.
    """

    def __init__(self, *args, **kwargs):
        fields = kwargs.pop('fields', None)
        # сохраняем поля полученные из UserSerializer(user, fields=('id', 'email'))
        # в fields и удаляем их из kwargs,
        # чтобы не передавать в родительский класс (ModelSerializer),
        # который не принимает такие аргументы.

        super().__init__(*args, **kwargs)
        # для инициалилзации используем конструктор родителя (ModelSerializer)
        # с полями определенными в Meta UserSerializer

        if fields is not None:
            # Drop any fields that are not specified in the `fields` argument.
            allowed = set(fields)  # поля из UserSerializer(user, fields=('id', 'email'))
            existing = set(self.fields) # поля из Meta UserSerializer
            for field_name in existing - allowed:
                self.fields.pop(field_name)  # удалили все поля кроме переданных
```

Созданный сериализатор позволит выполнить следующее:

```python
 class UserSerializer(DynamicFieldsModelSerializer):
    # создадим новый класс, наследуясь от динамического
    # тк нету своего __init__, возьмет от динамического
     class Meta:
        model = User
        fields = ['id', 'username', 'email']  # existing

print(UserSerializer(user, fields=('id', 'email')))
# {'id': 2, 'email': 'jon@example.com'}
# тк поля переданы, то все остальные были удалены

print(UserSerializer(user))
# {'id': 2, 'username': 'jonwatts', 'email': 'jon@example.com'}
# тк поля не переданы, вернулись существующие
```

Такой способ позволяет динамически включать или исключать поля в сериализаторе, исходя из переданного списка fields. Если fields передан в конструктор, то только те поля, которые указаны в этом списке, будут сериализованы. Все остальные поля будут удалены из self.fields и не попадут в сериализованный результат.












### 2. **Как работают ссылки**
- Заголовки в Markdown автоматически превращаются в якоря, к которым можно переходить с помощью ссылок.
- Ссылка вида `[Сериализация (Serialization)](#сериализация-serialization)` перемещает пользователя к разделу `## Сериализация (Serialization)`.

---

### 3. **Добавление ссылки "Назад к содержанию"**
Каждый раздел заканчивается ссылкой `[К содержанию](#содержание)`, чтобы пользователь мог быстро вернуться к началу.

---

### 4. **Предпросмотр файла**
- Откройте файл `README.md` в PyCharm.
- Нажмите на значок глазика в правом верхнем углу, чтобы включить предпросмотр Markdown.

---




