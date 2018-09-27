---
layout: post
title:  "파이썬 실전 패턴"
date:   2018-09-27 21:31:00 +0900
categories: learning-tech
tags: [python, pattern, design-pattern, "design pattern", python2, python3, refactoring]
---

좋은 발표자료를 찾았다. [파이썬 실전 패턴!](https://speakerdeck.com/halst/python-best-practice-patterns) :) 봅시당.

### Composed Method
- 한가지 일만 하는 메소드를 작성하라.
- 메소드 안 코드는 동일한 추상화 수준을 가져야 한다.
- 그렇다면, 자연스럽게 작은 메소드들로 프로그램을 작성할 수 있다.

 ```python
 # Bad
class Boilder:
    ...
    def safety_check(self):
        # Convert fixed-point floating-point:
        temperature = self.modbus.read_holding()
        pressure_psi = self.abb_f100.register / F100_FACTOR
        if psi_to_pascal(pressure_psi) > MAX_PRESSURE:
            if temperature > MAX_TEMPERATURE:
                # Shutdown!
                self.pnoz.relay[15] &= MASK_POWER_COIL
                self.pnoz.port.write('$PL,15\0')
                sleep(RELAY_RESPONSE_DELAY)
                # Successfull shutdown?
                if self.pnoz.relay[16] & MASK_POWER_OFF:
                    # Paly alarm:
                    with open(BUZZER_MP3_FILE) as f:
                        play_sound(r.read())
```

```python
# Better
class Boilder:
    ...
    def safety_check(self):
        if any([self.temperature > MAX_TEMPERATURE,
                self.pressure > MAX_PRESSURE]):
            if not self.shutdown():
                self.alarm()

    def alarm(self):
        with open(BUZZER_MP3_FILE) as f:
            play_sound(f.read())

    @property
    def pressure(self):
        pressure_psi = self.abb_f100.register / F100_FACTOR
        return psi_to_pascal(pressure_psi)
    ...
```

### Constructor Method
- 인스턴스를 잘 만들 수 있는 메소드를 제공하라.
    - 모든 required 인자들 (반드시 넘겨야하는 인자들) 은 이 메소드로 넘겨라.

```python
point = Point()
point.x = 12
point.y = 5

point = Point(x=12, y=5)

point = Point.polar(r=13, theta=22.6)
```

```python
class Point:

    def __init__(self, x, y):
        self.x, self.y = x, y

    @classmethod
    def polar(class_, r, theta):
        return class_(r * cos(theta),
                      r * sin(theta))
```

### Method Object
- 한 메소드 안 여러 라인이 서로 다른 인자들을 많이 공유하고 있을 때? 어떻게 하나?
- (개인적인 경험으로는 한 메소드 안에서 extract method 로 여러 작은 메소드를 뽑을 때, extract method 되는 메소드들끼리 여러 인자를 공유하는 경우, 그 인자들을 메소드 인자로 넘겨주고 반환 받아야할 때가 있다. 그 때 메소드 인자 개수나, 반환 개수나 엄청나게 늘어나 코드가 보기 싫어진다.)

```python
# Bad
def send_task(self, task, job, obligation):
    ...
    processed = ...
    ...
    copied = ...
    ...
    executed = ...
    ...
    100 more lines
```

```python
# Bad
def prepare_task(self, task, job, obligation
                 processed, copied, executed):
    ...
    ...
    return processed, copied, executed
```

```python
class TaskSender:

    def __init__(self, task, job, obligation):
        self.task = task
        self.job = job
        self.obligation = obligation
        self.processed = []
        self.copied = []
        self.executed = []

    def __call__(self):
        ...
        self.processed = ...
        ...
        self.copied = ...
        ...
        self.executed = ...
        ...
        100 more lines
```

```python
# Better
class TaskSender:

    def __init__(self, task, job, obligation):
        self.task = task
        self.job = job
        self.obligation = obligation
        self.processed = []
        self.copied = []
        self.executed = []

    def __call__(self):
        self.prepare()
        self.process()
        self.execute()

    ...
```

### Execute Around Method
- 항상 함께 있어야하는 코드 쌍은 어떻게 표현해야할까?

```python
f = open('file.txt', 'w')
f.write('hi')
f.close()

with open('file.txt', 'w') as f:
    f.write('hi')

with pytest.raises(ValueError):
    int('hi')

with client.indent(4, quote=colored.blue('>')):
    puts('This is indented text.')

with SomeProtocol(host, port) as protocol:
    protocol.send(['get', signal])
    result = protocol.receive()
```

```python
class SomeProtocol:

    def __init__(self, host, port):
        self.host, self.port = host, port

    def __enter__(self):
        self._client = socket()
        self._client.connect((self.host, self.port))
        return self

    def __exit__(self, exception, 
                 value, traceback):
        self._client.close()

    def send(self, payload): ...

    def receive(self): ...
```

### Debug Printing Method
- `__str__` 는 유저용
    - e.g. `print(point)`
- `__repr__` 는 디버그용/인터렉티브 모드용

```python
>>> Point(12, 5)
<__main__.Point instance at 0x100b4a758>

>>> Point(12, 5)
Point(x=12, y=5)

>>> print(Point(12, 5))
(12, 5)
```

```python
class Point:
    ...
    def __str__(self):
        return '({x}, {y})'.format(x=self.x,
                                   y=self.y)

    def __repr__(self):
        return '{}(x={}, y={})'.format(
                   self.__class__.__name__,
                   self.x, self.y)
```

### Method Comment
- 작은 메소드는 코멘트보다 유용하다.

```python
# Bad
if self.flags & 0b1000:  # Am I visible?
    ...

# Better
@property
def is_visible(self):
    return self.flags & 0b1000

if self.is_visible:
    ...

# Tell my station to process me
self.station.process(self)
```

### Choosing Message
```python
# Bad
if type(entry) is Film:
    responsible = entry.producer
else:
    responsible = entry.author

# Shouldn't use type() or isinstance() in conditional --> smelly

# Better
class Film:
    ...
    @property
    def responsible(self):
        return self.producer

entry.responsible
```

### Intention Revealing Message
- 구현이 간단한 경우, 어떻게 의도를 전달할 것인가?

```python
# Bad
class ParagraphEditor:
    ...
    def highlight(self, rectangle):
        self.reverse(rectangle)

# Better
class ParagraphEditor:
    ...
    highlight = reverse  # More readable, more composable
```

### Constant Method (Constant Class Variable)

```python
# Bad
_DEFAULT_PORT = 1234

class SomeProtocol:
    ...
    def __enter__(self):
        self._client = socket()
        self._client.connect((self.host, self.port or _DEFAULT_PORT))
        return self

# If you want to subclass SomeProtocol, you would have to overwrite every method!

# Better
class SomeProtocol:

    _default_port = 1234
    ...
    def __enter__(self):
        self._client = socket()
        self._client.connect((self.host, self.port or self._default_port))
        return self
```

### Direct and indirect variable access
- 더 이상 getter, setter 는 그만!

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

# Sometimes need more flexibility --> use properties

class Point:
    def __init__(self, x, y):
        self._x, self._y = x, y

    @property
    def x(self):
        return self._x

    @x.setter
    def x(self, value):
        self._x = value
```

### Enumeration (Iteration) Method

```python
# Bad
# Can't change type of collection
#     e.g. can't change employees from a list to a set
class Department:
    def __init__(self, *employees):
        self.employees = employees

for employee in department.employees:
    ...
    
# Better
class Department:
    def __init__(self, *employees):
        self._employees = employees

    def __iter__(self):
        return iter(self._employees)

for employee in department:  # More readable, more composable
    ...
```

### Temporary Variable
```python
# Bad
class Rectangle:
    def bottom_right(self):
        return Point(self.left + self.width,
                    self.top + self.height)

# Better to use temporary variables for readability
class Rectangle:
    ...
    def bottom_right(self):
        right = self.left + self.width
        bottom = self.top + self.height
        return Point(right, bottom)
```

### Sets
- `set` 을 사용하라. (중복 제거 등 여러가지 일을 할 수 있음)

```python

item in a_set
item not in a_set

# a_set <= other
a_set.is_subset(other)

# a_set | other
a_set.union(other)

# a_set & other
a_set.intersection(other)

# a_set - other
a_set.difference(other)
```

### Equality method
```python
obj == obj2
obj1 is obj2

class Book:
    ...
    def __eq__(self, other):
        if not isinstance(other, self.__class__):
            return NotImplemented
        return (self.author == other.author and 
                self.title == other.title)
```

### Hashing method
```python
class Book:
    ...
    def __hash__(self):
        return hash(self.author) ^ hash(self.other)
```

### Sorted collection
```python
class Book:
    ...
    def __lt__(self, other):
        return (self.author, self.title) < (other.author, other.title)
```

### Concatenation
```python
class Config:
    def __init__(self, **entries):
        self.entries = entries

    def __add__(self, other):
        entries = (self.entries.items() + 
                    other.entries.items())
        return Config(**entries)
default_config = Config(color=False, port=8080)
config = default_config + Config(color=True)
```

### Concatenating Streams
```python
for each in big_list + another_big_list:
    ...
for each in itertools.chain(big_list, another_big_list):
    ...
```

### Guard Clause
```python
def append(entries):
    if entries is None:
        return
    ...
```

### Simple enumeration parameter
- iteration 파라미터 변수명을 무엇으로 해야할지 아이디어가 떠오르지 않으면, 그냥 `each`를 사용하라.

```python
# Awkward
for options_shortcut in self.options_shortcuts:
    options_shortcut.this()
    options_shortcut.that()

# Better
for each in self.options_shortcuts:
    each.this()
    each.that()   
```

### Cascade
- setter 와 같이 멤버 변수에 값을 쓰고 반환 값이 없는 경우, `self` 반환해주자. (메소드간 cascading 을 위해)

```python
# Instead of this
self.release_water()
self.shutdown()
self.alarm()

class Reactor:
    ...
    def release_water(self):
        self.valve.open()
        return self

self.release_water().shutdown().alarm()   
```

### Interesting Return Value
- `None` 이라도 명시적으로 반환 값을 적어줌이 좋다.

```python
# Meh
def match(self, items):
    for each in items:
        if each.match(self):
            return each
# Is this supposed to reach the end? Is this a bug?

# Better
def match(self, items):
    for each in items:
        if each.match(self):
            return each
    return None
```