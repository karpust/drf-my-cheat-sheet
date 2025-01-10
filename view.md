# Class-based Views

## Содержание

- [APIView](#apiview)
  - [Представления на основе классов](#представления-на-основе-классов)
    - [Основные отличия APIView от обычных View](#основные-отличия-apiview-от-обычных-view)
    - [Пример использования APIView](#пример-использования-apiview)
    - [Атрибуты политики API](#атрибуты-политики-api)
    - [Методы инициализации политики API](#методы-инициализации-политики-api)
    - [Методы реализации политики API](#методы-реализации-политики-api)
    - [Методы диспетчеризации](#методы-диспетчеризации)
  - [Представления на основе функций](#представления-на-основе-функций)
    - [@api_view()](#api_view)
    - [Декораторы для настройки политики API](#декораторы-для-настройки-политики-api)
    - [Декоратор схемы представления](#декоратор-схемы-представления)

---

# APIView

# Представления на основе классов

## Основные отличия APIView от обычных View

Классы APIView отличаются от обычных View следующими способами:

- Запросы, передаваемые методам-обработчикам, будут экземплярами Request из REST framework, а не HttpRequest из Django.
- Методы-обработчики могут возвращать экземпляры Response из REST framework вместо HttpResponse из Django. Представление будет управлять согласованием содержимого и установкой правильного рендерера в ответе.
- Любые исключения APIException будут перехвачены и преобразованы в соответствующие ответы.
- Входящие запросы будут аутентифицированы, и перед передачей запроса методу-обработчику будут выполнены проверки разрешений и/или ограничения скорости.

## Пример использования APIView

Использование класса APIView практически не отличается от использования обычных классов View. Как обычно, входящий запрос направляется соответствующему методу-обработчику, такому как `.get()` или `.post()`. Кроме того, в классе можно задать ряд атрибутов, управляющих различными аспектами политики API.

Пример:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import authentication, permissions
from django.contrib.auth.models import User

class ListUsers(APIView):
    """
    Представление для отображения всех пользователей в системе.

    * Требуется токен-аутентификация.
    * Только администраторы могут получить доступ к этому представлению.
    """
    authentication_classes = [authentication.TokenAuthentication]
    permission_classes = [permissions.IsAdminUser]

    def get(self, request, format=None):
        """
        Возвращает список всех пользователей.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

> Примечание: Полные методы, атрибуты и связи между APIView, GenericAPIView, различными Mixin и Viewset могут быть поначалу сложными. Помимо документации здесь, ресурс Classy Django REST Framework предоставляет удобный справочник с методами и атрибутами для каждого из классов представлений Django REST framework.

## Атрибуты политики API

Следующие атрибуты управляют подключаемыми аспектами представлений API:

- `.renderer_classes`
- `.parser_classes`
- `.authentication_classes`
- `.throttle_classes`
- `.permission_classes`
- `.content_negotiation_class`

## Методы инициализации политики API

Следующие методы используются REST framework для инициализации различных подключаемых политик API. Обычно их не требуется переопределять.

- `.get_renderers(self)`
- `.get_parsers(self)`
- `.get_authenticators(self)`
- `.get_throttles(self)`
- `.get_permissions(self)`
- `.get_content_negotiator(self)`
- `.get_exception_handler(self)`

## Методы реализации политики API

Следующие методы вызываются перед передачей запроса методу-обработчику:

- `.check_permissions(self, request)`
- `.check_throttles(self, request)`
- `.perform_content_negotiation(self, request, force=False)`

## Методы диспетчеризации

Следующие методы вызываются непосредственно методом `.dispatch()` представления. Они выполняют любые действия, которые необходимо выполнить до или после вызова методов-обработчиков, таких как `.get()`, `.post()`, `.put()`, `.patch()` и `.delete()`.

- `.initial(self, request, *args, **kwargs)`

  Выполняет любые действия, необходимые до вызова метода-обработчика. Этот метод используется для обеспечения разрешений и ограничения скорости, а также для согласования содержимого.

  Обычно этот метод не требуется переопределять.

- `.handle_exception(self, exc)`

  Любое исключение, вызванное методом-обработчиком, будет передано этому методу, который либо возвращает экземпляр Response, либо повторно генерирует исключение.

  Реализация по умолчанию обрабатывает любые подклассы `rest_framework.exceptions.APIException`, а также исключения `Http404` и `PermissionDenied` из Django и возвращает соответствующий ответ об ошибке.

  Если вам нужно настроить ответы об ошибках, возвращаемые вашим API, вы должны переопределить этот метод.

- `.initialize_request(self, request, *args, **kwargs)`

  Обеспечивает, чтобы объект запроса, передаваемый методу-обработчику, был экземпляром Request, а не обычным HttpRequest из Django.

  Обычно этот метод не требуется переопределять.

- `.finalize_response(self, request, response, *args, **kwargs)`

  Обеспечивает, чтобы любой объект Response, возвращаемый методом-обработчиком, был преобразован в правильный тип содержимого в соответствии с согласованием содержимого.

  Обычно этот метод не требуется переопределять.




# Представления на основе функций

> Утверждение, [что представления на основе классов] всегда являются лучшим решением, — это ошибка.
>
> — Ник Коглан

REST framework также позволяет работать с обычными представлениями на основе функций. Он предоставляет набор простых декораторов, которые оборачивают ваши представления на основе функций, чтобы они получали экземпляр `Request` (вместо обычного Django `HttpRequest`) и могли возвращать `Response` (вместо Django `HttpResponse`). Кроме того, он позволяет настраивать, как будет обрабатываться запрос.

## @api_view()

**Сигнатура:** `@api_view(http_method_names=['GET'])`

Основой этой функциональности является декоратор `api_view`, который принимает список HTTP-методов, на которые должно реагировать ваше представление. Например, вот как можно написать очень простое представление, которое вручную возвращает некоторые данные:

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view()
def hello_world(request):
    return Response({"message": "Привет, мир!"})
```

Это представление будет использовать рендеры, парсеры, классы аутентификации и другие компоненты, указанные в настройках по умолчанию.

По умолчанию будут приниматься только методы `GET`. На другие методы будет возвращаться ответ "405 Method Not Allowed". Чтобы изменить это поведение, укажите, какие методы допускаются, следующим образом:

```python
@api_view(['GET', 'POST'])
def hello_world(request):
    if request.method == 'POST':
        return Response({"message": "Получены данные!", "data": request.data})
    return Response({"message": "Привет, мир!"})
```

## Декораторы для настройки политики API

Чтобы переопределить настройки по умолчанию, REST framework предоставляет набор дополнительных декораторов, которые можно добавлять к вашим представлениям. Эти декораторы должны следовать после (ниже) декоратора `@api_view`. Например, чтобы создать представление, которое использует ограничение на количество запросов (throttle), разрешая вызывать его только один раз в день для конкретного пользователя, используйте декоратор `@throttle_classes`, передав список классов ограничений:

```python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
    rate = '1/day'

@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message": "Привет на сегодня! До завтра!"})
```

Эти декораторы соответствуют атрибутам, которые устанавливаются в подклассах `APIView`, описанных выше.

Доступные декораторы:

- `@renderer_classes(...)`
- `@parser_classes(...)`
- `@authentication_classes(...)`
- `@throttle_classes(...)`
- `@permission_classes(...)`

Каждый из этих декораторов принимает один аргумент, который должен быть списком или кортежем классов.

## Декоратор схемы представления

Чтобы переопределить генерацию схемы по умолчанию для представлений на основе функций, можно использовать декоратор `@schema`. Этот декоратор должен следовать после (ниже) декоратора `@api_view`. Например:

```python
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # переопределение логики интроспекции представления...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Привет на сегодня! До завтра!"})
```

Этот декоратор принимает один экземпляр `AutoSchema`, экземпляр подкласса `AutoSchema` или экземпляр `ManualSchema`, как описано в документации по схемам. Можно передать `None`, чтобы исключить представление из генерации схемы:

```python
@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Не появится в схеме!"})
```

