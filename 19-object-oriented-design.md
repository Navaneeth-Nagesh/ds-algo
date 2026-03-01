# Chapter 19: Object-Oriented Design (Low-Level Design)

[← Previous: Design Patterns](18-design-patterns.md) | [Back to README →](README.md)

---

## 19.1 SOLID Principles

### Single Responsibility Principle (SRP)

```
A class should have ONE reason to change.

BAD:
class UserService:
    def create_user(self, data): ...
    def send_email(self, user): ...      # Email is a separate concern
    def generate_report(self, user): ...  # Reporting is a separate concern

GOOD:
class UserService:
    def create_user(self, data): ...

class EmailService:
    def send_email(self, user): ...

class ReportService:
    def generate_report(self, user): ...
```

### Open/Closed Principle (OCP)

```
Open for extension, closed for modification.
Add new behavior without changing existing code.
```

```python
from abc import ABC, abstractmethod

# BAD: Adding a new shape requires modifying AreaCalculator
class AreaCalculatorBad:
    def calculate(self, shape):
        if shape.type == "circle":
            return 3.14 * shape.radius ** 2
        elif shape.type == "rectangle":
            return shape.width * shape.height
        # Must modify this class for every new shape!

# GOOD: New shapes extend without modifying existing code
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return 3.14159 * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    def area(self):
        return self.width * self.height

# Adding Triangle doesn't modify any existing class
class Triangle(Shape):
    def __init__(self, base, height):
        self.base = base
        self.height = height
    def area(self):
        return 0.5 * self.base * self.height

def total_area(shapes: list[Shape]) -> float:
    return sum(s.area() for s in shapes)
```

### Liskov Substitution Principle (LSP)

```
Subtypes must be substitutable for their base types.
If it looks like a duck but needs batteries, you have the wrong abstraction.
```

```python
# BAD: Square violates LSP when extending Rectangle
class RectangleBad:
    def __init__(self, width, height):
        self._width = width
        self._height = height

    def set_width(self, w): self._width = w
    def set_height(self, h): self._height = h
    def area(self): return self._width * self._height

class SquareBad(RectangleBad):
    def set_width(self, w):
        self._width = self._height = w  # Breaks expectations!
    def set_height(self, h):
        self._width = self._height = h

# Client code breaks:
def resize(rect: RectangleBad):
    rect.set_width(5)
    rect.set_height(10)
    assert rect.area() == 50  # FAILS for SquareBad! (100)

# GOOD: Separate abstractions
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    def area(self): return self.width * self.height

class Square(Shape):
    def __init__(self, side):
        self.side = side
    def area(self): return self.side ** 2
```

### Interface Segregation Principle (ISP)

```
Clients should not depend on interfaces they don't use.
Many specific interfaces > one fat interface.
```

```python
# BAD: One fat interface
class Worker(ABC):
    @abstractmethod
    def work(self): pass
    @abstractmethod
    def eat(self): pass
    @abstractmethod
    def sleep(self): pass

class Robot(Worker):
    def work(self): ...
    def eat(self): raise NotImplementedError  # Robot doesn't eat!
    def sleep(self): raise NotImplementedError

# GOOD: Segregated interfaces
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Feedable(ABC):
    @abstractmethod
    def eat(self): pass

class Human(Workable, Feedable):
    def work(self): print("Working")
    def eat(self): print("Eating")

class Robot(Workable):  # Only implements what it needs
    def work(self): print("Working 24/7")
```

### Dependency Inversion Principle (DIP)

```
High-level modules should not depend on low-level modules.
Both should depend on abstractions.
```

```python
# BAD: High-level depends on low-level
class MySQLDatabase:
    def insert(self, table, data): ...

class UserServiceBad:
    def __init__(self):
        self.db = MySQLDatabase()  # Tight coupling to MySQL!

# GOOD: Both depend on abstraction
class Database(ABC):
    @abstractmethod
    def insert(self, table: str, data: dict): pass
    @abstractmethod
    def find(self, table: str, query: dict) -> list: pass

class MySQLDatabase(Database):
    def insert(self, table, data): print(f"MySQL INSERT into {table}")
    def find(self, table, query): return [{"id": 1}]

class MongoDatabase(Database):
    def insert(self, table, data): print(f"Mongo INSERT into {table}")
    def find(self, table, query): return [{"_id": 1}]

class UserService:
    def __init__(self, db: Database):  # Depends on abstraction
        self.db = db

    def create_user(self, data):
        self.db.insert("users", data)

# Can swap implementations without changing UserService
service = UserService(MySQLDatabase())
service = UserService(MongoDatabase())  # Easy swap!
```

---

## 19.2 UML Class Diagram Basics

```
Class Representation:
  ┌──────────────────────┐
  │      ClassName        │
  ├──────────────────────┤
  │ - private_attr: Type  │     -  private
  │ + public_attr: Type   │     +  public
  │ # protected_attr: Type│     #  protected
  ├──────────────────────┤
  │ + method(): ReturnType│
  │ - helper(): void      │
  └──────────────────────┘

Relationships:
  ───────→   Association       (A uses B)
  ──────◇→   Aggregation       (A has B, B can exist without A)
  ──────◆→   Composition       (A has B, B can't exist without A)
  ──────▷    Inheritance        (A is-a B)
  - - - -▷   Implementation     (A implements interface B)
  - - - -→   Dependency         (A depends on B)

Multiplicity:
  1      Exactly one
  0..1   Zero or one
  *      Zero or more
  1..*   One or more
  3..5   Three to five
```

---

## 19.3 Design: Parking Lot System

### Requirements

```
- Multiple levels, each with spots of different sizes (small, medium, large)
- Vehicle types: motorcycle, car, bus
- Assign nearest available spot
- Track parked vehicles, calculate fees
- Entry/exit gates
```

### Class Design

```python
from abc import ABC, abstractmethod
from enum import Enum
from datetime import datetime
from typing import Optional, List

class VehicleType(Enum):
    MOTORCYCLE = 1
    CAR = 2
    BUS = 3

class SpotSize(Enum):
    SMALL = 1     # motorcycle
    MEDIUM = 2    # car
    LARGE = 3     # bus

class Vehicle:
    def __init__(self, license_plate: str, vehicle_type: VehicleType):
        self.license_plate = license_plate
        self.vehicle_type = vehicle_type

    def get_required_spot_size(self) -> SpotSize:
        mapping = {
            VehicleType.MOTORCYCLE: SpotSize.SMALL,
            VehicleType.CAR: SpotSize.MEDIUM,
            VehicleType.BUS: SpotSize.LARGE,
        }
        return mapping[self.vehicle_type]

class ParkingSpot:
    def __init__(self, spot_id: str, size: SpotSize, level: int):
        self.spot_id = spot_id
        self.size = size
        self.level = level
        self.vehicle: Optional[Vehicle] = None

    @property
    def is_available(self) -> bool:
        return self.vehicle is None

    def can_fit(self, vehicle: Vehicle) -> bool:
        return self.is_available and self.size.value >= vehicle.get_required_spot_size().value

    def park(self, vehicle: Vehicle):
        if not self.can_fit(vehicle):
            raise ValueError("Cannot park here")
        self.vehicle = vehicle

    def remove_vehicle(self) -> Vehicle:
        v = self.vehicle
        self.vehicle = None
        return v

class ParkingTicket:
    def __init__(self, vehicle: Vehicle, spot: ParkingSpot):
        self.ticket_id = f"T-{id(self)}"
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = datetime.now()
        self.exit_time: Optional[datetime] = None
        self.amount_paid: float = 0.0

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, ticket: ParkingTicket) -> float:
        pass

class HourlyPricing(PricingStrategy):
    RATES = {
        VehicleType.MOTORCYCLE: 1.0,
        VehicleType.CAR: 2.0,
        VehicleType.BUS: 5.0,
    }

    def calculate(self, ticket: ParkingTicket) -> float:
        duration = ticket.exit_time - ticket.entry_time
        hours = max(1, int(duration.total_seconds() / 3600) + 1)
        rate = self.RATES[ticket.vehicle.vehicle_type]
        return hours * rate

class ParkingLevel:
    def __init__(self, level_number: int, spots: List[ParkingSpot]):
        self.level_number = level_number
        self.spots = spots

    def find_available_spot(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        for spot in self.spots:
            if spot.can_fit(vehicle):
                return spot
        return None

    def available_count(self) -> dict:
        counts = {}
        for spot in self.spots:
            if spot.is_available:
                counts[spot.size] = counts.get(spot.size, 0) + 1
        return counts

class ParkingLot:
    _instance = None

    def __init__(self, name: str, levels: List[ParkingLevel],
                 pricing: PricingStrategy):
        self.name = name
        self.levels = levels
        self.pricing = pricing
        self.active_tickets: dict[str, ParkingTicket] = {}  # license → ticket

    def park_vehicle(self, vehicle: Vehicle) -> ParkingTicket:
        for level in self.levels:
            spot = level.find_available_spot(vehicle)
            if spot:
                spot.park(vehicle)
                ticket = ParkingTicket(vehicle, spot)
                self.active_tickets[vehicle.license_plate] = ticket
                return ticket
        raise Exception("Parking lot is full")

    def unpark_vehicle(self, license_plate: str) -> float:
        ticket = self.active_tickets.get(license_plate)
        if not ticket:
            raise ValueError("Vehicle not found")

        ticket.exit_time = datetime.now()
        amount = self.pricing.calculate(ticket)
        ticket.amount_paid = amount
        ticket.spot.remove_vehicle()
        del self.active_tickets[license_plate]
        return amount

    def get_availability(self) -> dict:
        total = {}
        for level in self.levels:
            total[level.level_number] = level.available_count()
        return total

# Usage
spots_l1 = ([ParkingSpot(f"L1-S{i}", SpotSize.SMALL, 1) for i in range(10)] +
            [ParkingSpot(f"L1-M{i}", SpotSize.MEDIUM, 1) for i in range(20)] +
            [ParkingSpot(f"L1-L{i}", SpotSize.LARGE, 1) for i in range(5)])

lot = ParkingLot("Downtown Parking",
                 [ParkingLevel(1, spots_l1)],
                 HourlyPricing())

car = Vehicle("ABC-123", VehicleType.CAR)
ticket = lot.park_vehicle(car)
print(f"Parked at {ticket.spot.spot_id}")
```

---

## 19.4 Design: Elevator System

### Requirements

```
- Multiple elevators in a building
- Passengers request from any floor (up/down)
- Efficient dispatching (minimize wait time)
- Handle capacity limits
```

```python
from enum import Enum
from typing import List, Optional
from collections import defaultdict
import heapq

class Direction(Enum):
    UP = 1
    DOWN = -1
    IDLE = 0

class ElevatorState(Enum):
    MOVING = "moving"
    STOPPED = "stopped"
    MAINTENANCE = "maintenance"

class Request:
    def __init__(self, floor: int, direction: Direction):
        self.floor = floor
        self.direction = direction

class Elevator:
    def __init__(self, elevator_id: int, capacity: int = 10,
                 min_floor: int = 0, max_floor: int = 20):
        self.id = elevator_id
        self.capacity = capacity
        self.min_floor = min_floor
        self.max_floor = max_floor
        self.current_floor = 0
        self.direction = Direction.IDLE
        self.state = ElevatorState.STOPPED
        self.passengers = 0
        self.destinations = set()  # Floors to visit
        self.up_stops = set()
        self.down_stops = set()

    @property
    def is_full(self) -> bool:
        return self.passengers >= self.capacity

    def add_destination(self, floor: int):
        self.destinations.add(floor)
        if floor > self.current_floor:
            self.up_stops.add(floor)
        else:
            self.down_stops.add(floor)

    def step(self):
        """Move one floor in current direction."""
        if self.direction == Direction.UP:
            self.current_floor += 1
            self.up_stops.discard(self.current_floor)
        elif self.direction == Direction.DOWN:
            self.current_floor -= 1
            self.down_stops.discard(self.current_floor)

        self.destinations.discard(self.current_floor)
        self._update_direction()

    def _update_direction(self):
        if self.direction == Direction.UP:
            if self.up_stops:
                return  # Continue up
            elif self.down_stops:
                self.direction = Direction.DOWN
            else:
                self.direction = Direction.IDLE
        elif self.direction == Direction.DOWN:
            if self.down_stops:
                return
            elif self.up_stops:
                self.direction = Direction.UP
            else:
                self.direction = Direction.IDLE
        else:  # IDLE
            if self.up_stops:
                self.direction = Direction.UP
            elif self.down_stops:
                self.direction = Direction.DOWN

    def should_stop(self) -> bool:
        return self.current_floor in self.destinations

class ElevatorDispatcher:
    """LOOK algorithm (elevator algorithm) with nearest elevator dispatch."""

    def __init__(self, elevators: List[Elevator]):
        self.elevators = elevators

    def dispatch(self, request: Request) -> Elevator:
        best = None
        best_score = float('inf')

        for elevator in self.elevators:
            if elevator.state == ElevatorState.MAINTENANCE:
                continue
            if elevator.is_full:
                continue

            score = self._calculate_score(elevator, request)
            if score < best_score:
                best_score = score
                best = elevator

        if best:
            best.add_destination(request.floor)
        return best

    def _calculate_score(self, elevator: Elevator, request: Request) -> int:
        """Lower score = better match."""
        distance = abs(elevator.current_floor - request.floor)

        # Same direction and on the way
        if elevator.direction == Direction.IDLE:
            return distance

        if (elevator.direction == Direction.UP and
            request.direction == Direction.UP and
            elevator.current_floor <= request.floor):
            return distance  # Best case: same direction, on the way

        if (elevator.direction == Direction.DOWN and
            request.direction == Direction.DOWN and
            elevator.current_floor >= request.floor):
            return distance

        # Wrong direction — will need to reverse
        return distance + 20  # Penalty

class ElevatorSystem:
    def __init__(self, num_elevators: int, num_floors: int):
        self.elevators = [
            Elevator(i, max_floor=num_floors)
            for i in range(num_elevators)
        ]
        self.dispatcher = ElevatorDispatcher(self.elevators)
        self.num_floors = num_floors

    def request_elevator(self, floor: int, direction: Direction) -> Elevator:
        request = Request(floor, direction)
        return self.dispatcher.dispatch(request)

    def select_floor(self, elevator_id: int, floor: int):
        self.elevators[elevator_id].add_destination(floor)

    def status(self) -> list:
        return [(e.id, e.current_floor, e.direction.name)
                for e in self.elevators]
```

---

## 19.5 Design: Library Management System

```python
from enum import Enum
from datetime import datetime, timedelta
from typing import Optional, List

class BookStatus(Enum):
    AVAILABLE = "available"
    CHECKED_OUT = "checked_out"
    RESERVED = "reserved"
    LOST = "lost"

class MemberStatus(Enum):
    ACTIVE = "active"
    SUSPENDED = "suspended"
    CLOSED = "closed"

class Book:
    def __init__(self, isbn: str, title: str, author: str, copies: int = 1):
        self.isbn = isbn
        self.title = title
        self.author = author
        self.total_copies = copies
        self.items: List['BookItem'] = []

class BookItem:
    def __init__(self, barcode: str, book: Book):
        self.barcode = barcode
        self.book = book
        self.status = BookStatus.AVAILABLE
        self.due_date: Optional[datetime] = None
        self.borrowed_by: Optional['Member'] = None

class Member:
    MAX_BOOKS = 5
    LOAN_PERIOD_DAYS = 14
    FINE_PER_DAY = 0.50

    def __init__(self, member_id: str, name: str, email: str):
        self.member_id = member_id
        self.name = name
        self.email = email
        self.status = MemberStatus.ACTIVE
        self.borrowed_books: List[BookItem] = []
        self.total_fines: float = 0.0

    def can_borrow(self) -> bool:
        return (self.status == MemberStatus.ACTIVE and
                len(self.borrowed_books) < self.MAX_BOOKS and
                self.total_fines == 0)

class LibrarySystem:
    def __init__(self):
        self.books: dict[str, Book] = {}          # isbn → Book
        self.items: dict[str, BookItem] = {}       # barcode → BookItem
        self.members: dict[str, Member] = {}       # member_id → Member

    def add_book(self, isbn: str, title: str, author: str, copies: int = 1):
        book = Book(isbn, title, author, copies)
        self.books[isbn] = book
        for i in range(copies):
            item = BookItem(f"{isbn}-{i+1}", book)
            book.items.append(item)
            self.items[item.barcode] = item

    def register_member(self, member_id: str, name: str, email: str):
        self.members[member_id] = Member(member_id, name, email)

    def checkout(self, member_id: str, barcode: str) -> bool:
        member = self.members.get(member_id)
        item = self.items.get(barcode)

        if not member or not item:
            return False
        if not member.can_borrow():
            return False
        if item.status != BookStatus.AVAILABLE:
            return False

        item.status = BookStatus.CHECKED_OUT
        item.borrowed_by = member
        item.due_date = datetime.now() + timedelta(days=Member.LOAN_PERIOD_DAYS)
        member.borrowed_books.append(item)
        return True

    def return_book(self, barcode: str) -> float:
        item = self.items.get(barcode)
        if not item or item.status != BookStatus.CHECKED_OUT:
            return 0.0

        fine = 0.0
        if datetime.now() > item.due_date:
            overdue_days = (datetime.now() - item.due_date).days
            fine = overdue_days * Member.FINE_PER_DAY
            item.borrowed_by.total_fines += fine

        item.borrowed_by.borrowed_books.remove(item)
        item.status = BookStatus.AVAILABLE
        item.borrowed_by = None
        item.due_date = None
        return fine

    def search(self, query: str) -> List[Book]:
        query_lower = query.lower()
        return [b for b in self.books.values()
                if query_lower in b.title.lower() or
                   query_lower in b.author.lower()]
```

---

## 19.6 Design: Chess Game

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Optional, List, Tuple

class Color(Enum):
    WHITE = "white"
    BLACK = "black"

class Piece(ABC):
    def __init__(self, color: Color):
        self.color = color
        self.has_moved = False

    @abstractmethod
    def symbol(self) -> str: pass

    @abstractmethod
    def valid_moves(self, pos: Tuple[int, int],
                    board: 'Board') -> List[Tuple[int, int]]: pass

    def _in_bounds(self, r, c):
        return 0 <= r < 8 and 0 <= c < 8

class King(Piece):
    def symbol(self): return "K" if self.color == Color.WHITE else "k"

    def valid_moves(self, pos, board):
        r, c = pos
        moves = []
        for dr in [-1, 0, 1]:
            for dc in [-1, 0, 1]:
                if dr == 0 and dc == 0:
                    continue
                nr, nc = r + dr, c + dc
                if self._in_bounds(nr, nc):
                    target = board.get(nr, nc)
                    if target is None or target.color != self.color:
                        moves.append((nr, nc))
        return moves

class Rook(Piece):
    def symbol(self): return "R" if self.color == Color.WHITE else "r"

    def valid_moves(self, pos, board):
        return self._slide(pos, board, [(0,1),(0,-1),(1,0),(-1,0)])

    def _slide(self, pos, board, directions):
        r, c = pos
        moves = []
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            while self._in_bounds(nr, nc):
                target = board.get(nr, nc)
                if target is None:
                    moves.append((nr, nc))
                elif target.color != self.color:
                    moves.append((nr, nc))
                    break
                else:
                    break
                nr, nc = nr + dr, nc + dc
        return moves

class Knight(Piece):
    def symbol(self): return "N" if self.color == Color.WHITE else "n"

    def valid_moves(self, pos, board):
        r, c = pos
        moves = []
        for dr, dc in [(-2,-1),(-2,1),(-1,-2),(-1,2),
                        (1,-2),(1,2),(2,-1),(2,1)]:
            nr, nc = r + dr, c + dc
            if self._in_bounds(nr, nc):
                target = board.get(nr, nc)
                if target is None or target.color != self.color:
                    moves.append((nr, nc))
        return moves

class Pawn(Piece):
    def symbol(self): return "P" if self.color == Color.WHITE else "p"

    def valid_moves(self, pos, board):
        r, c = pos
        moves = []
        direction = -1 if self.color == Color.WHITE else 1

        # Forward one
        nr = r + direction
        if self._in_bounds(nr, c) and board.get(nr, c) is None:
            moves.append((nr, c))
            # Forward two (first move)
            if not self.has_moved:
                nr2 = r + 2 * direction
                if self._in_bounds(nr2, c) and board.get(nr2, c) is None:
                    moves.append((nr2, c))

        # Diagonal captures
        for dc in [-1, 1]:
            nr, nc = r + direction, c + dc
            if self._in_bounds(nr, nc):
                target = board.get(nr, nc)
                if target and target.color != self.color:
                    moves.append((nr, nc))
        return moves

# Bishop and Queen follow similar sliding patterns as Rook
class Bishop(Piece):
    def symbol(self): return "B" if self.color == Color.WHITE else "b"
    def valid_moves(self, pos, board):
        return Rook._slide(self, pos, board, [(1,1),(1,-1),(-1,1),(-1,-1)])

class Queen(Piece):
    def symbol(self): return "Q" if self.color == Color.WHITE else "q"
    def valid_moves(self, pos, board):
        return Rook._slide(self, pos, board,
            [(0,1),(0,-1),(1,0),(-1,0),(1,1),(1,-1),(-1,1),(-1,-1)])

class Board:
    def __init__(self):
        self.grid = [[None]*8 for _ in range(8)]
        self._setup()

    def _setup(self):
        # Place pieces in standard chess positions
        order = [Rook, Knight, Bishop, Queen, King, Bishop, Knight, Rook]
        for c, piece_cls in enumerate(order):
            self.grid[7][c] = piece_cls(Color.WHITE)
            self.grid[0][c] = piece_cls(Color.BLACK)
        for c in range(8):
            self.grid[6][c] = Pawn(Color.WHITE)
            self.grid[1][c] = Pawn(Color.BLACK)

    def get(self, r, c) -> Optional[Piece]:
        return self.grid[r][c]

    def move(self, fr: Tuple[int,int], to: Tuple[int,int]) -> Optional[Piece]:
        piece = self.grid[fr[0]][fr[1]]
        captured = self.grid[to[0]][to[1]]
        self.grid[to[0]][to[1]] = piece
        self.grid[fr[0]][fr[1]] = None
        piece.has_moved = True
        return captured

class ChessGame:
    def __init__(self):
        self.board = Board()
        self.current_turn = Color.WHITE
        self.move_history = []
        self.is_over = False
        self.winner = None

    def make_move(self, fr: Tuple[int,int], to: Tuple[int,int]) -> bool:
        piece = self.board.get(*fr)
        if not piece or piece.color != self.current_turn:
            return False

        if to not in piece.valid_moves(fr, self.board):
            return False

        captured = self.board.move(fr, to)
        self.move_history.append((fr, to, captured))

        if isinstance(captured, King):
            self.is_over = True
            self.winner = self.current_turn

        self.current_turn = (Color.BLACK if self.current_turn == Color.WHITE
                             else Color.WHITE)
        return True
```

---

## 19.7 Design: Vending Machine

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Optional

class Item:
    def __init__(self, name: str, price: float, quantity: int):
        self.name = name
        self.price = price
        self.quantity = quantity

# State pattern for vending machine
class VendingState(ABC):
    @abstractmethod
    def insert_money(self, machine: 'VendingMachine', amount: float): pass
    @abstractmethod
    def select_item(self, machine: 'VendingMachine', code: str): pass
    @abstractmethod
    def dispense(self, machine: 'VendingMachine'): pass
    @abstractmethod
    def cancel(self, machine: 'VendingMachine'): pass

class IdleState(VendingState):
    def insert_money(self, machine, amount):
        machine.balance += amount
        print(f"Inserted ${amount:.2f}. Balance: ${machine.balance:.2f}")
        machine.state = HasMoneyState()

    def select_item(self, machine, code):
        print("Please insert money first")

    def dispense(self, machine):
        print("Please insert money and select an item")

    def cancel(self, machine):
        print("Nothing to cancel")

class HasMoneyState(VendingState):
    def insert_money(self, machine, amount):
        machine.balance += amount
        print(f"Added ${amount:.2f}. Balance: ${machine.balance:.2f}")

    def select_item(self, machine, code):
        item = machine.inventory.get(code)
        if not item:
            print(f"Invalid code: {code}")
            return
        if item.quantity <= 0:
            print(f"{item.name} is sold out")
            return
        if machine.balance < item.price:
            print(f"Need ${item.price - machine.balance:.2f} more")
            return

        machine.selected_item = code
        machine.state = DispensingState()
        machine.state.dispense(machine)

    def dispense(self, machine):
        print("Please select an item first")

    def cancel(self, machine):
        refund = machine.balance
        machine.balance = 0
        machine.state = IdleState()
        print(f"Refunded ${refund:.2f}")

class DispensingState(VendingState):
    def insert_money(self, machine, amount):
        print("Please wait, dispensing...")

    def select_item(self, machine, code):
        print("Please wait, dispensing...")

    def dispense(self, machine):
        item = machine.inventory[machine.selected_item]
        item.quantity -= 1
        change = machine.balance - item.price
        machine.balance = 0
        machine.selected_item = None
        print(f"Dispensed: {item.name}")
        if change > 0:
            print(f"Change: ${change:.2f}")
        machine.state = IdleState()

    def cancel(self, machine):
        print("Cannot cancel while dispensing")

class VendingMachine:
    def __init__(self):
        self.inventory: dict[str, Item] = {}
        self.balance: float = 0.0
        self.selected_item: Optional[str] = None
        self.state: VendingState = IdleState()

    def add_item(self, code: str, item: Item):
        self.inventory[code] = item

    def insert_money(self, amount: float):
        self.state.insert_money(self, amount)

    def select_item(self, code: str):
        self.state.select_item(self, code)

    def cancel(self):
        self.state.cancel(self)

# Usage
vm = VendingMachine()
vm.add_item("A1", Item("Cola", 1.50, 10))
vm.add_item("A2", Item("Chips", 2.00, 5))
vm.add_item("B1", Item("Water", 1.00, 20))

vm.select_item("A1")    # Please insert money first
vm.insert_money(1.00)   # Inserted $1.00
vm.insert_money(1.00)   # Added $1.00. Balance: $2.00
vm.select_item("A1")    # Dispensed: Cola, Change: $0.50
```

---

## 19.8 Design: Hotel Booking System

```python
from datetime import date, timedelta
from enum import Enum
from typing import List, Optional

class RoomType(Enum):
    SINGLE = "single"
    DOUBLE = "double"
    SUITE = "suite"

class BookingStatus(Enum):
    CONFIRMED = "confirmed"
    CHECKED_IN = "checked_in"
    CHECKED_OUT = "checked_out"
    CANCELLED = "cancelled"

class Room:
    def __init__(self, number: str, room_type: RoomType,
                 price_per_night: float, floor: int):
        self.number = number
        self.room_type = room_type
        self.price_per_night = price_per_night
        self.floor = floor

class Guest:
    def __init__(self, guest_id: str, name: str, email: str, phone: str):
        self.guest_id = guest_id
        self.name = name
        self.email = email
        self.phone = phone

class Booking:
    _counter = 0

    def __init__(self, guest: Guest, room: Room,
                 check_in: date, check_out: date):
        Booking._counter += 1
        self.booking_id = f"BK-{Booking._counter:06d}"
        self.guest = guest
        self.room = room
        self.check_in = check_in
        self.check_out = check_out
        self.status = BookingStatus.CONFIRMED
        self.total_amount = self._calculate_total()

    def _calculate_total(self) -> float:
        nights = (self.check_out - self.check_in).days
        return nights * self.room.price_per_night

    @property
    def nights(self) -> int:
        return (self.check_out - self.check_in).days

class HotelBookingSystem:
    def __init__(self, name: str):
        self.name = name
        self.rooms: dict[str, Room] = {}
        self.guests: dict[str, Guest] = {}
        self.bookings: List[Booking] = []

    def add_room(self, room: Room):
        self.rooms[room.number] = room

    def register_guest(self, guest: Guest):
        self.guests[guest.guest_id] = guest

    def search_available_rooms(self, room_type: RoomType,
                                check_in: date, check_out: date) -> List[Room]:
        booked_rooms = set()
        for booking in self.bookings:
            if booking.status in (BookingStatus.CONFIRMED, BookingStatus.CHECKED_IN):
                # Check date overlap
                if (booking.check_in < check_out and
                    booking.check_out > check_in):
                    booked_rooms.add(booking.room.number)

        return [r for r in self.rooms.values()
                if r.room_type == room_type and
                   r.number not in booked_rooms]

    def make_booking(self, guest_id: str, room_number: str,
                     check_in: date, check_out: date) -> Optional[Booking]:
        guest = self.guests.get(guest_id)
        room = self.rooms.get(room_number)

        if not guest or not room:
            return None

        # Verify room is available for these dates
        available = self.search_available_rooms(room.room_type, check_in, check_out)
        if room not in available:
            return None

        booking = Booking(guest, room, check_in, check_out)
        self.bookings.append(booking)
        return booking

    def cancel_booking(self, booking_id: str) -> bool:
        for booking in self.bookings:
            if booking.booking_id == booking_id:
                if booking.status == BookingStatus.CONFIRMED:
                    booking.status = BookingStatus.CANCELLED
                    return True
        return False

    def check_in(self, booking_id: str) -> bool:
        for booking in self.bookings:
            if (booking.booking_id == booking_id and
                booking.status == BookingStatus.CONFIRMED):
                booking.status = BookingStatus.CHECKED_IN
                return True
        return False

    def check_out(self, booking_id: str) -> float:
        for booking in self.bookings:
            if (booking.booking_id == booking_id and
                booking.status == BookingStatus.CHECKED_IN):
                booking.status = BookingStatus.CHECKED_OUT
                return booking.total_amount
        return 0.0
```

---

## 19.9 Design: Food Delivery System

```python
from abc import ABC, abstractmethod
from enum import Enum
from datetime import datetime
from typing import List, Optional
import math

class OrderStatus(Enum):
    PLACED = "placed"
    CONFIRMED = "confirmed"
    PREPARING = "preparing"
    READY = "ready"
    PICKED_UP = "picked_up"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Location:
    def __init__(self, lat: float, lon: float):
        self.lat = lat
        self.lon = lon

    def distance_to(self, other: 'Location') -> float:
        # Simplified distance (km)
        return math.sqrt((self.lat - other.lat)**2 +
                         (self.lon - other.lon)**2) * 111

class MenuItem:
    def __init__(self, item_id: str, name: str, price: float,
                 prep_time_min: int = 15):
        self.item_id = item_id
        self.name = name
        self.price = price
        self.prep_time_min = prep_time_min
        self.available = True

class Restaurant:
    def __init__(self, restaurant_id: str, name: str, location: Location):
        self.restaurant_id = restaurant_id
        self.name = name
        self.location = location
        self.menu: dict[str, MenuItem] = {}
        self.is_open = True
        self.rating = 4.0

    def add_menu_item(self, item: MenuItem):
        self.menu[item.item_id] = item

class Customer:
    def __init__(self, customer_id: str, name: str, location: Location):
        self.customer_id = customer_id
        self.name = name
        self.location = location
        self.addresses: List[Location] = [location]

class DeliveryAgent:
    def __init__(self, agent_id: str, name: str, location: Location):
        self.agent_id = agent_id
        self.name = name
        self.location = location
        self.is_available = True
        self.current_order: Optional['Order'] = None
        self.rating = 4.5

class OrderItem:
    def __init__(self, menu_item: MenuItem, quantity: int):
        self.menu_item = menu_item
        self.quantity = quantity
        self.subtotal = menu_item.price * quantity

class Order:
    _counter = 0

    def __init__(self, customer: Customer, restaurant: Restaurant,
                 items: List[OrderItem], delivery_address: Location):
        Order._counter += 1
        self.order_id = f"ORD-{Order._counter:06d}"
        self.customer = customer
        self.restaurant = restaurant
        self.items = items
        self.delivery_address = delivery_address
        self.status = OrderStatus.PLACED
        self.agent: Optional[DeliveryAgent] = None
        self.created_at = datetime.now()
        self.delivery_fee = 2.99

    @property
    def subtotal(self) -> float:
        return sum(item.subtotal for item in self.items)

    @property
    def total(self) -> float:
        return self.subtotal + self.delivery_fee

class NotificationService:
    def notify(self, user_id: str, message: str):
        print(f"[NOTIFY {user_id}] {message}")

class FoodDeliverySystem:
    def __init__(self):
        self.restaurants: dict[str, Restaurant] = {}
        self.customers: dict[str, Customer] = {}
        self.agents: dict[str, DeliveryAgent] = {}
        self.orders: dict[str, Order] = {}
        self.notifier = NotificationService()

    def search_restaurants(self, location: Location,
                           radius_km: float = 10) -> List[Restaurant]:
        nearby = []
        for r in self.restaurants.values():
            if r.is_open and r.location.distance_to(location) <= radius_km:
                nearby.append(r)
        return sorted(nearby, key=lambda r: r.rating, reverse=True)

    def place_order(self, customer_id: str, restaurant_id: str,
                    item_quantities: dict[str, int],
                    delivery_address: Location) -> Optional[Order]:
        customer = self.customers.get(customer_id)
        restaurant = self.restaurants.get(restaurant_id)
        if not customer or not restaurant or not restaurant.is_open:
            return None

        items = []
        for item_id, qty in item_quantities.items():
            menu_item = restaurant.menu.get(item_id)
            if not menu_item or not menu_item.available:
                return None
            items.append(OrderItem(menu_item, qty))

        order = Order(customer, restaurant, items, delivery_address)
        self.orders[order.order_id] = order

        self.notifier.notify(restaurant_id, f"New order {order.order_id}")
        return order

    def assign_agent(self, order_id: str) -> Optional[DeliveryAgent]:
        order = self.orders.get(order_id)
        if not order:
            return None

        # Find nearest available agent
        best_agent = None
        best_distance = float('inf')

        for agent in self.agents.values():
            if agent.is_available:
                dist = agent.location.distance_to(order.restaurant.location)
                if dist < best_distance:
                    best_distance = dist
                    best_agent = agent

        if best_agent:
            best_agent.is_available = False
            best_agent.current_order = order
            order.agent = best_agent
            order.status = OrderStatus.CONFIRMED
            self.notifier.notify(best_agent.agent_id,
                                f"New delivery: {order.order_id}")
        return best_agent

    def update_order_status(self, order_id: str, status: OrderStatus):
        order = self.orders.get(order_id)
        if order:
            order.status = status
            self.notifier.notify(order.customer.customer_id,
                                f"Order {order_id}: {status.value}")

            if status == OrderStatus.DELIVERED and order.agent:
                order.agent.is_available = True
                order.agent.current_order = None
```

---

## 19.10 Design: ATM System

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Optional
from datetime import datetime

class TransactionType(Enum):
    WITHDRAWAL = "withdrawal"
    DEPOSIT = "deposit"
    BALANCE_INQUIRY = "balance_inquiry"
    TRANSFER = "transfer"

class Account:
    def __init__(self, account_number: str, pin: str, balance: float = 0):
        self.account_number = account_number
        self._pin_hash = hash(pin)
        self.balance = balance
        self.daily_withdrawal = 0.0
        self.daily_limit = 1000.0

    def verify_pin(self, pin: str) -> bool:
        return hash(pin) == self._pin_hash

    def can_withdraw(self, amount: float) -> bool:
        return (amount <= self.balance and
                self.daily_withdrawal + amount <= self.daily_limit)

    def withdraw(self, amount: float) -> bool:
        if self.can_withdraw(amount):
            self.balance -= amount
            self.daily_withdrawal += amount
            return True
        return False

    def deposit(self, amount: float):
        self.balance += amount

class Transaction:
    _counter = 0

    def __init__(self, account: Account, txn_type: TransactionType,
                 amount: float = 0):
        Transaction._counter += 1
        self.txn_id = f"TXN-{Transaction._counter:08d}"
        self.account_number = account.account_number
        self.txn_type = txn_type
        self.amount = amount
        self.timestamp = datetime.now()
        self.success = False

class CashDispenser:
    def __init__(self):
        self.denominations = {100: 100, 50: 200, 20: 500, 10: 1000}

    def can_dispense(self, amount: float) -> bool:
        remaining = int(amount)
        for denom in sorted(self.denominations.keys(), reverse=True):
            count = min(remaining // denom, self.denominations[denom])
            remaining -= count * denom
        return remaining == 0

    def dispense(self, amount: float) -> dict:
        result = {}
        remaining = int(amount)
        for denom in sorted(self.denominations.keys(), reverse=True):
            count = min(remaining // denom, self.denominations[denom])
            if count > 0:
                result[denom] = count
                self.denominations[denom] -= count
                remaining -= count * denom
        return result

class ATM:
    def __init__(self, atm_id: str, bank_system: 'BankSystem'):
        self.atm_id = atm_id
        self.bank = bank_system
        self.cash_dispenser = CashDispenser()
        self.current_account: Optional[Account] = None
        self.transactions: list[Transaction] = []

    def authenticate(self, account_number: str, pin: str) -> bool:
        account = self.bank.get_account(account_number)
        if account and account.verify_pin(pin):
            self.current_account = account
            return True
        return False

    def check_balance(self) -> float:
        if not self.current_account:
            raise Exception("Not authenticated")
        txn = Transaction(self.current_account, TransactionType.BALANCE_INQUIRY)
        txn.success = True
        self.transactions.append(txn)
        return self.current_account.balance

    def withdraw(self, amount: float) -> Optional[dict]:
        if not self.current_account:
            raise Exception("Not authenticated")

        txn = Transaction(self.current_account, TransactionType.WITHDRAWAL, amount)

        if not self.cash_dispenser.can_dispense(amount):
            print("ATM cannot dispense this amount")
            self.transactions.append(txn)
            return None

        if self.current_account.withdraw(amount):
            bills = self.cash_dispenser.dispense(amount)
            txn.success = True
            self.transactions.append(txn)
            print(f"Dispensed: {bills}")
            return bills
        else:
            print("Insufficient funds or daily limit exceeded")
            self.transactions.append(txn)
            return None

    def deposit(self, amount: float):
        if not self.current_account:
            raise Exception("Not authenticated")
        self.current_account.deposit(amount)
        txn = Transaction(self.current_account, TransactionType.DEPOSIT, amount)
        txn.success = True
        self.transactions.append(txn)

    def logout(self):
        self.current_account = None
        print("Session ended. Please take your card.")

class BankSystem:
    def __init__(self):
        self.accounts: dict[str, Account] = {}

    def get_account(self, account_number: str) -> Optional[Account]:
        return self.accounts.get(account_number)

    def create_account(self, account_number: str, pin: str,
                       initial_balance: float = 0):
        self.accounts[account_number] = Account(account_number, pin,
                                                  initial_balance)
```

---

## 19.11 Design: Tic-Tac-Toe

```python
from enum import Enum
from typing import Optional, Tuple

class Mark(Enum):
    X = "X"
    O = "O"
    EMPTY = " "

class TicTacToe:
    def __init__(self, size: int = 3):
        self.size = size
        self.board = [[Mark.EMPTY] * size for _ in range(size)]
        self.current_player = Mark.X
        self.moves_count = 0
        self.winner: Optional[Mark] = None

        # O(1) win check using counters
        self.row_counts = {Mark.X: [0]*size, Mark.O: [0]*size}
        self.col_counts = {Mark.X: [0]*size, Mark.O: [0]*size}
        self.diag_counts = {Mark.X: 0, Mark.O: 0}
        self.anti_diag_counts = {Mark.X: 0, Mark.O: 0}

    def make_move(self, row: int, col: int) -> bool:
        if not self._is_valid(row, col):
            return False

        player = self.current_player
        self.board[row][col] = player
        self.moves_count += 1

        # Update counters
        self.row_counts[player][row] += 1
        self.col_counts[player][col] += 1
        if row == col:
            self.diag_counts[player] += 1
        if row + col == self.size - 1:
            self.anti_diag_counts[player] += 1

        # Check win in O(1)
        if (self.row_counts[player][row] == self.size or
            self.col_counts[player][col] == self.size or
            self.diag_counts[player] == self.size or
            self.anti_diag_counts[player] == self.size):
            self.winner = player

        self.current_player = Mark.O if player == Mark.X else Mark.X
        return True

    def _is_valid(self, row: int, col: int) -> bool:
        return (0 <= row < self.size and
                0 <= col < self.size and
                self.board[row][col] == Mark.EMPTY and
                self.winner is None)

    @property
    def is_draw(self) -> bool:
        return self.moves_count == self.size ** 2 and self.winner is None

    @property
    def is_over(self) -> bool:
        return self.winner is not None or self.is_draw

    def display(self):
        for row in self.board:
            print("|".join(m.value for m in row))
            print("-" * (self.size * 2 - 1))

game = TicTacToe()
game.make_move(0, 0)  # X
game.make_move(1, 1)  # O
game.make_move(0, 1)  # X
game.make_move(1, 0)  # O
game.make_move(0, 2)  # X wins!
game.display()
print(f"Winner: {game.winner}")
```

---

## 19.12 Design: LRU Cache (Production-Grade)

```python
import threading
from typing import Optional, Any

class DLLNode:
    __slots__ = ['key', 'value', 'prev', 'next', 'freq']

    def __init__(self, key=None, value=None):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None
        self.freq = 1

class ThreadSafeLRUCache:
    """Production-grade LRU cache with TTL support."""

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key → DLLNode
        # Sentinel nodes (avoid null checks)
        self.head = DLLNode()  # Most recent
        self.tail = DLLNode()  # Least recent
        self.head.next = self.tail
        self.tail.prev = self.head
        self.lock = threading.RLock()
        self.hits = 0
        self.misses = 0

    def get(self, key: str) -> Optional[Any]:
        with self.lock:
            if key in self.cache:
                node = self.cache[key]
                self._move_to_front(node)
                self.hits += 1
                return node.value
            self.misses += 1
            return None

    def put(self, key: str, value: Any):
        with self.lock:
            if key in self.cache:
                node = self.cache[key]
                node.value = value
                self._move_to_front(node)
                return

            if len(self.cache) >= self.capacity:
                self._evict()

            node = DLLNode(key, value)
            self.cache[key] = node
            self._add_to_front(node)

    def delete(self, key: str) -> bool:
        with self.lock:
            if key in self.cache:
                node = self.cache.pop(key)
                self._remove(node)
                return True
            return False

    def _add_to_front(self, node: DLLNode):
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove(self, node: DLLNode):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _move_to_front(self, node: DLLNode):
        self._remove(node)
        self._add_to_front(node)

    def _evict(self):
        lru = self.tail.prev
        self._remove(lru)
        del self.cache[lru.key]

    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0

    def __len__(self):
        return len(self.cache)
```

---

## 19.13 Design: Logger Framework

```python
from abc import ABC, abstractmethod
from enum import IntEnum
from datetime import datetime
from typing import List
import threading
import json

class LogLevel(IntEnum):
    DEBUG = 10
    INFO = 20
    WARNING = 30
    ERROR = 40
    CRITICAL = 50

class LogRecord:
    def __init__(self, level: LogLevel, message: str, **kwargs):
        self.level = level
        self.message = message
        self.timestamp = datetime.now()
        self.thread_name = threading.current_thread().name
        self.extra = kwargs

class LogHandler(ABC):
    def __init__(self, level: LogLevel = LogLevel.DEBUG):
        self.level = level
        self.formatter = None

    @abstractmethod
    def emit(self, record: LogRecord): pass

    def handle(self, record: LogRecord):
        if record.level >= self.level:
            self.emit(record)

class ConsoleHandler(LogHandler):
    def emit(self, record: LogRecord):
        msg = self._format(record)
        print(msg)

    def _format(self, record: LogRecord) -> str:
        return (f"[{record.timestamp.isoformat()}] "
                f"{record.level.name:8} "
                f"[{record.thread_name}] "
                f"{record.message}")

class FileHandler(LogHandler):
    def __init__(self, filename: str, level: LogLevel = LogLevel.DEBUG):
        super().__init__(level)
        self.filename = filename
        self._lock = threading.Lock()

    def emit(self, record: LogRecord):
        with self._lock:
            with open(self.filename, 'a') as f:
                f.write(json.dumps({
                    "timestamp": record.timestamp.isoformat(),
                    "level": record.level.name,
                    "message": record.message,
                    "thread": record.thread_name,
                    **record.extra
                }) + "\n")

class Logger:
    _instances = {}
    _lock = threading.Lock()

    def __init__(self, name: str, level: LogLevel = LogLevel.DEBUG):
        self.name = name
        self.level = level
        self.handlers: List[LogHandler] = []

    @classmethod
    def get_logger(cls, name: str) -> 'Logger':
        with cls._lock:
            if name not in cls._instances:
                cls._instances[name] = Logger(name)
            return cls._instances[name]

    def add_handler(self, handler: LogHandler):
        self.handlers.append(handler)

    def _log(self, level: LogLevel, message: str, **kwargs):
        if level >= self.level:
            record = LogRecord(level, message, logger=self.name, **kwargs)
            for handler in self.handlers:
                handler.handle(record)

    def debug(self, msg: str, **kwargs): self._log(LogLevel.DEBUG, msg, **kwargs)
    def info(self, msg: str, **kwargs): self._log(LogLevel.INFO, msg, **kwargs)
    def warning(self, msg: str, **kwargs): self._log(LogLevel.WARNING, msg, **kwargs)
    def error(self, msg: str, **kwargs): self._log(LogLevel.ERROR, msg, **kwargs)
    def critical(self, msg: str, **kwargs): self._log(LogLevel.CRITICAL, msg, **kwargs)

# Usage
logger = Logger.get_logger("app")
logger.add_handler(ConsoleHandler(LogLevel.INFO))
logger.add_handler(FileHandler("app.log", LogLevel.ERROR))

logger.info("Server started", port=8080)
logger.error("Database connection failed", db="postgres", retry=3)
```

---

## 19.15 Design: Movie Ticket Booking (BookMyShow)

```python
from enum import Enum
from datetime import datetime
from typing import List, Optional
import threading

class SeatType(Enum):
    REGULAR = "regular"
    PREMIUM = "premium"
    VIP = "vip"

class SeatStatus(Enum):
    AVAILABLE = "available"
    LOCKED = "locked"      # Temporarily held during booking
    BOOKED = "booked"

class Seat:
    def __init__(self, seat_id: str, row: str, number: int,
                 seat_type: SeatType, price: float):
        self.seat_id = seat_id
        self.row = row
        self.number = number
        self.seat_type = seat_type
        self.price = price
        self.status = SeatStatus.AVAILABLE
        self._lock = threading.Lock()

    def try_lock(self) -> bool:
        with self._lock:
            if self.status == SeatStatus.AVAILABLE:
                self.status = SeatStatus.LOCKED
                return True
            return False

    def confirm(self):
        with self._lock:
            self.status = SeatStatus.BOOKED

    def release(self):
        with self._lock:
            if self.status == SeatStatus.LOCKED:
                self.status = SeatStatus.AVAILABLE

class Movie:
    def __init__(self, movie_id: str, title: str, duration_min: int):
        self.movie_id = movie_id
        self.title = title
        self.duration_min = duration_min

class Screen:
    def __init__(self, screen_id: str, name: str, seats: List[Seat]):
        self.screen_id = screen_id
        self.name = name
        self.seats = {s.seat_id: s for s in seats}

class Show:
    def __init__(self, show_id: str, movie: Movie, screen: Screen,
                 start_time: datetime):
        self.show_id = show_id
        self.movie = movie
        self.screen = screen
        self.start_time = start_time
        # Each show has its own copy of seat availability
        self.seats = {sid: Seat(sid, s.row, s.number, s.seat_type, s.price)
                      for sid, s in screen.seats.items()}

    def get_available_seats(self) -> List[Seat]:
        return [s for s in self.seats.values() if s.status == SeatStatus.AVAILABLE]

class Booking:
    _counter = 0

    def __init__(self, user_id: str, show: Show, seats: List[Seat]):
        Booking._counter += 1
        self.booking_id = f"BK-{Booking._counter:08d}"
        self.user_id = user_id
        self.show = show
        self.seats = seats
        self.total = sum(s.price for s in seats)
        self.created_at = datetime.now()

class BookingService:
    LOCK_TIMEOUT_SEC = 300  # 5 minutes

    def __init__(self):
        self.bookings: dict[str, Booking] = {}

    def select_seats(self, show: Show, seat_ids: List[str]) -> bool:
        """Try to lock selected seats. All-or-nothing."""
        locked = []
        for sid in seat_ids:
            seat = show.seats.get(sid)
            if not seat or not seat.try_lock():
                # Rollback already-locked seats
                for s in locked:
                    s.release()
                return False
            locked.append(seat)
        return True

    def confirm_booking(self, user_id: str, show: Show,
                        seat_ids: List[str]) -> Optional[Booking]:
        seats = [show.seats[sid] for sid in seat_ids]
        for seat in seats:
            seat.confirm()
        booking = Booking(user_id, show, seats)
        self.bookings[booking.booking_id] = booking
        return booking
```

---

## 19.16 Design: In-Memory File System

```python
from abc import ABC, abstractmethod
from datetime import datetime
from typing import Optional, List

class FileSystemNode(ABC):
    def __init__(self, name: str, parent: Optional['Directory'] = None):
        self.name = name
        self.parent = parent
        self.created_at = datetime.now()
        self.modified_at = datetime.now()

    @abstractmethod
    def size(self) -> int: pass

    @property
    def path(self) -> str:
        parts = []
        node = self
        while node:
            parts.append(node.name)
            node = node.parent
        return "/".join(reversed(parts)) or "/"

class File(FileSystemNode):
    def __init__(self, name: str, parent=None):
        super().__init__(name, parent)
        self.content = ""

    def size(self) -> int:
        return len(self.content.encode('utf-8'))

    def write(self, data: str):
        self.content = data
        self.modified_at = datetime.now()

    def append(self, data: str):
        self.content += data
        self.modified_at = datetime.now()

    def read(self) -> str:
        return self.content

class Directory(FileSystemNode):
    def __init__(self, name: str, parent=None):
        super().__init__(name, parent)
        self.children: dict[str, FileSystemNode] = {}

    def size(self) -> int:
        return sum(c.size() for c in self.children.values())

    def add(self, node: FileSystemNode):
        if node.name in self.children:
            raise FileExistsError(f"{node.name} already exists")
        node.parent = self
        self.children[node.name] = node

    def remove(self, name: str):
        if name not in self.children:
            raise FileNotFoundError(name)
        del self.children[name]

    def list(self) -> List[str]:
        return sorted(self.children.keys())

class FileSystem:
    def __init__(self):
        self.root = Directory("")

    def _resolve(self, path: str) -> FileSystemNode:
        if path == "/":
            return self.root
        parts = path.strip("/").split("/")
        node = self.root
        for part in parts:
            if not isinstance(node, Directory) or part not in node.children:
                raise FileNotFoundError(path)
            node = node.children[part]
        return node

    def mkdir(self, path: str):
        parts = path.strip("/").split("/")
        node = self.root
        for part in parts:
            if part not in node.children:
                node.add(Directory(part))
            node = node.children[part]

    def create_file(self, path: str, content: str = ""):
        parts = path.strip("/").split("/")
        parent = self.root
        for part in parts[:-1]:
            if part not in parent.children:
                parent.add(Directory(part))
            parent = parent.children[part]
        f = File(parts[-1])
        f.write(content)
        parent.add(f)

    def read_file(self, path: str) -> str:
        node = self._resolve(path)
        if not isinstance(node, File):
            raise IsADirectoryError(path)
        return node.read()

    def ls(self, path: str = "/") -> List[str]:
        node = self._resolve(path)
        if isinstance(node, Directory):
            return node.list()
        return [node.name]

    def find(self, path: str, name: str) -> List[str]:
        """Recursive search under path."""
        results = []
        start = self._resolve(path)
        def dfs(node):
            if node.name == name:
                results.append(node.path)
            if isinstance(node, Directory):
                for child in node.children.values():
                    dfs(child)
        dfs(start)
        return results

# Usage
fs = FileSystem()
fs.mkdir("/home/user/docs")
fs.create_file("/home/user/docs/notes.txt", "Hello World")
print(fs.ls("/home/user/docs"))       # ['notes.txt']
print(fs.read_file("/home/user/docs/notes.txt"))  # Hello World
```

---

## 19.17 Design: Splitwise / Expense Sharing

```python
from enum import Enum
from typing import List, Dict
from collections import defaultdict
import heapq

class SplitType(Enum):
    EQUAL = "equal"
    EXACT = "exact"
    PERCENT = "percent"

class User:
    def __init__(self, user_id: str, name: str):
        self.user_id = user_id
        self.name = name

class Expense:
    _counter = 0

    def __init__(self, paid_by: str, amount: float,
                 participants: List[str], split_type: SplitType,
                 shares: Dict[str, float] = None, description: str = ""):
        Expense._counter += 1
        self.expense_id = f"EXP-{Expense._counter:06d}"
        self.paid_by = paid_by
        self.amount = amount
        self.description = description
        self.splits = self._calculate_splits(participants, split_type, shares)

    def _calculate_splits(self, participants, split_type, shares):
        splits = {}
        if split_type == SplitType.EQUAL:
            share = self.amount / len(participants)
            for uid in participants:
                splits[uid] = share
        elif split_type == SplitType.EXACT:
            splits = dict(shares)
        elif split_type == SplitType.PERCENT:
            for uid, pct in shares.items():
                splits[uid] = self.amount * pct / 100
        return splits

class SplitwiseService:
    def __init__(self):
        self.users: Dict[str, User] = {}
        self.expenses: List[Expense] = []
        # balances[A][B] > 0 means A owes B that amount
        self.balances: Dict[str, Dict[str, float]] = defaultdict(lambda: defaultdict(float))

    def add_user(self, user: User):
        self.users[user.user_id] = user

    def add_expense(self, expense: Expense):
        self.expenses.append(expense)
        payer = expense.paid_by
        for uid, share in expense.splits.items():
            if uid != payer:
                self.balances[uid][payer] += share
                self.balances[payer][uid] -= share

    def get_balance(self, user_id: str) -> Dict[str, float]:
        """Positive = owes, negative = owed."""
        return {uid: amt for uid, amt in self.balances[user_id].items() if abs(amt) > 0.01}

    def simplify_debts(self) -> List[tuple]:
        """Minimize number of transactions using greedy algorithm.
        Net amount per person, then greedily settle max creditor with max debtor."""
        net = defaultdict(float)
        for uid, debts in self.balances.items():
            for other, amt in debts.items():
                net[uid] += amt  # positive = owes net, negative = is owed net

        # Split into debtors (positive net) and creditors (negative net)
        debtors = []  # (amount, user_id) — people who owe
        creditors = []  # (-amount, user_id) — people who are owed
        for uid, amount in net.items():
            if amount > 0.01:
                debtors.append((amount, uid))
            elif amount < -0.01:
                creditors.append((-amount, uid))

        # Greedy: match largest debtor with largest creditor
        transactions = []
        debtors.sort(reverse=True)
        creditors.sort(reverse=True)
        i = j = 0
        while i < len(debtors) and j < len(creditors):
            settle = min(debtors[i][0], creditors[j][0])
            transactions.append((debtors[i][1], creditors[j][1], round(settle, 2)))
            debtors[i] = (debtors[i][0] - settle, debtors[i][1])
            creditors[j] = (creditors[j][0] - settle, creditors[j][1])
            if debtors[i][0] < 0.01:
                i += 1
            if creditors[j][0] < 0.01:
                j += 1
        return transactions

# Usage
svc = SplitwiseService()
svc.add_user(User("A", "Alice"))
svc.add_user(User("B", "Bob"))
svc.add_user(User("C", "Charlie"))

svc.add_expense(Expense("A", 300, ["A", "B", "C"], SplitType.EQUAL))
svc.add_expense(Expense("B", 120, ["A", "B"], SplitType.EQUAL))
print(svc.simplify_debts())  # Minimal transactions to settle all
```

---

## 19.14 LLD Quick Reference

```
Problem              │ Key Patterns         │ Key Classes
─────────────────────│──────────────────────│──────────────────────
Parking Lot          │ Strategy, Singleton  │ ParkingLot, Level, Spot, Vehicle, Ticket
Elevator System      │ State, Strategy      │ Elevator, Dispatcher, Request, Controller
Library System       │ Observer, Strategy   │ Library, Book, Member, Loan, Search
Chess Game           │ Template, Strategy   │ Board, Piece(6 types), Game, Move
Vending Machine      │ State               │ VendingMachine, Item, Coin, State
Hotel Booking        │ Observer, Strategy   │ Hotel, Room, Guest, Booking, Payment
Food Delivery        │ Observer, Strategy   │ Restaurant, Order, Agent, Customer
ATM                  │ State, Chain of Resp │ ATM, Account, Transaction, Dispenser
Tic-Tac-Toe          │ —                   │ Board, Player, Game
LRU Cache            │ —                   │ Cache, DLLNode
Logger               │ Observer, Singleton  │ Logger, Handler, Record, Formatter
Movie Ticket Booking │ State, Strategy      │ Show, Screen, Seat, Booking
In-Memory File System│ Composite           │ File, Directory, FileSystem
Expense Sharing      │ Strategy            │ Expense, Balance, DebtSimplifier

Common LLD Approaches:
  1. Identify entities and relationships
  2. Define interfaces/abstract classes
  3. Apply SOLID principles
  4. Apply design patterns where appropriate
  5. Handle concurrency (if multi-threaded)
  6. Consider error handling and edge cases
  7. Think about extensibility
```

---

[← Previous: Design Patterns](18-design-patterns.md) | [Back to README →](README.md)
