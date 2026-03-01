# Chapter 18: Design Patterns

[← Previous: Classic System Designs](17-system-design-classic.md) | [Next: Object-Oriented Design →](19-object-oriented-design.md)

---

## 18.1 Design Pattern Overview

```
Design Patterns = Proven solutions to recurring software design problems

Categories:
  Creational (5):   Object creation mechanisms
  Structural (7):   Composition of classes/objects
  Behavioral (11):  Communication between objects

                    ┌─── Creational ───┐
                    │  Factory          │
                    │  Abstract Factory │
                    │  Builder          │
                    │  Singleton        │
                    │  Prototype        │
                    ├─── Structural ────┤
                    │  Adapter          │
                    │  Bridge           │
                    │  Composite        │
                    │  Decorator        │
                    │  Facade           │
                    │  Flyweight        │
                    │  Proxy            │
                    ├─── Behavioral ────┤
                    │  Chain of Resp.   │
                    │  Command          │
                    │  Iterator         │
                    │  Mediator         │
                    │  Memento          │
                    │  Observer         │
                    │  State            │
                    │  Strategy         │
                    │  Template Method  │
                    │  Visitor          │
                    │  Interpreter      │
                    └──────────────────┘
```

---

# Part A: Creational Patterns

## 18.2 Factory Method

```
Problem:  Create objects without specifying exact class
When:     Object creation logic should be separate from usage
          Subclasses should decide which class to instantiate
```

```python
from abc import ABC, abstractmethod

# Product interface
class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass

# Concrete products
class EmailNotification(Notification):
    def send(self, message: str):
        print(f"Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str):
        print(f"SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str):
        print(f"Push: {message}")

# Factory
class NotificationFactory:
    _creators = {
        "email": EmailNotification,
        "sms": SMSNotification,
        "push": PushNotification,
    }

    @staticmethod
    def create(channel: str) -> Notification:
        creator = NotificationFactory._creators.get(channel)
        if not creator:
            raise ValueError(f"Unknown channel: {channel}")
        return creator()

# Usage
notification = NotificationFactory.create("email")
notification.send("Hello!")
```

---

## 18.3 Abstract Factory

```
Problem:  Create families of related objects without specifying concrete classes
When:     System should be independent of how products are created
          Need to ensure products from same family are used together
```

```python
from abc import ABC, abstractmethod

# Abstract products
class Button(ABC):
    @abstractmethod
    def render(self): pass

class Checkbox(ABC):
    @abstractmethod
    def render(self): pass

# Concrete products — Light theme
class LightButton(Button):
    def render(self):
        return "[ Light Button ]"

class LightCheckbox(Checkbox):
    def render(self):
        return "[✓] Light Checkbox"

# Concrete products — Dark theme
class DarkButton(Button):
    def render(self):
        return "[ Dark Button ]"

class DarkCheckbox(Checkbox):
    def render(self):
        return "[✓] Dark Checkbox"

# Abstract factory
class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: pass

    @abstractmethod
    def create_checkbox(self) -> Checkbox: pass

# Concrete factories
class LightThemeFactory(UIFactory):
    def create_button(self): return LightButton()
    def create_checkbox(self): return LightCheckbox()

class DarkThemeFactory(UIFactory):
    def create_button(self): return DarkButton()
    def create_checkbox(self): return DarkCheckbox()

# Client code — works with any factory
def build_ui(factory: UIFactory):
    button = factory.create_button()
    checkbox = factory.create_checkbox()
    print(button.render(), checkbox.render())

build_ui(DarkThemeFactory())  # All dark theme components
```

---

## 18.4 Builder

```
Problem:  Construct complex objects step by step
When:     Object has many optional parameters
          Same construction process should create different representations
```

```python
class HttpRequest:
    def __init__(self):
        self.method = "GET"
        self.url = ""
        self.headers = {}
        self.body = None
        self.timeout = 30
        self.retries = 0

    def __repr__(self):
        return f"HttpRequest({self.method} {self.url}, headers={self.headers})"

class HttpRequestBuilder:
    def __init__(self, url: str):
        self._request = HttpRequest()
        self._request.url = url

    def method(self, method: str) -> 'HttpRequestBuilder':
        self._request.method = method
        return self

    def header(self, key: str, value: str) -> 'HttpRequestBuilder':
        self._request.headers[key] = value
        return self

    def body(self, data) -> 'HttpRequestBuilder':
        self._request.body = data
        return self

    def timeout(self, seconds: int) -> 'HttpRequestBuilder':
        self._request.timeout = seconds
        return self

    def retries(self, count: int) -> 'HttpRequestBuilder':
        self._request.retries = count
        return self

    def build(self) -> HttpRequest:
        # Validation
        if self._request.method in ("POST", "PUT") and not self._request.body:
            raise ValueError("Body required for POST/PUT")
        return self._request

# Fluent interface
request = (HttpRequestBuilder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body({"name": "Alice", "email": "alice@example.com"})
    .timeout(10)
    .retries(3)
    .build())
```

---

## 18.5 Singleton

```
Problem:  Ensure a class has only one instance with global access
When:     Shared resource (database connection pool, config, logger)
Caution:  Often considered an anti-pattern — use dependency injection instead
```

```python
import threading

# Thread-safe Singleton
class DatabasePool:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, max_connections=10):
        if not hasattr(self, '_initialized'):
            self._initialized = True
            self.max_connections = max_connections
            self.pool = []
            print(f"Pool created with {max_connections} connections")

# Both are the same instance
pool1 = DatabasePool(10)
pool2 = DatabasePool(20)
assert pool1 is pool2

# Pythonic alternative: Module-level singleton
# Just use a module — Python modules are singletons by nature
# config.py → import config → same instance everywhere

# Another alternative: Borg pattern (shared state, different instances)
class Borg:
    _shared_state = {}
    def __init__(self):
        self.__dict__ = self._shared_state
```

---

## 18.6 Prototype

```
Problem:  Create new objects by cloning an existing instance
When:     Creating an object is expensive (complex initialization, DB reads)
          Need many similar objects with slight differences
```

```python
import copy

class GameUnit:
    def __init__(self, name, hp, attack, defense, abilities=None):
        self.name = name
        self.hp = hp
        self.attack = attack
        self.defense = defense
        self.abilities = abilities or []
        # Imagine expensive initialization: loading textures, AI config, etc.

    def clone(self):
        """Deep copy to create independent instance."""
        return copy.deepcopy(self)

    def __repr__(self):
        return f"{self.name}(HP:{self.hp}, ATK:{self.attack})"

# Prototype registry
class UnitRegistry:
    def __init__(self):
        self._prototypes = {}

    def register(self, key, prototype):
        self._prototypes[key] = prototype

    def create(self, key, **overrides):
        proto = self._prototypes[key].clone()
        for k, v in overrides.items():
            setattr(proto, k, v)
        return proto

# Setup prototypes once
registry = UnitRegistry()
registry.register("warrior", GameUnit("Warrior", 100, 15, 10, ["slash"]))
registry.register("mage", GameUnit("Mage", 60, 25, 5, ["fireball", "heal"]))

# Create units by cloning (cheap, no re-initialization)
w1 = registry.create("warrior", name="Knight", hp=120)
w2 = registry.create("warrior", name="Berserker", attack=25)
m1 = registry.create("mage", name="Archmage")
```

---

# Part B: Structural Patterns

## 18.7 Adapter

```
Problem:  Make incompatible interfaces work together
When:     Using a third-party library with different interface
          Legacy code integration
```

```python
# Existing interface our code expects
class PaymentProcessor:
    def pay(self, amount: float, currency: str) -> bool:
        raise NotImplementedError

# Third-party library with incompatible interface
class StripeAPI:
    def create_charge(self, amount_cents: int, currency_code: str,
                      idempotency_key: str) -> dict:
        print(f"Stripe: Charging {amount_cents} cents in {currency_code}")
        return {"id": "ch_123", "status": "succeeded"}

# Adapter: Makes StripeAPI work like PaymentProcessor
class StripeAdapter(PaymentProcessor):
    def __init__(self, stripe: StripeAPI):
        self._stripe = stripe

    def pay(self, amount: float, currency: str) -> bool:
        import uuid
        amount_cents = int(amount * 100)
        result = self._stripe.create_charge(
            amount_cents=amount_cents,
            currency_code=currency.lower(),
            idempotency_key=str(uuid.uuid4())
        )
        return result["status"] == "succeeded"

# Client code works with our interface
def checkout(processor: PaymentProcessor, amount: float):
    if processor.pay(amount, "USD"):
        print("Payment successful!")

stripe_adapter = StripeAdapter(StripeAPI())
checkout(stripe_adapter, 29.99)
```

---

## 18.8 Bridge

```
Problem:  Separate abstraction from implementation so both can vary independently
When:     Avoid class explosion from combinations (N shapes × M renderers)
```

```python
from abc import ABC, abstractmethod

# Implementation hierarchy (how to render)
class Renderer(ABC):
    @abstractmethod
    def render_circle(self, x, y, radius): pass

    @abstractmethod
    def render_rectangle(self, x, y, width, height): pass

class SVGRenderer(Renderer):
    def render_circle(self, x, y, radius):
        return f'<circle cx="{x}" cy="{y}" r="{radius}"/>'

    def render_rectangle(self, x, y, width, height):
        return f'<rect x="{x}" y="{y}" width="{width}" height="{height}"/>'

class CanvasRenderer(Renderer):
    def render_circle(self, x, y, radius):
        return f'ctx.arc({x}, {y}, {radius}, 0, 2*Math.PI)'

    def render_rectangle(self, x, y, width, height):
        return f'ctx.fillRect({x}, {y}, {width}, {height})'

# Abstraction hierarchy (what to draw)
class Shape(ABC):
    def __init__(self, renderer: Renderer):
        self.renderer = renderer  # Bridge to implementation

    @abstractmethod
    def draw(self): pass

class Circle(Shape):
    def __init__(self, renderer, x, y, radius):
        super().__init__(renderer)
        self.x, self.y, self.radius = x, y, radius

    def draw(self):
        return self.renderer.render_circle(self.x, self.y, self.radius)

# N shapes × M renderers without class explosion
circle_svg = Circle(SVGRenderer(), 10, 20, 5)
circle_canvas = Circle(CanvasRenderer(), 10, 20, 5)
print(circle_svg.draw())     # <circle cx="10" cy="20" r="5"/>
print(circle_canvas.draw())  # ctx.arc(10, 20, 5, 0, 2*Math.PI)
```

---

## 18.9 Composite

```
Problem:  Treat individual objects and compositions uniformly (tree structure)
When:     File systems, UI component trees, organization charts
```

```python
from abc import ABC, abstractmethod
from typing import List

class FileSystemNode(ABC):
    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    def size(self) -> int: pass

    @abstractmethod
    def display(self, indent: int = 0) -> str: pass

class File(FileSystemNode):
    def __init__(self, name: str, size: int):
        super().__init__(name)
        self._size = size

    def size(self) -> int:
        return self._size

    def display(self, indent=0):
        return " " * indent + f"📄 {self.name} ({self._size} bytes)"

class Directory(FileSystemNode):
    def __init__(self, name: str):
        super().__init__(name)
        self.children: List[FileSystemNode] = []

    def add(self, node: FileSystemNode):
        self.children.append(node)
        return self

    def size(self) -> int:
        return sum(child.size() for child in self.children)

    def display(self, indent=0):
        lines = [" " * indent + f"📁 {self.name} ({self.size()} bytes)"]
        for child in self.children:
            lines.append(child.display(indent + 2))
        return "\n".join(lines)

# Usage — uniform interface for files and directories
root = Directory("src")
root.add(File("main.py", 1024))
root.add(File("utils.py", 512))
tests = Directory("tests")
tests.add(File("test_main.py", 768))
root.add(tests)

print(root.display())
print(f"Total: {root.size()} bytes")
# 📁 src (2304 bytes)
#   📄 main.py (1024 bytes)
#   📄 utils.py (512 bytes)
#   📁 tests (768 bytes)
#     📄 test_main.py (768 bytes)
```

---

## 18.10 Decorator

```
Problem:  Add behavior to objects dynamically without altering their class
When:     Adding features like logging, caching, auth to existing code
          Alternative to subclassing for extending functionality
```

```python
from abc import ABC, abstractmethod
import time
import functools

# Component interface
class DataSource(ABC):
    @abstractmethod
    def read(self) -> str: pass

    @abstractmethod
    def write(self, data: str) -> None: pass

# Concrete component
class FileDataSource(DataSource):
    def __init__(self, filename: str):
        self.filename = filename
        self._data = ""

    def read(self) -> str:
        return self._data

    def write(self, data: str):
        self._data = data

# Base decorator
class DataSourceDecorator(DataSource):
    def __init__(self, source: DataSource):
        self._source = source

    def read(self) -> str:
        return self._source.read()

    def write(self, data: str):
        self._source.write(data)

# Concrete decorators
class EncryptionDecorator(DataSourceDecorator):
    def read(self) -> str:
        data = super().read()
        return self._decrypt(data)

    def write(self, data: str):
        super().write(self._encrypt(data))

    def _encrypt(self, data): return f"[ENC]{data}[/ENC]"
    def _decrypt(self, data): return data.replace("[ENC]", "").replace("[/ENC]", "")

class CompressionDecorator(DataSourceDecorator):
    def read(self) -> str:
        data = super().read()
        return self._decompress(data)

    def write(self, data: str):
        super().write(self._compress(data))

    def _compress(self, data): return f"[ZIP]{data}[/ZIP]"
    def _decompress(self, data): return data.replace("[ZIP]", "").replace("[/ZIP]", "")

# Stack decorators — order matters!
source = FileDataSource("data.txt")
encrypted = EncryptionDecorator(source)
compressed = CompressionDecorator(encrypted)

compressed.write("Hello, World!")
print(source.read())  # [ZIP][ENC]Hello, World![/ENC][/ZIP]


# Python decorator (function-based) — same concept, different syntax
def retry(max_attempts=3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(2 ** attempt)
        return wrapper
    return decorator

def cache(func):
    _cache = {}
    @functools.wraps(func)
    def wrapper(*args):
        if args not in _cache:
            _cache[args] = func(*args)
        return _cache[args]
    return wrapper

@retry(max_attempts=3)
@cache
def fetch_user(user_id):
    # expensive network call
    pass
```

---

## 18.11 Facade

```
Problem:  Provide a simple interface to a complex subsystem
When:     Simplifying library/framework usage
          Reducing coupling between client and subsystem
```

```python
# Complex subsystem classes
class VideoDecoder:
    def decode(self, filename): return f"decoded_{filename}"

class AudioDecoder:
    def decode(self, filename): return f"audio_{filename}"

class SubtitleParser:
    def parse(self, filename): return f"subs_{filename}"

class VideoRenderer:
    def render(self, video, audio, subs):
        return f"Rendering: {video} + {audio} + {subs}"

class StreamBuffer:
    def __init__(self, size=1024):
        self.size = size
    def allocate(self):
        return f"Buffer({self.size}KB)"

# Facade — simple interface hiding complexity
class VideoPlayerFacade:
    def __init__(self):
        self._video_decoder = VideoDecoder()
        self._audio_decoder = AudioDecoder()
        self._subtitle_parser = SubtitleParser()
        self._renderer = VideoRenderer()
        self._buffer = StreamBuffer()

    def play(self, filename: str):
        """One simple method hides 5+ subsystem interactions."""
        self._buffer.allocate()
        video = self._video_decoder.decode(filename)
        audio = self._audio_decoder.decode(filename)
        subs = self._subtitle_parser.parse(filename)
        return self._renderer.render(video, audio, subs)

# Client uses simple interface
player = VideoPlayerFacade()
player.play("movie.mp4")
```

---

## 18.12 Flyweight

```
Problem:  Share fine-grained objects to save memory
When:     Application uses large number of similar objects
          Most object state can be shared (intrinsic vs extrinsic)
```

```python
class CharacterStyle:
    """Flyweight: shared intrinsic state (font, size, color)."""
    _cache = {}

    def __new__(cls, font, size, color):
        key = (font, size, color)
        if key not in cls._cache:
            instance = super().__new__(cls)
            instance.font = font
            instance.size = size
            instance.color = color
            cls._cache[key] = instance
        return cls._cache[key]

    def render(self, char, x, y):
        return f"'{char}' at ({x},{y}) in {self.font} {self.size}pt {self.color}"

# Extrinsic state: character, position (unique per occurrence)
class Character:
    __slots__ = ['char', 'x', 'y', 'style']  # Memory optimization

    def __init__(self, char, x, y, style: CharacterStyle):
        self.char = char
        self.x = x
        self.y = y
        self.style = style  # Shared flyweight

    def render(self):
        return self.style.render(self.char, self.x, self.y)

# Document with 1 million characters but only ~10 unique styles
body_style = CharacterStyle("Arial", 12, "black")
heading_style = CharacterStyle("Arial", 24, "blue")
same_body = CharacterStyle("Arial", 12, "black")

assert body_style is same_body  # Same instance! Memory saved.

# 1M characters share ~10 style objects instead of each having their own
chars = [Character("a", i, 0, body_style) for i in range(1000000)]
print(f"Style objects in memory: {len(CharacterStyle._cache)}")  # ~10, not 1M
```

---

## 18.13 Proxy

```
Problem:  Control access to an object
When:     Lazy loading, access control, logging, caching, remote objects
```

```python
from abc import ABC, abstractmethod
import time

class Database(ABC):
    @abstractmethod
    def query(self, sql: str) -> list: pass

class RealDatabase(Database):
    def __init__(self, connection_string: str):
        print(f"Connecting to {connection_string}...")
        time.sleep(0.1)  # Expensive connection
        self._conn = connection_string

    def query(self, sql: str) -> list:
        return [f"result for: {sql}"]

# Virtual Proxy (lazy initialization)
class LazyDatabaseProxy(Database):
    def __init__(self, connection_string: str):
        self._connection_string = connection_string
        self._real_db = None  # Not created yet

    def query(self, sql: str) -> list:
        if self._real_db is None:
            self._real_db = RealDatabase(self._connection_string)
        return self._real_db.query(sql)

# Protection Proxy (access control)
class SecureDatabaseProxy(Database):
    def __init__(self, real_db: Database, allowed_users: set):
        self._real_db = real_db
        self._allowed_users = allowed_users
        self._current_user = None

    def authenticate(self, user: str):
        self._current_user = user

    def query(self, sql: str) -> list:
        if self._current_user not in self._allowed_users:
            raise PermissionError(f"User {self._current_user} not authorized")
        if sql.strip().upper().startswith(("DROP", "DELETE", "TRUNCATE")):
            raise PermissionError("Destructive queries not allowed")
        return self._real_db.query(sql)

# Caching Proxy
class CachingDatabaseProxy(Database):
    def __init__(self, real_db: Database, ttl: float = 60):
        self._real_db = real_db
        self._cache = {}
        self._timestamps = {}
        self._ttl = ttl

    def query(self, sql: str) -> list:
        now = time.time()
        if sql in self._cache and now - self._timestamps[sql] < self._ttl:
            print(f"Cache hit: {sql}")
            return self._cache[sql]
        result = self._real_db.query(sql)
        self._cache[sql] = result
        self._timestamps[sql] = now
        return result
```

---

# Part C: Behavioral Patterns

## 18.14 Observer

```
Problem:  One-to-many dependency: when one object changes, notify all dependents
When:     Event systems, UI updates, pub/sub
```

```python
from abc import ABC, abstractmethod
from typing import List, Any
from collections import defaultdict

class EventEmitter:
    """Flexible observer with named events."""
    def __init__(self):
        self._listeners = defaultdict(list)

    def on(self, event: str, callback):
        self._listeners[event].append(callback)
        return self  # Chainable

    def off(self, event: str, callback):
        self._listeners[event].remove(callback)

    def emit(self, event: str, *args, **kwargs):
        for callback in self._listeners[event]:
            callback(*args, **kwargs)

# Usage
class UserService(EventEmitter):
    def register(self, username, email):
        # ... create user ...
        user = {"username": username, "email": email}
        self.emit("user_registered", user)
        return user

# Subscribers are decoupled from UserService
service = UserService()
service.on("user_registered", lambda u: print(f"Send welcome email to {u['email']}"))
service.on("user_registered", lambda u: print(f"Create default settings for {u['username']}"))
service.on("user_registered", lambda u: print(f"Log: New user {u['username']}"))

service.register("alice", "alice@example.com")
# Send welcome email to alice@example.com
# Create default settings for alice
# Log: New user alice
```

---

## 18.15 Strategy

```
Problem:  Select algorithm at runtime from a family of algorithms
When:     Multiple ways to do the same thing (sorting, compression, routing)
```

```python
from abc import ABC, abstractmethod
from typing import List

class CompressionStrategy(ABC):
    @abstractmethod
    def compress(self, data: bytes) -> bytes: pass

    @abstractmethod
    def decompress(self, data: bytes) -> bytes: pass

class GzipStrategy(CompressionStrategy):
    def compress(self, data: bytes) -> bytes:
        import gzip
        return gzip.compress(data)

    def decompress(self, data: bytes) -> bytes:
        import gzip
        return gzip.decompress(data)

class LZ4Strategy(CompressionStrategy):
    def compress(self, data: bytes) -> bytes:
        # Fast compression, lower ratio
        return b"lz4:" + data  # simplified

    def decompress(self, data: bytes) -> bytes:
        return data[4:]

class NoCompression(CompressionStrategy):
    def compress(self, data: bytes) -> bytes: return data
    def decompress(self, data: bytes) -> bytes: return data

# Context uses strategy
class FileStorage:
    def __init__(self, strategy: CompressionStrategy = None):
        self._strategy = strategy or NoCompression()
        self._store = {}

    def set_strategy(self, strategy: CompressionStrategy):
        self._strategy = strategy

    def save(self, key: str, data: bytes):
        self._store[key] = self._strategy.compress(data)

    def load(self, key: str) -> bytes:
        return self._strategy.decompress(self._store[key])

# Swap algorithms at runtime
storage = FileStorage(GzipStrategy())
storage.save("log", b"large log data...")
storage.set_strategy(LZ4Strategy())  # Switch to faster compression
storage.save("realtime", b"time-sensitive data...")
```

---

## 18.16 Command

```
Problem:  Encapsulate request as an object (parameterize, queue, undo/redo)
When:     Undo/redo, macro recording, task queuing, transaction logging
```

```python
from abc import ABC, abstractmethod
from typing import List

class Command(ABC):
    @abstractmethod
    def execute(self): pass

    @abstractmethod
    def undo(self): pass

class TextEditor:
    def __init__(self):
        self.content = ""

    def __repr__(self):
        return f"TextEditor('{self.content}')"

class InsertCommand(Command):
    def __init__(self, editor: TextEditor, text: str, position: int):
        self.editor = editor
        self.text = text
        self.position = position

    def execute(self):
        self.editor.content = (self.editor.content[:self.position] +
                                self.text +
                                self.editor.content[self.position:])

    def undo(self):
        self.editor.content = (self.editor.content[:self.position] +
                                self.editor.content[self.position + len(self.text):])

class DeleteCommand(Command):
    def __init__(self, editor: TextEditor, position: int, length: int):
        self.editor = editor
        self.position = position
        self.length = length
        self.deleted_text = ""

    def execute(self):
        self.deleted_text = self.editor.content[self.position:self.position + self.length]
        self.editor.content = (self.editor.content[:self.position] +
                                self.editor.content[self.position + self.length:])

    def undo(self):
        self.editor.content = (self.editor.content[:self.position] +
                                self.deleted_text +
                                self.editor.content[self.position:])

class CommandHistory:
    def __init__(self):
        self._undo_stack: List[Command] = []
        self._redo_stack: List[Command] = []

    def execute(self, command: Command):
        command.execute()
        self._undo_stack.append(command)
        self._redo_stack.clear()  # New action invalidates redo

    def undo(self):
        if self._undo_stack:
            command = self._undo_stack.pop()
            command.undo()
            self._redo_stack.append(command)

    def redo(self):
        if self._redo_stack:
            command = self._redo_stack.pop()
            command.execute()
            self._undo_stack.append(command)

# Usage
editor = TextEditor()
history = CommandHistory()

history.execute(InsertCommand(editor, "Hello", 0))       # "Hello"
history.execute(InsertCommand(editor, " World!", 5))      # "Hello World!"
history.execute(DeleteCommand(editor, 5, 6))              # "Hello!"
history.undo()                                             # "Hello World!"
history.undo()                                             # "Hello"
history.redo()                                             # "Hello World!"
```

---

## 18.17 State

```
Problem:  Object behavior changes based on internal state
When:     State machines with complex transitions
          Avoiding large if/elif chains based on state
```

```python
from abc import ABC, abstractmethod

class OrderState(ABC):
    @abstractmethod
    def pay(self, order): pass
    @abstractmethod
    def ship(self, order): pass
    @abstractmethod
    def deliver(self, order): pass
    @abstractmethod
    def cancel(self, order): pass

class PendingState(OrderState):
    def pay(self, order):
        print("Payment received!")
        order.state = PaidState()

    def ship(self, order):
        print("Error: Can't ship unpaid order")

    def deliver(self, order):
        print("Error: Can't deliver unpaid order")

    def cancel(self, order):
        print("Order cancelled")
        order.state = CancelledState()

class PaidState(OrderState):
    def pay(self, order):
        print("Already paid")

    def ship(self, order):
        print("Order shipped!")
        order.state = ShippedState()

    def deliver(self, order):
        print("Error: Must ship first")

    def cancel(self, order):
        print("Refund issued, order cancelled")
        order.state = CancelledState()

class ShippedState(OrderState):
    def pay(self, order): print("Already paid")
    def ship(self, order): print("Already shipped")
    def deliver(self, order):
        print("Order delivered!")
        order.state = DeliveredState()
    def cancel(self, order):
        print("Error: Can't cancel shipped order")

class DeliveredState(OrderState):
    def pay(self, order): print("Already paid")
    def ship(self, order): print("Already delivered")
    def deliver(self, order): print("Already delivered")
    def cancel(self, order): print("Error: Already delivered")

class CancelledState(OrderState):
    def pay(self, order): print("Order is cancelled")
    def ship(self, order): print("Order is cancelled")
    def deliver(self, order): print("Order is cancelled")
    def cancel(self, order): print("Already cancelled")

class Order:
    def __init__(self):
        self.state: OrderState = PendingState()

    def pay(self): self.state.pay(self)
    def ship(self): self.state.ship(self)
    def deliver(self): self.state.deliver(self)
    def cancel(self): self.state.cancel(self)

order = Order()
order.ship()     # Error: Can't ship unpaid order
order.pay()      # Payment received!
order.ship()     # Order shipped!
order.cancel()   # Error: Can't cancel shipped order
order.deliver()  # Order delivered!
```

---

## 18.18 Template Method

```
Problem:  Define algorithm skeleton, let subclasses override specific steps
When:     Common algorithm with varying steps (data mining, report generation)
```

```python
from abc import ABC, abstractmethod

class DataMiner(ABC):
    def mine(self, path: str):
        """Template method — fixed algorithm skeleton."""
        data = self.extract(path)
        parsed = self.parse(data)
        analyzed = self.analyze(parsed)
        report = self.format_report(analyzed)
        self.send_report(report)
        return report

    @abstractmethod
    def extract(self, path: str) -> str:
        """Each subclass extracts differently."""
        pass

    @abstractmethod
    def parse(self, data: str) -> list:
        """Each subclass parses differently."""
        pass

    # Common steps with default implementation
    def analyze(self, records: list) -> dict:
        return {"count": len(records), "records": records}

    def format_report(self, analysis: dict) -> str:
        return f"Report: {analysis['count']} records processed"

    def send_report(self, report: str):
        print(f"Sending: {report}")

class CSVMiner(DataMiner):
    def extract(self, path: str) -> str:
        return "name,age\nAlice,30\nBob,25"

    def parse(self, data: str) -> list:
        lines = data.strip().split("\n")
        headers = lines[0].split(",")
        return [dict(zip(headers, line.split(","))) for line in lines[1:]]

class JSONMiner(DataMiner):
    def extract(self, path: str) -> str:
        return '[{"name": "Alice", "age": 30}]'

    def parse(self, data: str) -> list:
        import json
        return json.loads(data)

CSVMiner().mine("data.csv")    # Same algorithm, different extract/parse
JSONMiner().mine("data.json")
```

---

## 18.19 Chain of Responsibility

```
Problem:  Pass request along a chain of handlers until one processes it
When:     Middleware pipelines, event handling, auth chains
```

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class Request:
    path: str
    method: str
    headers: dict
    body: dict = None
    user: str = None

@dataclass
class Response:
    status: int
    body: dict

class Middleware(ABC):
    def __init__(self):
        self._next: 'Middleware' = None

    def set_next(self, handler: 'Middleware') -> 'Middleware':
        self._next = handler
        return handler  # Enables chaining

    def handle(self, request: Request) -> Response:
        if self._next:
            return self._next.handle(request)
        return Response(200, {"message": "OK"})

class AuthMiddleware(Middleware):
    def handle(self, request: Request) -> Response:
        token = request.headers.get("Authorization")
        if not token:
            return Response(401, {"error": "Unauthorized"})
        request.user = token.split()[-1]  # Extract user from token
        return super().handle(request)

class RateLimitMiddleware(Middleware):
    def __init__(self, max_requests=100):
        super().__init__()
        self._counts = {}
        self._max = max_requests

    def handle(self, request: Request) -> Response:
        user = request.user or request.headers.get("X-Forwarded-For", "anon")
        self._counts[user] = self._counts.get(user, 0) + 1
        if self._counts[user] > self._max:
            return Response(429, {"error": "Rate limit exceeded"})
        return super().handle(request)

class LoggingMiddleware(Middleware):
    def handle(self, request: Request) -> Response:
        print(f"[LOG] {request.method} {request.path}")
        response = super().handle(request)
        print(f"[LOG] Response: {response.status}")
        return response

class RouteHandler(Middleware):
    def handle(self, request: Request) -> Response:
        return Response(200, {"message": f"Hello, {request.user}!"})

# Build chain: Log → Auth → RateLimit → Handler
chain = LoggingMiddleware()
chain.set_next(AuthMiddleware()).set_next(
    RateLimitMiddleware()).set_next(RouteHandler())

# Process request through chain
req = Request("/api/data", "GET", {"Authorization": "Bearer alice"})
resp = chain.handle(req)
```

---

## 18.20 Iterator

```
Problem:  Access elements of a collection sequentially without exposing internals
When:     Custom collections, lazy evaluation, infinite sequences
```

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class BSTIterator:
    """In-order iterator using controlled stack — O(h) space, O(1) amortized next()."""
    def __init__(self, root: TreeNode):
        self.stack = []
        self._push_left(root)

    def _push_left(self, node):
        while node:
            self.stack.append(node)
            node = node.left

    def __iter__(self):
        return self

    def __next__(self):
        if not self.stack:
            raise StopIteration
        node = self.stack.pop()
        self._push_left(node.right)
        return node.val

# Python generators — the Pythonic way to create iterators
def fibonacci():
    """Infinite Fibonacci iterator."""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

def take(n, iterable):
    """Take first n items from any iterable."""
    for i, item in enumerate(iterable):
        if i >= n:
            break
        yield item

# Usage
tree = TreeNode(5, TreeNode(3, TreeNode(1), TreeNode(4)), TreeNode(7))
print(list(BSTIterator(tree)))  # [1, 3, 4, 5, 7]
print(list(take(10, fibonacci())))  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

## 18.21 Mediator

```
Problem:  Reduce chaotic dependencies between objects via central coordinator
When:     Chat rooms, air traffic control, UI form components
```

```python
from abc import ABC, abstractmethod

class ChatMediator:
    def __init__(self):
        self._users = {}
        self._rooms = {}

    def register(self, user: 'User'):
        self._users[user.name] = user

    def create_room(self, room_name: str):
        self._rooms[room_name] = set()

    def join_room(self, user_name: str, room_name: str):
        self._rooms.setdefault(room_name, set()).add(user_name)

    def send_message(self, sender: str, room: str, message: str):
        for user_name in self._rooms.get(room, set()):
            if user_name != sender:
                self._users[user_name].receive(sender, room, message)

    def direct_message(self, sender: str, recipient: str, message: str):
        if recipient in self._users:
            self._users[recipient].receive(sender, "DM", message)

class User:
    def __init__(self, name: str, mediator: ChatMediator):
        self.name = name
        self._mediator = mediator
        mediator.register(self)

    def send(self, room: str, message: str):
        self._mediator.send_message(self.name, room, message)

    def dm(self, recipient: str, message: str):
        self._mediator.direct_message(self.name, recipient, message)

    def receive(self, sender: str, channel: str, message: str):
        print(f"[{self.name}] ({channel}) {sender}: {message}")

# Users don't know about each other — only the mediator
chat = ChatMediator()
alice = User("Alice", chat)
bob = User("Bob", chat)
charlie = User("Charlie", chat)

chat.create_room("general")
chat.join_room("Alice", "general")
chat.join_room("Bob", "general")
chat.join_room("Charlie", "general")

alice.send("general", "Hello everyone!")
# [Bob] (general) Alice: Hello everyone!
# [Charlie] (general) Alice: Hello everyone!
```

---

## 18.22 Memento

```
Problem:  Capture and restore object's internal state (undo/snapshot)
When:     Save/load game state, form drafts, version history
```

```python
from dataclasses import dataclass
from typing import List
import copy

@dataclass(frozen=True)
class EditorMemento:
    """Immutable snapshot of editor state."""
    content: str
    cursor_position: int
    selection: tuple  # (start, end) or None

class Editor:
    def __init__(self):
        self.content = ""
        self.cursor_position = 0
        self.selection = None

    def type(self, text: str):
        self.content = (self.content[:self.cursor_position] +
                        text + self.content[self.cursor_position:])
        self.cursor_position += len(text)

    def save(self) -> EditorMemento:
        return EditorMemento(self.content, self.cursor_position, self.selection)

    def restore(self, memento: EditorMemento):
        self.content = memento.content
        self.cursor_position = memento.cursor_position
        self.selection = memento.selection

class History:
    def __init__(self, editor: Editor):
        self._editor = editor
        self._snapshots: List[EditorMemento] = []
        self._current = -1

    def save(self):
        # Remove any forward history (after undo)
        self._snapshots = self._snapshots[:self._current + 1]
        self._snapshots.append(self._editor.save())
        self._current += 1

    def undo(self):
        if self._current > 0:
            self._current -= 1
            self._editor.restore(self._snapshots[self._current])

    def redo(self):
        if self._current < len(self._snapshots) - 1:
            self._current += 1
            self._editor.restore(self._snapshots[self._current])

editor = Editor()
history = History(editor)

history.save()                                  # Save empty state
editor.type("Hello"); history.save()            # "Hello"
editor.type(" World"); history.save()           # "Hello World"
history.undo()                                  # "Hello"
history.undo()                                  # ""
history.redo()                                  # "Hello"
```

---

## 18.23 Visitor

```
Problem:  Add new operations to object structures without modifying them
When:     Operations on heterogeneous object hierarchies (AST, document nodes)
```

```python
from abc import ABC, abstractmethod

# Element hierarchy (rarely changes)
class ASTNode(ABC):
    @abstractmethod
    def accept(self, visitor: 'ASTVisitor'):
        pass

class NumberNode(ASTNode):
    def __init__(self, value: float):
        self.value = value
    def accept(self, visitor): return visitor.visit_number(self)

class BinaryOpNode(ASTNode):
    def __init__(self, op: str, left: ASTNode, right: ASTNode):
        self.op = op
        self.left = left
        self.right = right
    def accept(self, visitor): return visitor.visit_binary_op(self)

class UnaryOpNode(ASTNode):
    def __init__(self, op: str, operand: ASTNode):
        self.op = op
        self.operand = operand
    def accept(self, visitor): return visitor.visit_unary_op(self)

# Visitor interface
class ASTVisitor(ABC):
    @abstractmethod
    def visit_number(self, node: NumberNode): pass
    @abstractmethod
    def visit_binary_op(self, node: BinaryOpNode): pass
    @abstractmethod
    def visit_unary_op(self, node: UnaryOpNode): pass

# New operations added without modifying AST classes
class Evaluator(ASTVisitor):
    def visit_number(self, node): return node.value
    def visit_binary_op(self, node):
        l, r = node.left.accept(self), node.right.accept(self)
        ops = {'+': l+r, '-': l-r, '*': l*r, '/': l/r}
        return ops[node.op]
    def visit_unary_op(self, node):
        val = node.operand.accept(self)
        return -val if node.op == '-' else val

class PrettyPrinter(ASTVisitor):
    def visit_number(self, node): return str(node.value)
    def visit_binary_op(self, node):
        l = node.left.accept(self)
        r = node.right.accept(self)
        return f"({l} {node.op} {r})"
    def visit_unary_op(self, node):
        return f"({node.op}{node.operand.accept(self)})"

# AST: (3 + 4) * (-2)
ast = BinaryOpNode("*",
    BinaryOpNode("+", NumberNode(3), NumberNode(4)),
    UnaryOpNode("-", NumberNode(2)))

print(PrettyPrinter().visit_binary_op(ast))  # ((3 + 4) * (-2))
print(Evaluator().visit_binary_op(ast))       # -14.0
```

---

## 18.24 Interpreter

```
Problem:  Define a grammar and interpret sentences in that language
When:     DSLs, query parsers, rule engines, math expression evaluators
```

```python
from abc import ABC, abstractmethod

class Expression(ABC):
    @abstractmethod
    def interpret(self, context: dict) -> bool:
        pass

class Variable(Expression):
    def __init__(self, name: str):
        self.name = name
    def interpret(self, context: dict) -> bool:
        return context.get(self.name, False)

class And(Expression):
    def __init__(self, left: Expression, right: Expression):
        self.left = left
        self.right = right
    def interpret(self, context: dict) -> bool:
        return self.left.interpret(context) and self.right.interpret(context)

class Or(Expression):
    def __init__(self, left: Expression, right: Expression):
        self.left = left
        self.right = right
    def interpret(self, context: dict) -> bool:
        return self.left.interpret(context) or self.right.interpret(context)

class Not(Expression):
    def __init__(self, expr: Expression):
        self.expr = expr
    def interpret(self, context: dict) -> bool:
        return not self.expr.interpret(context)

# Rule: (is_premium AND has_subscription) OR is_admin
rule = Or(
    And(Variable("is_premium"), Variable("has_subscription")),
    Variable("is_admin")
)

# Evaluate for different users
print(rule.interpret({"is_premium": True, "has_subscription": True}))  # True
print(rule.interpret({"is_premium": True, "has_subscription": False})) # False
print(rule.interpret({"is_admin": True}))                               # True
```

---

## 18.25 Beyond GoF: Modern Patterns

### Repository Pattern

```python
from abc import ABC, abstractmethod
from typing import Optional, List

class User:
    def __init__(self, user_id: str, name: str, email: str):
        self.user_id = user_id
        self.name = name
        self.email = email

class UserRepository(ABC):
    """Abstracts data access from business logic."""
    @abstractmethod
    def find_by_id(self, user_id: str) -> Optional[User]: pass
    @abstractmethod
    def find_by_email(self, email: str) -> Optional[User]: pass
    @abstractmethod
    def save(self, user: User) -> None: pass
    @abstractmethod
    def delete(self, user_id: str) -> None: pass
    @abstractmethod
    def find_all(self) -> List[User]: pass

class InMemoryUserRepository(UserRepository):
    def __init__(self):
        self._store = {}
    def find_by_id(self, user_id):
        return self._store.get(user_id)
    def find_by_email(self, email):
        return next((u for u in self._store.values() if u.email == email), None)
    def save(self, user):
        self._store[user.user_id] = user
    def delete(self, user_id):
        self._store.pop(user_id, None)
    def find_all(self):
        return list(self._store.values())

class PostgresUserRepository(UserRepository):
    def __init__(self, connection):
        self.conn = connection
    def find_by_id(self, user_id):
        # cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        pass
    # ... same interface, different storage

# Business logic decoupled from storage
class UserService:
    def __init__(self, repo: UserRepository):  # Injected
        self.repo = repo
    def register(self, user_id, name, email):
        if self.repo.find_by_email(email):
            raise ValueError("Email already registered")
        self.repo.save(User(user_id, name, email))
```

### Dependency Injection Container

```python
class DIContainer:
    """Simple DI container with singleton and transient support."""
    def __init__(self):
        self._singletons = {}     # type → instance
        self._factories = {}      # type → factory_fn
        self._is_singleton = {}   # type → bool

    def register_singleton(self, interface, factory):
        self._factories[interface] = factory
        self._is_singleton[interface] = True

    def register_transient(self, interface, factory):
        self._factories[interface] = factory
        self._is_singleton[interface] = False

    def resolve(self, interface):
        if interface in self._singletons:
            return self._singletons[interface]
        factory = self._factories.get(interface)
        if not factory:
            raise KeyError(f"No registration for {interface}")
        instance = factory(self)  # Pass container for nested resolution
        if self._is_singleton.get(interface):
            self._singletons[interface] = instance
        return instance

# Usage
container = DIContainer()
container.register_singleton(UserRepository, lambda c: InMemoryUserRepository())
container.register_transient(UserService, lambda c: UserService(c.resolve(UserRepository)))

service = container.resolve(UserService)  # Auto-wired
```

### Null Object Pattern

```python
class Logger(ABC):
    @abstractmethod
    def log(self, message: str): pass

class ConsoleLogger(Logger):
    def log(self, message):
        print(f"[LOG] {message}")

class NullLogger(Logger):
    """Does nothing — eliminates null checks throughout codebase."""
    def log(self, message):
        pass  # No-op

class OrderProcessor:
    def __init__(self, logger: Logger = None):
        self.logger = logger or NullLogger()  # Never None

    def process(self, order_id: str):
        self.logger.log(f"Processing {order_id}")  # No null check needed
        # ... business logic ...
        self.logger.log(f"Completed {order_id}")
```

### Object Pool Pattern

```python
import threading
from collections import deque

class ObjectPool:
    """Reuse expensive objects (DB connections, threads)."""
    def __init__(self, factory, max_size=10):
        self._factory = factory
        self._max_size = max_size
        self._pool = deque()
        self._in_use = 0
        self._lock = threading.Lock()
        self._available = threading.Condition(self._lock)

    def acquire(self):
        with self._available:
            while not self._pool and self._in_use >= self._max_size:
                self._available.wait()  # Block until one is returned
            if self._pool:
                obj = self._pool.popleft()
            else:
                obj = self._factory()
            self._in_use += 1
            return obj

    def release(self, obj):
        with self._available:
            self._in_use -= 1
            self._pool.append(obj)
            self._available.notify()

    # Context manager for convenience
    def __call__(self):
        return PooledObject(self)

class PooledObject:
    def __init__(self, pool):
        self.pool = pool
        self.obj = None
    def __enter__(self):
        self.obj = self.pool.acquire()
        return self.obj
    def __exit__(self, *args):
        self.pool.release(self.obj)

# Usage
db_pool = ObjectPool(lambda: create_db_connection(), max_size=20)
with db_pool() as conn:
    conn.execute("SELECT ...")
# Connection automatically returned to pool
```

---

## 18.26 Pattern Selection Guide

```
Problem                              │ Pattern
─────────────────────────────────────│──────────────────
Create objects without specifying     │ Factory / Abstract Factory
Complex object construction           │ Builder
Single shared instance                │ Singleton
Clone expensive objects               │ Prototype
Incompatible interfaces               │ Adapter
Abstraction × Implementation          │ Bridge
Tree structures (part-whole)          │ Composite
Add behavior dynamically              │ Decorator
Simplify complex subsystem            │ Facade
Share fine-grained objects            │ Flyweight
Control access / lazy loading         │ Proxy
Notify dependents of change           │ Observer
Swap algorithms at runtime            │ Strategy
Encapsulate requests (undo/queue)     │ Command
Behavior changes with state           │ State
Algorithm skeleton with varying steps │ Template Method
Pass request along handler chain      │ Chain of Responsibility
Sequential access to collection       │ Iterator
Reduce coupling via coordinator       │ Mediator
Capture/restore object state          │ Memento
Add operations to object structure    │ Visitor
Define and interpret a grammar        │ Interpreter
```

---

[← Previous: Classic System Designs](17-system-design-classic.md) | [Next: Object-Oriented Design →](19-object-oriented-design.md)
