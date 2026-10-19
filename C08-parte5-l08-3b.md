(parte 5 di 8: L08.3 — Metaprogrammazione (esperimenti e10–e12) — file C08-parte5-l08-3b.md)

### Esperimento e10 — leggere lo schema dall'esterno

Il primo passo di ogni metaprogramma è **leggere** la dichiarazione. Stessa domanda ai quattro linguaggi: «quali campi ha `Product`, di che tipo, con quali regole?»

```python
import sys
from dataclasses import fields
from typing import get_type_hints

sys.path.insert(0, "../prog")
from main import Product  # the real class of the program

print("dataclasses.fields:", [f.name for f in fields(Product)])
for name, hint in get_type_hints(Product, include_extras=True).items():
    print(f"{name:13} {hint}")
```

```php
<?php
declare(strict_types=1);
// T2 - the rules: attribute classes with a check() method. An attribute is inert DATA until Reflection reads it.
interface Rule
{
    public function check(mixed $value): bool;
}

function shapeOk(string $maskChar, string $c): bool
{
    return match ($maskChar) {
        'A' => $c >= 'A' && $c <= 'Z',
        '9' => $c >= '0' && $c <= '9',
        default => $maskChar === $c,
    };
}

#[Attribute(Attribute::TARGET_PARAMETER)]
final class Between implements Rule
{
    public function __construct(private int $lo, private int $hi)
    {
    }

    public function check(mixed $value): bool
    {
        return is_int($value) && $this->lo <= $value && $value <= $this->hi;
    }
}

#[Attribute(Attribute::TARGET_PARAMETER)]
final class Length implements Rule
{
    public function __construct(private int $lo, private int $hi)
    {
    }

    public function check(mixed $value): bool
    {
        return is_string($value) && $this->lo <= mb_strlen($value) && mb_strlen($value) <= $this->hi;
    }
}

#[Attribute(Attribute::TARGET_PARAMETER)]
final class Shape implements Rule
{
    /** @param string $mask "A" = uppercase ASCII letter, "9" = ASCII digit, anything else = literal */
    public function __construct(private string $mask)
    {
    }

    public function check(mixed $value): bool
    {
        if (!is_string($value) || strlen($value) !== strlen($this->mask)) {
            return false;
        }
        for ($i = 0; $i < strlen($value); $i++) {
            if (!shapeOk($this->mask[$i], $value[$i])) {
                return false;
            }
        }
        return true;
    }
}

// T2 - the schema: ONE declaration holds the type and the rules of every column.
final readonly class Product
{
    public function __construct(
        #[Shape('AA999')] public string $sku,
        #[Length(1, 30)] public string $name,
        #[Length(1, 12)] public string $category,
        #[Between(0, 9999)] public int $qty,
        #[Between(1, 999999)] public int $price,
        #[Between(0, 90)] public ?int $max_discount,
    ) {
    }

    public function value(): int
    {
        return $this->qty * $this->price;
    }
}

foreach ((new ReflectionClass(Product::class))->getConstructor()->getParameters() as $p) {
    $t = $p->getType();
    $attrs = array_map(fn($a) => $a->getName() . '(' . implode(', ', array_map('json_encode', $a->getArguments())) . ')', $p->getAttributes());
    printf("%-13s %-8s %s\n", $p->getName(), ($t->allowsNull() ? '?' : '') . $t->getName(), implode(' ', $attrs));
}
```

```go
package main

import (
	"fmt"
	"reflect"
)

// Product is T2 - the schema: ONE declaration holds the type and, in the struct tags, the rules of every column.
// The fields are exported because package reflect can only write to exported fields (e12).
type Product struct {
	SKU         string `col:"sku" rule:"shape=AA999"`
	Name        string `col:"name" rule:"length=1:30"`
	Category    string `col:"category" rule:"length=1:12"`
	Qty         int    `col:"qty" rule:"between=0:9999"`
	Price       int    `col:"price" rule:"between=1:999999"`
	MaxDiscount *int   `col:"max_discount" rule:"between=0:90"` // a nil pointer is "absent"
}

func (p Product) Value() int { return p.Qty * p.Price }

func main() {
	t := reflect.TypeFor[Product]()
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Printf("%-12s %-8s col=%q rule=%q\n", f.Name, f.Type, f.Tag.Get("col"), f.Tag.Get("rule"))
	}
}
```

```rust
use std::any::type_name;

macro_rules! describe {
    (struct $name:ident { $( $field:ident : $ty:ty ),+ $(,)? }) => {
        #[allow(dead_code)]
        struct $name { $( $field: $ty ),+ }
        impl $name {
            // generated at COMPILE time: the compiler cannot be asked about fields at run time
            const FIELDS: &'static [(&'static str, &'static str)] = &[ $( (stringify!($field), stringify!($ty)) ),+ ];
        }
    };
}

describe! {
    struct Product {
        sku: String,
        qty: i64,
        max_discount: Option<i64>,
    }
}

fn main() {
    println!("type_name: {}", type_name::<Product>());
    for (name, ty) in Product::FIELDS {
        println!("{name:13} {ty}");
    }
}
```

```text
### e10.py
dataclasses.fields: ['sku', 'name', 'category', 'qty', 'price', 'max_discount']
sku           typing.Annotated[str, Shape(mask='AA999')]
name          typing.Annotated[str, Length(lo=1, hi=30)]
category      typing.Annotated[str, Length(lo=1, hi=12)]
qty           typing.Annotated[int, Between(lo=0, hi=9999)]
price         typing.Annotated[int, Between(lo=1, hi=999999)]
max_discount  typing.Annotated[int | None, Between(lo=0, hi=90)]
[exit 0]
### e10.php
sku           string   Shape("AA999")
name          string   Length(1, 30)
category      string   Length(1, 12)
qty           int      Between(0, 9999)
price         int      Between(1, 999999)
max_discount  ?int     Between(0, 90)
[exit 0]
### e10.go
SKU          string   col="sku" rule="shape=AA999"
Name         string   col="name" rule="length=1:30"
Category     string   col="category" rule="length=1:12"
Qty          int      col="qty" rule="between=0:9999"
Price        int      col="price" rule="between=1:999999"
MaxDiscount  *int     col="max_discount" rule="between=0:90"
[exit 0]
### e10.rs
type_name: e10::Product
sku           String
qty           i64
max_discount  Option<i64>
[exit 0]
```

Python, PHP e Go rispondono a **runtime**, leggendo i metadati. Rust risponde `type_name` e poche righe: il compilatore *non* ha un'API per elencare i campi di una struct dentro il programma; la lista `FIELDS` è una costante che **la macro ha scritto** a partire dalla stessa dichiarazione. Stessa informazione, ma in Rust è stata «copiata» a compile time, in Python/PHP/Go è stata «chiesta» a run time.

### Esperimento e11 — l'errore di battitura nella regola

Nello schema, `Between` diventa `Betwen` (una lettera sbagliata). Chi se ne accorge, e quando?

```python
import sys
import types
from collections import OrderedDict
from collections.abc import Callable, Hashable, Iterable, Iterator, Sequence
from dataclasses import dataclass
from typing import Annotated, Any, Protocol, get_args, get_origin, get_type_hints


# T2 - the rules: small objects with a check() method. They are only DATA attached to the fields.
def _shape_ok(mask_char: str, c: str) -> bool:
    if mask_char == "A":
        return "A" <= c <= "Z"
    if mask_char == "9":
        return "0" <= c <= "9"
    return mask_char == c


@dataclass(frozen=True)
class Between:
    lo: int
    hi: int

    def check(self, value: object) -> bool:
        return isinstance(value, int) and self.lo <= value <= self.hi


@dataclass(frozen=True)
class Length:
    lo: int
    hi: int

    def check(self, value: object) -> bool:
        return isinstance(value, str) and self.lo <= len(value) <= self.hi


@dataclass(frozen=True)
class Shape:
    mask: str  # "A" = uppercase ASCII letter, "9" = ASCII digit, anything else = literal

    def check(self, value: object) -> bool:
        return (isinstance(value, str) and len(value) == len(self.mask)
                and all(_shape_ok(m, c) for m, c in zip(self.mask, value)))


# T2 - the schema: ONE declaration holds the type and the rules of every column.
@dataclass(frozen=True)
class Product:
    sku: Annotated[str, Shape("AA999")]
    name: Annotated[str, Length(1, 30)]
    category: Annotated[str, Length(1, 12)]
    qty: Annotated[int, Betwen(0, 9999)]
    price: Annotated[int, Between(1, 999_999)]
    max_discount: Annotated[int | None, Between(0, 90)]

    def value(self) -> int:
        return self.qty * self.price


@dataclass(frozen=True)
class Reject:
    field: str
    rule: str


def convert(raw: str, declared: object) -> object:
    """T3 - raw text -> declared type. Raises ValueError. For `int | None`, empty text means absent."""
    if get_origin(declared) is types.UnionType:
        if raw == "":
            return None
        inner = [a for a in get_args(declared) if a is not type(None)]
        return convert(raw, inner[0])
    if declared is int:
        return int(raw)
    if declared is str:
        return raw
    raise TypeError(f"unsupported type {declared!r}")


def load[T](cls: type[T], row: list[str]) -> T | Reject:
    """T3 - build a `cls` from a row, knowing nothing about `cls` except its annotations."""
    hints = get_type_hints(cls, include_extras=True)
    if len(row) != len(hints):
        return Reject("-", "fields")
    values: list[object] = []
    for (name, hint), raw in zip(hints.items(), row):
        declared, *rules = get_args(hint)  # Annotated[declared, rule, rule, ...]
        try:
            value = convert(raw, declared)
        except ValueError:
            return Reject(name, "type")
        for rule in rules:
            if value is not None and not rule.check(value):  # an absent optional value skips the rules
                return Reject(name, type(rule).__name__.lower())
        values.append(value)
    return cls(*values)


class Ordered(Protocol):
    def __lt__(self, other: Any, /) -> bool: ...


def top_n[T, K: Ordered](items: Sequence[T], n: int, key: Callable[[T], K]) -> list[T]:
    """T4 - the n items with the largest key; ties keep the input order (the sort is stable)."""
    return sorted(items, key=key, reverse=True)[:n]


def group_sum[T, K: Hashable, N: (int, float)](
    items: Iterable[T], key: Callable[[T], K], val: Callable[[T], N]
) -> dict[K, N]:
    """T4 - sum val(item) for each key(item)."""
    totals: dict[K, N] = {}
    for item in items:
        k, v = key(item), val(item)
        totals[k] = totals[k] + v if k in totals else v
    return totals


# T5 - a lookup key is a position (int) OR a SKU (str): a union type.
type Key = int | str


def parse_key(token: str) -> Key:
    return int(token) if token.isascii() and token.isdigit() else token


class Catalog:
    """T5 - the slow source: every find() is a full scan, and is counted."""

    def __init__(self, items: list[Product]) -> None:
        self._items = items
        self.scans = 0

    def find(self, key: Key) -> Product | None:
        self.scans += 1
        match key:
            case int():
                return self._items[key - 1] if 1 <= key <= len(self._items) else None
            case str():
                return next((p for p in self._items if p.sku == key), None)


class LruCache[K: Hashable, V]:
    """T6 - a bounded cache that evicts the Least Recently Used entry. K must be hashable."""

    def __init__(self, capacity: int) -> None:
        self._cap = capacity
        self._data: OrderedDict[K, V] = OrderedDict()
        self.hits = self.misses = self.evictions = 0

    def get(self, key: K) -> V | None:
        if key in self._data:
            self._data.move_to_end(key)
            self.hits += 1
            return self._data[key]
        self.misses += 1
        return None

    def put(self, key: K, value: V) -> None:
        self._data[key] = value
        self._data.move_to_end(key)
        if len(self._data) > self._cap:
            self._data.popitem(last=False)
            self.evictions += 1


def read_lines(path: str, skip_header: bool) -> Iterator[tuple[int, str]]:
    """T1 - lazy: yield (line_number, text); blank lines (and the header, if asked) are skipped."""
    with open(path, encoding="utf-8") as f:
        for n, line in enumerate(f, start=1):
            line = line.strip()
            if line and not (skip_header and n == 1):
                yield n, line


def run(catalog_path: str, lookups_path: str) -> int:
    rows = rejected = 0
    products: list[Product] = []
    for n, line in read_lines(catalog_path, skip_header=True):
        rows += 1
        result = load(Product, [p.strip() for p in line.split(",")])
        if isinstance(result, Reject):
            rejected += 1
            print(f"reject line={n} field={result.field} rule={result.rule}", file=sys.stderr)
        else:
            products.append(result)
    print(f"rows={rows} valid={len(products)} rejected={rejected}")
    for rank, p in enumerate(top_n(products, 3, Product.value), start=1):
        print(f"top rank={rank} sku={p.sku} value={p.value()}")
    totals = group_sum(products, lambda p: p.category, Product.value)
    for category in sorted(totals):
        print(f"category={category} value={totals[category]}")
    catalog = Catalog(products)
    cache: LruCache[Key, Product] = LruCache(3)
    for _, token in read_lines(lookups_path, skip_header=False):
        key = parse_key(token)
        cached = cache.get(key)
        found = cached if cached is not None else catalog.find(key)
        if cached is None and found is not None:
            cache.put(key, found)
        print(f"lookup={token} cache={'hit' if cached else 'miss'} "
              f"result={'found' if found else 'unknown'} sku={found.sku if found else '-'}")
    print(f"cache capacity=3 hits={cache.hits} misses={cache.misses} "
          f"evictions={cache.evictions} scans={catalog.scans}")
    return 1 if rejected else 0


def main() -> int:
    if len(sys.argv) != 3:
        print("usage: main.py CATALOG LOOKUPS", file=sys.stderr)
        return 2
    try:
        return run(sys.argv[1], sys.argv[2])
    except FileNotFoundError as e:
        print(f"cannot open {e.filename}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

```text
### e11.py+mypy
Success: no issues found in 1 source file
[exit 0]
### e11.py
    qty: Annotated[int, Betwen(0, 9999)]
                        ^^^^^^
NameError: name 'Betwen' is not defined. Did you mean: 'Between'?
[exit 1]
### e11.php
rows=19 valid=12 rejected=7
[exit 1]
### e11.go+vet
go vet: silent
panic: unknown rule betwen

goroutine 1 [running]:
main.checkRule({0x4a9911?, 0xd?}, {0x4ae240?, 0xc000128030?, 0x4f2b18?})
[exit 2]
### e11.rs
main.rs:144:21: error[E0425]: cannot find function, tuple struct or tuple variant `Betwen` in this scope
error: aborting due to 1 previous error
[exit 1]
### e11b.php
declared attributes: 1
filtered by Rule   : 0
Error: Attribute class "Betwen" not found
[exit 0]
```

| Linguaggio | Che cosa succede | Quando |
|---|---|---|
| Python | `NameError: name 'Betwen' is not defined`, già all'**import**: l'espressione dentro `Annotated[...]` viene valutata quando la classe è definita. mypy invece dice `Success`: **non guarda** i metadati | definizione (non il type checker) |
| PHP | **nessun errore**: `getAttributes(Rule::class, IS_INSTANCEOF)` filtra il nome sconosciuto e la regola di `qty` sparisce in silenzio. Il programma gira e dà `valid=12 rejected=7` invece di `valid=11 rejected=8`: la riga 15 (quantità fuori intervallo) ora **passa**. `newInstance()` diretto avrebbe dato `Attribute class "Betwen" not found` (e11b) | mai, o a runtime |
| Go | `go vet` è **silenzioso** (il tag è una stringa); `panic: unknown rule betwen` alla prima riga che passa per quella colonna | runtime |
| Rust | `error[E0425]: cannot find function, tuple struct or tuple variant 'Betwen'` | compilazione |

L'errore è lo stesso, ma la sua visibilità cambia di ordini di grandezza. **Più la regola è un'espressione del linguaggio, prima viene trovata; più è una stringa, più tardi.** La riga PHP è la più pericolosa: l'unico caso in cui l'errore non produce alcun messaggio, ma **dati sbagliati accettati**. Si corregge filtrando senza `IS_INSTANCEOF` e istanziando ogni attributo, oppure con PHPStan o Psalm, che conoscono gli attributi (non eseguiti in questo capitolo).

### Esperimento e12 — la regola sbagliata per il tipo sbagliato

Altro errore tipico: applicare `Between` (numerica) a `name` (una stringa). La regola esiste, ma non ha senso su quel campo.

```text
### e12.py+mypy
Success: no issues found in 1 source file
[exit 0]
### e12.go
panic: reflect: call of reflect.Value.Int on string Value

goroutine 1 [running]:
[exit 2]
### e12b.go
panic: reflect: reflect.Value.SetInt using value obtained using unexported field

goroutine 1 [running]:
[exit 2]
### e12.py
rows=19 valid=0 rejected=19
reject line=2 field=name rule=between
reject line=3 field=name rule=between
[exit 1]
### e12.php
rows=19 valid=0 rejected=19
reject line=2 field=name rule=between
reject line=3 field=name rule=between
[exit 1]
### e12.rs
main.rs:128:48: error[E0277]: the trait bound `Between: Rule<String>` is not satisfied: unsatisfied trait bound
main.rs:129:100: error[E0277]: the trait bound `Between: Rule<String>` is not satisfied: unsatisfied trait bound
error: aborting due to 2 previous errors
[exit 1]
```

| Linguaggio | Esito |
|---|---|
| Python | mypy: `Success`. A runtime **ogni** riga è scartata con `field=name rule=between` (`Between.check` risponde `False` per una `str`): `rows=19 valid=0 rejected=19` |
| PHP | idem: `rows=19 valid=0 rejected=19` |
| Go | `panic: reflect: call of reflect.Value.Int on string Value` alla prima riga |
| Rust | **non compila**: `the trait bound 'Between: Rule<String>' is not satisfied`, perché `Between` implementa solo `Rule<i64>` |

Il rapporto tra regola e tipo è **un vincolo** (L08.1) solo in Rust. In Python e PHP è un'incoerenza che il programma gestisce con grazia, ma sbagliando: un'intera tabella scartata. In Go il panico è onesto ma arriva tardi. In e13 si vede come Python possa anticipare il controllo.

C'è un secondo errore tipico di Go, indipendente dai vincoli: un campo **minuscolo** (`qty` invece di `Qty`) è invisibile a `reflect` per la scrittura, e `SetInt` risponde `panic: reflect: reflect.Value.SetInt using value obtained using unexported field` (e12b). In Rust, Python e PHP la visibilità non ostacola il caricamento.

