# Конспект по Django
## 1. Начало проекта

Первым делом нужно настроить соединение с базами данных

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'books_db',
        'USER': 'books_user',
        'PASSWORD': 'books_user',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```
Далее создайте модели в `models.py`
```python
from django.db import models


class Book(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=7,decimal_places=2)
  
```
Добавьте ваше приложение в ```INSTALLED_APPS``` в ```settings.py```
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'store',
]
```
Создаём миграции
```bash
python3 manage.py makemigrations
```
Мигрируем
```bash
python3 manage.py migrate
```
Установите djangorestframework
```bash
pip install djangorestframework
```
Теперь создайте `serializers.py`
```python
from rest_framework.serializers import ModelSerializer

from store.models import Book


class BooksSerializer(ModelSerializer):
    class Meta:
        model = Book
        fields = "__all__"

```
Создайте вид в `views.py`
```python
from rest_framework.viewsets import ModelViewSet

from store.models import Book
from store.serializers import BooksSerializer


class BookViewSet(ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BooksSerializer
```
Зарегистрируйте новый роутер в `urls.py`
```python
from django.contrib import admin
from django.urls import path
from rest_framework.routers import SimpleRouter

from store.views import BookViewSet

router = SimpleRouter()
router.register(r'book', BookViewSet)

urlpatterns = [
    path('admin/', admin.site.urls),
]

urlpatterns += router.urls
```

## 2. Unit tests
Удалите `tests.py` в вашем приложении и создайте пакет ```tests```.
Для примера создадим файл `logic.py` в вашем приложении.
Также создайте `test_logic.py` в `<your_app>/tests/`.
Если возникают ошибки, настройте ваш тестовый интерпретатор следующим образом

![img.png](img.png)

`logic.py`
```python
def operations(a, b, c):
    if b == '+':
        return a + c
    if b == '-':
        return a - c
    if b == '*':
        return a * c
    if b == '/':
        return a / c

```
`test_logic.py`
```python
from django.test import TestCase

from store.logic import operations


class LogicTestCase(TestCase):
    def test_plus(self):
        result = operations(6, '+', 13)
        self.assertEqual(19, result)

    def test_minus(self):
        result = operations(6, '-', 13)
        self.assertEqual(-7, result)

    def test_multiply(self):
        result = operations(6, '*', 13)
        self.assertEqual(78, result)

```
Получаем:

![img_1.png](img_1.png)

Для возможности подключиться к СУБД от созданного пользователя, необходимо проверить настройки прав в конфигурационном файле pg_hba.conf.

Для начала смотрим путь расположения данных для PostgreSQL:

```=# SHOW config_file;```

В ответ мы получим, что-то на подобие:

```----------------------------------------- 
/var/lib/pgsql/13/data/postgresql.conf
(1 row)
```

* в данном примере /var/lib/pgsql/13/data/ — путь расположения конфигурационных файлов.

Открываем ```pg_hba.conf```:

```bash
sudo nano /var/lib/pgsql/9.6/data/pg_hba.conf
```

Добавляем права на подключение нашему созданному пользователю:


```
# IPv4 local connections:
host    all             dmosk           127.0.0.1/32            md5
```

* в данном примере мы разрешили подключаться пользователю dmosk ко всем базам на сервере (all) от узла 127.0.0.1 (localhost) с требованием пароля (md5).
* необходимо, чтобы данная строка была выше строки, которая прописана по умолчанию
host    all             all             127.0.0.1/32            ident.

После перезапускаем службу:

```bash
systemctl restart postgresql
```
Запускаем тесты
```bash
./manage.py test .
```
И видим, что тесты выполнены успешно
```
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
...
----------------------------------------------------------------------
Ran 3 tests in 0.008s

OK
Destroying test database for alias 'default'...
```
Создадим файл `test_api.py` с тестами для нашего проекта.
```python
from django.urls import reverse
from rest_framework import status

from store.models import Book
from rest_framework.test import APITestCase

from store.serializers import BooksSerializer


class BooksApiTestCase(APITestCase):
    def test_get(self):
        book_1 = Book.objects.create(name='Test book 1', price=25)
        book_2 = Book.objects.create(name='Test book 2', price=140)
        url = reverse('book-list')
        response = self.client.get(url)
        serializer_data = BooksSerializer([book_1, book_2], many=True).data
        self.assertEqual(status.HTTP_200_OK, response.status_code)
        self.assertEqual(serializer_data, response.data)

```
Если что-то поменяется в сериализаторе, то тест будет работать.
По сути мы сравниваем его с самим собой.
Создадим тест конкретно для сериализатора в `test_serializers.py`
```python
from django.test import TestCase

from store.models import Book
from store.serializers import BooksSerializer


class BooksSerializerTestCase(TestCase):
    def test_ok(self):
        book_1 = Book.objects.create(name='Test book 1', price=25)
        book_2 = Book.objects.create(name='Test book 2', price=140)
        data = BooksSerializer([book_1, book_2], many=True).data
        expected_data = [
            {
                'id': book_1.id,
                'name': 'Test book 1',
                'price': '25.00'
            },
            {
                'id': book_2.id,
                'name': 'Test book 2',
                'price': '140.00'
            },
        ]
        self.assertEqual(expected_data, data)
```
Очевидно, что многое повторяется и нужно что-то оптимизировать.
Но это не обязательно! Тесты должны быть простыми и копипаста тут помогает.
Установим `coverage` в виртуальное окружение.

Выполним
```bash
coverage run --source='.' ./manage.py test . 
```
Как видим, все тесты выполнены успешно.
Все 5 точек
```bash
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.....
----------------------------------------------------------------------
Ran 5 tests in 0.103s

OK
Destroying test database for alias 'default'...
```
Посмотрим детальную информацию по покрытию тестами наших фалов
```bash
coverage report
```
```bash
Name                               Stmts   Miss  Cover
------------------------------------------------------
books/__init__.py                      0      0   100%
books/asgi.py                          4      4     0%
books/settings.py                     19      0   100%
books/urls.py                          8      0   100%
books/wsgi.py                          4      4     0%
manage.py                             12      2    83%
store/__init__.py                      0      0   100%
store/admin.py                         1      0   100%
store/apps.py                          4      0   100%
store/logic.py                         9      2    78%
store/migrations/0001_initial.py       5      0   100%
store/migrations/__init__.py           0      0   100%
store/models.py                        4      0   100%
store/serializers.py                   6      0   100%
store/tests/__init__.py                0      0   100%
store/tests/test_api.py               14      0   100%
store/tests/test_logic.py             12      0   100%
store/tests/test_serializers.py       10      0   100%
store/views.py                         6      0   100%
------------------------------------------------------
TOTAL                                118     12    90%
```
Мы можем получить визуальный отчёт
```bash
coverage html
```
Создалась папка `htmlcov` в которой нужно открыть в браузере `index.html`

![img_2.png](img_2.png)

Посмотрим детально тест на `logic.py`

![img_3.png](img_3.png)

* Зелёные линии - участок кода куда заходил тест
* Красные линии - не протестированный участок когда

