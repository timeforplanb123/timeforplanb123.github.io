---
title: Примеры использования декораторов в Python
classes: wide
excerpt: "Примеры использования декораторов в Python."
categories:
  - code
  - notes
tags:
  - python
  - docs
  - code
toc: true
toc_label: "Getting Started"
---
## Полезные функции и методы

Начну с полезных функций и методов, которые будут использованы дальше, в примерах декораторов.

- `func.__name__` - получить имя функции с помощью специальной переменной `__name__`. Аналогично можно узнать имя класса - `cls.__name__`
- `getattr(cls, 'attribute_name', None)` - проверить наличие атрибута (метод, переменная класса или экземпляра класса). Возвращает атрибут(объект). `None` в конце игнорирует ошибки, если атрибута нет.

  Например, проверим, есть ли у класса `cls` метод `dispatch`:
`dispatch = getattr(cls, 'dispatch', None)`
- `hasattr(some_object, "__iter__")` - проверить, есть ли у объекта атрибут (переменная или метод):

  ```python
  In [15]: hasattr(some_object, "__iter__")
  False
  ```

  `getattr` возвращает сам объект, а `hasattr` возвращает `True` или `False`.

- `setattr(cls, 'attribute_name', attribute)` - присвоить атрибут классу `cls`. Если `attribute` - функция, то так можно присвоить метод классу `cls`. А так можно декорировать этот метод - `setattr(cls, 'attribute_name', verbose(attribute))`, где `verbose` - декоратор.

  Функция - это объект. С помощью замыкания и `setattr` можно добавить атрибуты:

  ```python
  def add_mark(**kwargs):
	  def decorator(func):
    	  for key, value in kwargs.items():
        	  setattr(func, key, value)
    	  return func
	  return decorator

  @add_mark(test=True, ordered=True)
  def test_function(a, b):
      return a + b


  In [73]: test_function.ordered
  Out[73]: True

  In [74]: test_function.test
  Out[74]: True
  ```

- `hasattr(object, '__call__')` или `callable(object)` - проверить, является ли объект вызываемым (например, методом класса).
  Еще можно сравнить тип объекта с уже существующим - `isinstance(object, type(self.__init__))`

  `callable_attributes = {k:v for k, v in cls.__dict__.items() if callable(v)}` - получить словарь с вызываемыми объектами класса cls с помощью генератора словаря

- `IPAddress.__dict__.items()` - получить атрибуты класса `IPAddress`

  ```python
  In [14]: IPAddress.__dict__.items()
  Out[14]: dict_items([('__module__', '__main__'), ('__init__', <function IPAddress.__init__ at 0x7f04c76abdc0>), ('__repr__', <function IPAddress.__repr__ at 0x7f04c76ab670>), ('__eq__', <function IPAddress.__eq__ at 0x7f04c76e1f70>), ('__lt__', <function IPAddress.__lt__ at 0x7f04c76e15e0>), ('__dict__', <attribute '__dict__' of 'IPAddress' objects>), ('__weakref__', <attribute '__weakref__' of 'IPAddress' objects>), ('__doc__', None), ('__hash__', None), ('__getattribute__', <function decorator.<locals>._newgetattr at 0x7f04c76ab820>)])
  ```
  или тоже самое другим способом:

  ```python
  In [16]: vars(IPAddress).items()
  Out[16]: dict_items([('__module__', '__main__'), ('__init__', <function IPAddress.__init__ at 0x7f04c76abdc0>), ('__repr__', <function IPAddress.__repr__ at 0x7f04c76ab670>), ('__eq__', <function IPAddress.__eq__ at 0x7f04c76e1f70>), ('__lt__', <function IPAddress.__lt__ at 0x7f04c76e15e0>), ('__dict__', <attribute '__dict__' of 'IPAddress' objects>), ('__weakref__', <attribute '__weakref__' of 'IPAddress' objects>), ('__doc__', None), ('__hash__', None), ('__getattribute__', <function decorator.<locals>._newgetattr at 0x7f04c76ab820>)])
  ```
  Справедливо и для экземпляров класса:
  ```python
  vars(object).items()
  ```
  А так можно получить атрибуты класса, зная только экземпляр:
  ```python
  In [15]: vars(obj.__class__)
  Out[15]: 
  mappingproxy({'__module__': '__main__',
                '__init__': <function __main__.IPAddress.__init__(self, ip)>,
                '__repr__': <function __main__.IPAddress.__repr__(self)>,
                '__eq__': <function __main__.IPAddress.__eq__(self, other)>,
                '__lt__': <function __main__.IPAddress.__lt__(self, other)>,
                '__dict__': <attribute '__dict__' of 'IPAddress' objects>,
                '__weakref__': <attribute '__weakref__' of 'IPAddress' objects>,
                '__doc__': None,
                '__hash__': None,
                '__gt__': <function functools._gt_from_lt(self, other, NotImplemented=NotImplemented)>,
                '__le__': <function functools._le_from_lt(self, other, NotImplemented=NotImplemented)>,
                '__ge__': <function functools._ge_from_lt(self, other, NotImplemented=NotImplemented)>})
  ```

  Пример:
  ```python
  In [17]: import ipaddress
      ...: from functools import total_ordering
      ...: 
      ...: 
      ...: def total_order(cls):
      ...:     return total_ordering(cls)
      ...: 
      ...: 
      ...: @total_order
      ...: class IPAddress:
      ...:     def __init__(self, ip):
      ...:         self._ip = int(ipaddress.ip_address(ip))
      ...: 
      ...:     def __repr__(self):
      ...:         return f"IPAddress('{self._ip}')"
      ...: 
      ...:     def __eq__(self, other):
      ...:         return self._ip == other._ip
      ...: 
      ...:     def __lt__(self, other):
      ...:         return self._ip < other._ip
      ...: 

  In [18]: obj = IPAddress("10.1.1.1")
  
  In [19]: obj.__dict__
  Out[19]: {'_ip': 167837953}
  
  In [20]: obj.__class__.__dict__
  Out[20]: 
  mappingproxy({'__module__': '__main__',
                '__init__': <function __main__.IPAddress.__init__(self, ip)>,
                '__repr__': <function __main__.IPAddress.__repr__(self)>,
                '__eq__': <function __main__.IPAddress.__eq__(self, other)>,
                '__lt__': <function __main__.IPAddress.__lt__(self, other)>,
                '__dict__': <attribute '__dict__' of 'IPAddress' objects>,
                '__weakref__': <attribute '__weakref__' of 'IPAddress' objects>,
                '__doc__': None,
                '__hash__': None,
                '__gt__': <function functools._gt_from_lt(self, other, NotImplemented=NotImplemented)>,
                '__le__': <function functools._le_from_lt(self, other, NotImplemented=NotImplemented)>,
                '__ge__': <function functools._ge_from_lt(self, other, NotImplemented=NotImplemented)>})

  In [21]: vars(obj)
  Out[21]: {'_ip': 167837953}

  In [22]: vars(obj.__class__)
  Out[22]: 
  mappingproxy({'__module__': '__main__',
                '__init__': <function __main__.IPAddress.__init__(self, ip)>,
                '__repr__': <function __main__.IPAddress.__repr__(self)>,
                '__eq__': <function __main__.IPAddress.__eq__(self, other)>,
                '__lt__': <function __main__.IPAddress.__lt__(self, other)>,
                '__dict__': <attribute '__dict__' of 'IPAddress' objects>,
                '__weakref__': <attribute '__weakref__' of 'IPAddress' objects>,
                '__doc__': None,
                '__hash__': None,
                '__gt__': <function functools._gt_from_lt(self, other, NotImplemented=NotImplemented)>,
                '__le__': <function functools._le_from_lt(self, other, NotImplemented=NotImplemented)>,
                '__ge__': <function functools._ge_from_lt(self, other, NotImplemented=NotImplemented)>})
  ```

- `frozenset`- получить `key`, `value` из словаря. Например:

  ```python
  In [27]: d = {'a': 123,
      ...: 'b': 1234,
      ...: }
  
  In [30]: frozenset(d.items())
  Out[30]: frozenset({('a', 123), ('b', 1234)})
  
  In [33]: k,v = frozenset(d.items())
  
  In [34]: k
  Out[34]: ('a', 123)
  
  In [35]: v
  Out[35]: ('b', 1234)
  ```

## Примеры декораторов

### Пример 1

Если необходимо обработать аргументы целевой функции (в примере ниже - это `upper`) декоратором (это - `verbose`), следует использовать конструкцию `args, kwargs` во внутренней функции декоратора(это - `wrapper`). Например:

```python
def verbose(func):
    def wrapper(*args, **kwargs):
        print(f'Вызываю функцию {func.__name__}')
        return func(*args, **kwargs)
    return wrapper

@verbose
def upper(string):
    return string.upper()

upper('line')
Вызываю функцию upper
'LINE'
```

Если аргументов целевой функции недостаточно, используем декоратор с аргументами(в примере ниже - это `restrict_args_type`). Тогда в замыкание добавляется еще один уровень вложенности. Например:

```python
def restrict_args_type(required_type):
    def decorator(func):
        @wraps(func)
        def wrapper(*args):
            if not all(isinstance(arg, required_type) for arg in args):
                raise ValueError(f'Все аргументы должны быть {required_type.__name__}')
            return func(*args)
        return wrapper
    return decorator

@restrict_args_type(str)
    def to_upper(*args):
         result = [s.upper() for s in args]
         return result

to_upper('a', 'a')
['A', 'A']
```
 
Для работы с аргументами декоратора есть удобный декоратор из модуля functools. Можно делать так:

```python
from functools import partial

def info(func, arg1, arg2):
    print('Decorator arg1 = ' + str(arg1))
    print('Decorator arg2 = ' + str(arg2))
    def wrapper(*args, **kwargs):
        print('Function {} args: {} kwargs: {}'.format(function.__name__, str(args), str(kwargs)))
        return function(*args, **kwargs)
    return wrapper

decorator_with_arguments = partial(info, arg1=3, arg2='Py')

@decorator_with_arguments
def doubler(number):
    return number * 2
    print(doubler(5))
```

### Пример 2

Если не нужно обрабатывать аргументы целевой функции (в примере ниже - `upper`), можно обойтись декоратором без внутренней функции.
Например, очевидное-невероятное, бесполезный декоратор, не делающий ничего(или, например, возвращающий имя функции с помощью `func.__name__`):

```python
In [1]: def decorator(func):
   ...:     return func
   ...:

In [2]: @decorator
   ...: def upper(string):
   ...:     return string.upper()
   ...:

In [3]: upper('line')
Out[3]: 'LINE'
```

Для примера выше можно использовать, хотя бы, декоратор с аргументами, чтобы как-то применить их в нашей целевой функции (`upper`). Например:

```python
def add_mark(**kwargs):
    def decorator(func):
        for key, value in kwargs.items():
            setattr(func, key, value)
        return func
    return decorator


@add_mark(test=True, ordered=True)
def test_function(a, b):
    return a + b


In [73]: test_function.ordered
Out[73]: True

In [74]: test_function.test
Out[74]: True
```

Или такой пример (декоратор не влияет на аргументы целевой функции, но аргумент декоратора формирует словарь):

```python
url_function_map = {}

def register(route):
    def decorator(func):
        url_function_map[route] = func
        return func
    return decorator

@register('/')
def func(a,b):
    return a+b

@register('/scripts')
def func2(a,b):
    return a+b


In [3]: url_function_map
Out[3]: {'/': <function __main__.func>, '/scripts': <function __main__.func2>}
```

### Пример 3

Внутренняя функция декоратора не обязана возвращать `func(*args, **kwargs)`. Такая запись вызывает функцию.

Иногда необходимо и достаточно вызвать функцию внутри самого декоратора и вернуть полученный результат. Например:

``` python
from netmiko import (ConnectHandler, NetMikoAuthenticationException,
                     NetMikoTimeoutException)


def retry(times):
    def decorator(func):
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            if not result:
                for i in range(times):
                    result = func(*args, **kwargs)
                    if result:
                        return result
            return result
            # если вернуть func(*args, **kwargs), то функция
            # выполнится повторно (первый вызов result = func(*args, **kwargs)
        return wrapper
    return decorator

device_params = {
    'device_type': 'cisco_ios',
    'ip': '192.168.100.1',
    'username': 'cisco',
    'password': 'cisco',
    'secret': 'cisco'
}

@retry(times = 3)
def send_show_command(device, show_command):
    print('Подключаюсь к', device['ip'])
    try:
        with ConnectHandler(**device) as ssh:
            ssh.enable()
            result = ssh.send_command(show_command)
        return result
    except (NetMikoAuthenticationException, NetMikoTimeoutException):
        return None


if __name__ == "__main__":
    output = send_show_command(device_params, 'sh clock')
```

Как правило, декоратор выполняет какое-то действие перед вызовом целевой функции через `return func(*args, **kwargs)`. Если нам нужно, например, с помощью декоратора, посчитать время целевой (декорируемой) функции, тогда нужно вызвать эту функцию в самом декораторе и вернуть результат. Например:

```python
from netmiko import ConnectHandler
from datetime import datetime
import time

def timecode(func):
    start_time = datetime.now()
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        finish_time = datetime.now() - start_time
        print(f"Функция выполнялась: {finish_time}")
        return result


device_params = {
    'device_type': 'cisco_ios',
    'ip': '192.168.100.1',
    'username': 'cisco',
    'password': 'cisco',
    'secret': 'cisco'
}

@timecode
def send_show_command(params, command):
    with ConnectHandler(**params) as ssh:
        ssh.enable()
        result = ssh.send_command(command)
    return result


if __name__ == "__main__":
    print(send_show_command(device_params, 'sh clock'))
```

### Пример 4. Декораторы классов

Если необходимо декорировать каждый вызываемый метод класса (или не каждый, а конкретные), то можно сделать так - [тык](https://github.com/natenka/advpyneng-examples-exercises/blob/main/examples/08_decorators/03_decorating_classes/class_decorator_verbose.py){:target="_blank"}

Немного про используемые в примере функции:
- `if callable(value) and name not in ('__repr__', '__str__')` - функция `callable(value)` возвращает `True` или `False` в зависимости от того, является ли объект `value `вызываемым, т.е. методом класса.
  Альтернативой функции `callable` является такая конструкция - `if hasattr(object, '__call__')`.
  С помощью `name not in ('__repr__', '__str__')` мы исключаем из декорируемых методов `__repr__` и `__str__`.

- `setattr(cls, name, verbose(value))` - возвращаем уже существующий метод, но декорированный декоратором `verbose`.
  При этом, если нужно передать декоратору аргументы, тогда выглядеть будет так - `setattr(cls, name, verbose(arg1, arg2)(value))`

Хороший пример, выполняющий тоже самое (декорирование вызываемых атрибутов класса), что и пример по ссылке выше:

*если где-то есть декоратор функции `track_exceptions_decorator`, тогда:*

```python
def track_exception(cls):
    # Get all callable attributes of the class
    callable_attributes = {k:v for k, v in cls.__dict__.items() 
                           if callable(v)}
    # Decorate each callable attribute of to the input class
    for name, func in callable_attributes.items():
        decorated = track_exceptions_decorator(func)
        setattr(cls, name, decorated)
    return cls

@track_exceptions
class A:
    def f1(self):
        print('1')

    def f2(self):
        print('2')
```

Альтернативным способом декорировать методы класса может быть использование `__getattribute__` в классе-декораторе.
Специальный метод `__getattribute__` вызывается каждый раз, когда мы обращаемся к любому атрибуту класса(переменной экземпляра класса, методу).

Например так вызывается метод:
```python
cls.__getattribute__(object, '__ge__')
```
или
```python
object.__getattribute__('__ge__')
```

Не самый удачный пример такого класса-декоратора (полностью заменяет декорируемый класс) - [тык](https://tirinox.ru/class-decorator/){:target="_blank"}

Создать новые методы в классе с помощью декораторов можно, например, так

```python
def create_init(cls):
	...

def create_repr(cls):
	...

def create_dataclass(cls):
    print('создаем dataclass')
    cls.__init__ = create_init(cls)
    cls.__repr__ = create_repr(cls)
    return cls

@create_dataclass
class Book:
	pass

```
Пример - [тык](https://github.com/natenka/advpyneng-examples-exercises/blob/main/examples/08_decorators/03_decorating_classes/class_decorator_dataclass.py){:target="_blank"}

### Пример 5. Модуль functools

Определенно стоит изучить возможности модуля `functools`. В нем набор готовых декораторов.
Документация на модуль - [тык](https://docs.python.org/3.11/library/functools.html){:target="_blank"}.

Например, вы можете реализовать в своем классе один из методов `__lt__()`, `__le__()`, `__gt__()`, `__ge__()`, добавить `__eq__`, а все недостающие методы за вас реализует декоратор `total_ordering`:

```python
import ipaddress
from functools import total_ordering


def total_order(cls):
    return total_ordering(cls)


@total_order
class IPAddress:
    def __init__(self, ip):
        self._ip = int(ipaddress.ip_address(ip))

    def __repr__(self):
        return f"IPAddress('{self._ip}')"

    def __eq__(self, other):
        return self._ip == other._ip

    def __lt__(self, other):
		return self._ip < other._ip
```

Тоже самое, но вручную:

```python
import ipaddress

def __le__(self, other):
    return list(vars(self).values()) <= list(vars(other).values())

def __ge__(self, other):
    return list(vars(self).values()) >= list(vars(other).values())

def total_order(cls):
    if not "__eq__" or not "__lt__" in vars(cls).keys():
        raise ValueError
    magic_methods = {"__le__": __le__,
                     "__ge__": __ge__,
                    }
    for k, v in magic_methods.items():
        if k not in vars(cls).keys():
            setattr(cls, k, v)
    return cls

@total_order
class IPAddress:
    def __init__(self, ip):
        self._ip = int(ipaddress.ip_address(ip))

    def __repr__(self):
        return f"IPAddress('{self._ip}')"

    def __eq__(self, other):
        return self._ip == other._ip

    def __lt__(self, other):
        return self._ip < other._ip
```

Т.к. при использовании декораторов, информация исходной функции заменяется внутренней функцией декоратора в том же модуле `functools` есть декоратор `wraps`, который исправляет ситуацию. Теперь не нужно вручную заполнять специальные переменные `__name__`, `__doc__`, `__module__`, например, так:

```python
def decorator(func):
	def decorated_func(*args, **kwargs):
		...
		return func(*args, **kwargs)
	decorated_func.__name__ = func.__name__
	return decorated_func
```

### Пример 6. Класс как декоратор

Функцию можно декорировать классом. Этот способ является альтернативой декораторам с аргументами.

Если в классе реализован специальный метод `__call__`, то экземпляр такого класса будет вызываемым (как функция) и называться функтором.
Таким образом, в `__call__` мы можем передавать декорируемую функцию, например:

```python
from functools import wraps
class Repeater:
    def __init__(self, n):
        self.n = n
    def __call__(self, f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            for _ in range(self.n):
                f(*args, **kwargs)
        return wrapper
@Repeater(3)
def foo():
    print('foo')
foo() 
# foo
# foo
# foo
```

### Полезные ссылки и источники примеров

- [Декораторы](https://advpyneng.readthedocs.io/ru/latest/book/08_decorators/index.html){:target="_blank"}
- [Примеры декораторов](https://github.com/natenka/advpyneng-examples-exercises/tree/main/examples/08_decorators){:target="_blank"}
- [Примеры декораторов](https://realpython.com/primer-on-python-decorators/){:target="_blank"}
- [Модуль functools](https://docs.python.org/3.11/library/functools.html){:target="_blank"}
- [Класс-декоратор и декоратор класса](https://tirinox.ru/class-decorator/){:target="_blank"}
