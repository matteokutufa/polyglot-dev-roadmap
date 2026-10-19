(parte 8 di 8: Appendice — programmi completi — file C08-parte8-appendice.md)

## Appendice — Il programma completo

### Python

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
    qty: Annotated[int, Between(0, 9999)]
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
    # T7 - the lookups go through the cache; then the final report and the exit code
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

### PHP

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

final readonly class Reject
{
    public function __construct(public string $field, public string $rule)
    {
    }
}

/** T3 - text to int, strictly: optional sign, digits, no overflow (filter_var refuses leading zeros, so strip them). */
function parseInt(string $raw): ?int
{
    if (preg_match('/^([+-]?)0*([0-9]+)\z/', $raw, $m) !== 1) {
        return null;
    }
    $n = filter_var(($m[1] === '-' ? '-' : '') . $m[2], FILTER_VALIDATE_INT);
    return $n === false ? null : $n;
}

/**
 * T3 - build a $class from a row, knowing nothing about $class except its constructor and attributes.
 *
 * @template T of object
 * @param class-string<T> $class
 * @param list<string> $row
 * @return T|Reject
 */
function load(string $class, array $row): object
{
    $params = (new ReflectionClass($class))->getConstructor()?->getParameters() ?? [];
    if (count($row) !== count($params)) {
        return new Reject('-', 'fields');
    }
    $args = [];
    foreach ($params as $i => $param) {
        $type = $param->getType();
        assert($type instanceof ReflectionNamedType);
        $raw = $row[$i];
        if ($raw === '' && $type->allowsNull()) {
            $args[] = null; // an absent optional value skips the rules
            continue;
        }
        $value = match ($type->getName()) {
            'int' => parseInt($raw),
            'string' => $raw,
            default => throw new LogicException('unsupported type ' . $type->getName()),
        };
        if ($value === null) {
            return new Reject($param->getName(), 'type');
        }
        foreach ($param->getAttributes(Rule::class, ReflectionAttribute::IS_INSTANCEOF) as $attr) {
            $rule = $attr->newInstance();
            if (!$rule->check($value)) {
                return new Reject($param->getName(), strtolower($attr->getName()));
            }
        }
        $args[] = $value;
    }
    return new $class(...$args);
}

/**
 * T4 - the n items with the largest key; ties keep the input order (usort is stable since PHP 8.0).
 *
 * @template T
 * @template K of int|float|string
 * @param list<T> $items
 * @param callable(T): K $key
 * @return list<T>
 */
function topN(array $items, int $n, callable $key): array
{
    usort($items, fn($a, $b) => $key($b) <=> $key($a));
    return array_slice($items, 0, $n);
}

/**
 * T4 - sum val(item) for each key(item).
 *
 * @template T
 * @template K of int|string
 * @template N of int|float
 * @param iterable<T> $items
 * @param callable(T): K $key
 * @param callable(T): N $val
 * @return array<K,N>
 */
function groupSum(iterable $items, callable $key, callable $val): array
{
    $totals = [];
    foreach ($items as $item) {
        $k = $key($item);
        $totals[$k] = ($totals[$k] ?? 0) + $val($item);
    }
    return $totals;
}

// T5 - a lookup key is a position (int) OR a SKU (string): a union type, `int|string`.
function parseKey(string $token): int|string
{
    return ctype_digit($token) ? (int) $token : $token;
}

final class Catalog
{
    public int $scans = 0;

    /** @param list<Product> $items */
    public function __construct(private readonly array $items)
    {
    }

    /** T5 - the slow source: every find() is a full scan, and is counted. */
    public function find(int|string $key): ?Product
    {
        $this->scans++;
        if (is_int($key)) {
            return $this->items[$key - 1] ?? null;
        }
        foreach ($this->items as $p) {
            if ($p->sku === $key) {
                return $p;
            }
        }
        return null;
    }
}

/**
 * T6 - a bounded cache that evicts the Least Recently Used entry. K must be usable as an array key.
 *
 * @template K of int|string
 * @template V
 */
final class LruCache
{
    /** @var array<K,V> oldest first */
    private array $data = [];
    public int $hits = 0;
    public int $misses = 0;
    public int $evictions = 0;

    public function __construct(private readonly int $capacity)
    {
    }

    /**
     * @param K $key
     * @return V|null
     */
    public function get(int|string $key): mixed
    {
        if (array_key_exists($key, $this->data)) {
            $value = $this->data[$key];
            unset($this->data[$key]);
            $this->data[$key] = $value; // re-insert: now the newest
            $this->hits++;
            return $value;
        }
        $this->misses++;
        return null;
    }

    /**
     * @param K $key
     * @param V $value
     */
    public function put(int|string $key, mixed $value): void
    {
        unset($this->data[$key]);
        $this->data[$key] = $value;
        if (count($this->data) > $this->capacity) {
            unset($this->data[array_key_first($this->data)]);
            $this->evictions++;
        }
    }
}

/** T1 - lazy: yield line_number => text; blank lines (and the header, if asked) are skipped. */
function readLines(string $path, bool $skipHeader): Generator
{
    $f = @fopen($path, 'rb');
    if ($f === false) {
        throw new RuntimeException("cannot open $path");
    }
    try {
        $n = 0;
        while (($line = fgets($f)) !== false) {
            $n++;
            $line = trim($line);
            if ($line !== '' && !($skipHeader && $n === 1)) {
                yield $n => $line;
            }
        }
    } finally {
        fclose($f);
    }
}

function run(string $catalogPath, string $lookupsPath): int
{
    $rows = $rejected = 0;
    /** @var list<Product> $products */
    $products = [];
    foreach (readLines($catalogPath, true) as $n => $line) {
        $rows++;
        $result = load(Product::class, array_map('trim', explode(',', $line)));
        if ($result instanceof Reject) {
            $rejected++;
            fwrite(STDERR, "reject line=$n field=$result->field rule=$result->rule\n");
        } else {
            $products[] = $result;
        }
    }
    echo "rows=$rows valid=" . count($products) . " rejected=$rejected\n";
    foreach (topN($products, 3, fn(Product $p) => $p->value()) as $i => $p) {
        echo 'top rank=' . ($i + 1) . " sku=$p->sku value={$p->value()}\n";
    }
    $totals = groupSum($products, fn(Product $p) => $p->category, fn(Product $p) => $p->value());
    ksort($totals);
    foreach ($totals as $category => $value) {
        echo "category=$category value=$value\n";
    }
    // T7 - the lookups go through the cache; then the final report and the exit code
    $catalog = new Catalog($products);
    /** @var LruCache<int|string, Product> $cache */
    $cache = new LruCache(3);
    foreach (readLines($lookupsPath, false) as $token) {
        $key = parseKey($token);
        $cached = $cache->get($key);
        $found = $cached ?? $catalog->find($key);
        if ($cached === null && $found !== null) {
            $cache->put($key, $found);
        }
        echo "lookup=$token cache=" . ($cached !== null ? 'hit' : 'miss')
            . ' result=' . ($found !== null ? 'found' : 'unknown')
            . ' sku=' . ($found?->sku ?? '-') . "\n";
    }
    echo "cache capacity=3 hits=$cache->hits misses=$cache->misses evictions=$cache->evictions scans=$catalog->scans\n";
    return $rejected > 0 ? 1 : 0;
}

if ($argc !== 3) {
    fwrite(STDERR, "usage: main.php CATALOG LOOKUPS\n");
    exit(2);
}
try {
    exit(run($argv[1], $argv[2]));
} catch (RuntimeException $e) {
    fwrite(STDERR, $e->getMessage() . "\n");
    exit(2);
}
```

### Rust

```rust
use std::collections::{BTreeMap, HashMap, VecDeque};
use std::fs::File;
use std::hash::Hash;
use std::io::{BufRead, BufReader};
use std::ops::Add;
use std::process::ExitCode;

/// T2 - why a row was refused: the column and the rule.
#[derive(Debug, Clone, PartialEq, Eq)]
struct Reject {
    field: &'static str,
    rule: &'static str,
}

/// T2 - a rule that can judge values of type T. `Between` judges i64, `Length` judges String: the
/// pairing is checked by the compiler.
trait Rule<T> {
    fn check(&self, v: &T) -> bool;
    fn name(&self) -> &'static str;
}

struct Between(i64, i64);
struct Length(usize, usize);
struct Shape(&'static str); // "A" = uppercase ASCII letter, "9" = ASCII digit, anything else = literal

impl Rule<i64> for Between {
    fn check(&self, v: &i64) -> bool {
        self.0 <= *v && *v <= self.1
    }
    fn name(&self) -> &'static str {
        "between"
    }
}

impl Rule<String> for Length {
    fn check(&self, v: &String) -> bool {
        (self.0..=self.1).contains(&v.chars().count())
    }
    fn name(&self) -> &'static str {
        "length"
    }
}

impl Rule<String> for Shape {
    fn check(&self, v: &String) -> bool {
        v.len() == self.0.len()
            && self.0.bytes().zip(v.bytes()).all(|(m, c)| match m {
                b'A' => c.is_ascii_uppercase(),
                b'9' => c.is_ascii_digit(),
                _ => m == c,
            })
    }
    fn name(&self) -> &'static str {
        "shape"
    }
}

/// A rule for T also judges Option<T>: an absent value passes (blanket implementation).
impl<T, R: Rule<T>> Rule<Option<T>> for R {
    fn check(&self, v: &Option<T>) -> bool {
        v.as_ref().is_none_or(|x| Rule::<T>::check(self, x))
    }
    fn name(&self) -> &'static str {
        Rule::<T>::name(self)
    }
}

/// T3 - raw text to a declared type. `None` means the text is not valid for that type.
trait FieldType: Sized {
    fn parse(raw: &str) -> Option<Self>;
}

impl FieldType for String {
    fn parse(raw: &str) -> Option<Self> {
        Some(raw.to_string())
    }
}

impl FieldType for i64 {
    fn parse(raw: &str) -> Option<Self> {
        raw.parse().ok()
    }
}

/// For Option<T>, empty text means absent.
impl<T: FieldType> FieldType for Option<T> {
    fn parse(raw: &str) -> Option<Self> {
        if raw.is_empty() { Some(None) } else { T::parse(raw).map(Some) }
    }
}

/// T3 - what the generic loader needs from a type: it can be built from a row.
trait Schema: Sized {
    const COLUMNS: usize;
    fn from_row(row: &[&str]) -> Result<Self, Reject>;
}

/// T3 - generic over any type that has a schema.
fn load<T: Schema>(row: &[&str]) -> Result<T, Reject> {
    T::from_row(row)
}

/// T2 - the macro writes the struct AND its `from_row` from one declaration: type and rule per column.
macro_rules! schema {
    (struct $name:ident { $( $field:ident : $ty:ty => $rule:expr ),+ $(,)? }) => {
        #[derive(Debug, Clone, PartialEq, Eq)]
        struct $name { $( $field: $ty ),+ }

        impl Schema for $name {
            const COLUMNS: usize = [$( stringify!($field) ),+].len();

            fn from_row(row: &[&str]) -> Result<Self, Reject> {
                if row.len() != Self::COLUMNS {
                    return Err(Reject { field: "-", rule: "fields" });
                }
                let mut cells = row.iter();
                Ok($name { $(
                    $field: {
                        let raw = cells.next().expect("length checked above");
                        let v = <$ty as FieldType>::parse(raw)
                            .ok_or(Reject { field: stringify!($field), rule: "type" })?;
                        let rule = $rule;
                        if !Rule::<$ty>::check(&rule, &v) {
                            return Err(Reject { field: stringify!($field), rule: Rule::<$ty>::name(&rule) });
                        }
                        v
                    }
                ),+ })
            }
        }
    };
}

schema! {
    struct Product {
        sku: String => Shape("AA999"),
        name: String => Length(1, 30),
        category: String => Length(1, 12),
        qty: i64 => Between(0, 9999),
        price: i64 => Between(1, 999_999),
        max_discount: Option<i64> => Between(0, 90),
    }
}

impl Product {
    fn value(&self) -> i64 {
        self.qty * self.price
    }
}

/// T4 - the n items with the largest key; ties keep the input order (the sort is stable).
fn top_n<T, K: Ord>(items: &[T], n: usize, key: impl Fn(&T) -> K) -> Vec<&T> {
    let mut refs: Vec<&T> = items.iter().collect();
    refs.sort_by(|a, b| key(b).cmp(&key(a)));
    refs.truncate(n);
    refs
}

/// T4 - sum val(item) for each key(item); BTreeMap keeps the keys sorted.
fn group_sum<T, K: Ord, N: Add<Output = N> + Copy>(
    items: &[T],
    key: impl Fn(&T) -> K,
    val: impl Fn(&T) -> N,
) -> BTreeMap<K, N> {
    let mut totals = BTreeMap::new();
    for item in items {
        let v = val(item);
        totals.entry(key(item)).and_modify(|t| *t = *t + v).or_insert(v);
    }
    totals
}

/// T5 - a lookup key is a position OR a SKU: a sum type (enum).
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
enum Key {
    Pos(usize),
    Sku(String),
}

fn parse_key(token: &str) -> Key {
    match token.parse::<usize>() {
        Ok(n) if token.bytes().all(|b| b.is_ascii_digit()) => Key::Pos(n),
        _ => Key::Sku(token.to_string()),
    }
}

/// T5 - the slow source: every find() is a full scan, and is counted.
struct Catalog {
    items: Vec<Product>,
    scans: u32,
}

impl Catalog {
    fn find(&mut self, key: &Key) -> Option<&Product> {
        self.scans += 1;
        match key {
            Key::Pos(n) => n.checked_sub(1).and_then(|i| self.items.get(i)),
            Key::Sku(s) => self.items.iter().find(|p| &p.sku == s),
        }
    }
}

/// T6 - a bounded cache that evicts the Least Recently Used entry. K needs Eq + Hash (map key) and Clone.
struct LruCache<K: Eq + Hash + Clone, V> {
    capacity: usize,
    data: HashMap<K, V>,
    order: VecDeque<K>, // oldest first
    hits: u32,
    misses: u32,
    evictions: u32,
}

impl<K: Eq + Hash + Clone, V> LruCache<K, V> {
    fn new(capacity: usize) -> Self {
        LruCache { capacity, data: HashMap::new(), order: VecDeque::new(), hits: 0, misses: 0, evictions: 0 }
    }

    fn touch(&mut self, key: &K) {
        if let Some(i) = self.order.iter().position(|k| k == key) {
            let k = self.order.remove(i).expect("position is valid");
            self.order.push_back(k);
        }
    }

    fn get(&mut self, key: &K) -> Option<&V> {
        if self.data.contains_key(key) {
            self.touch(key);
            self.hits += 1;
            self.data.get(key)
        } else {
            self.misses += 1;
            None
        }
    }

    fn put(&mut self, key: K, value: V) {
        if self.data.insert(key.clone(), value).is_some() {
            self.touch(&key);
            return;
        }
        self.order.push_back(key);
        if self.order.len() > self.capacity {
            let oldest = self.order.pop_front().expect("not empty");
            self.data.remove(&oldest);
            self.evictions += 1;
        }
    }
}

/// T1 - lazy: an iterator of (line_number, text); blank lines (and the header, if asked) are skipped.
fn read_lines(path: &str, skip_header: bool) -> Result<impl Iterator<Item = (usize, String)>, String> {
    let f = File::open(path).map_err(|_| path.to_string())?;
    Ok(BufReader::new(f).lines().enumerate().filter_map(move |(i, l)| {
        let t = l.ok()?.trim().to_string();
        if t.is_empty() || (skip_header && i == 0) { None } else { Some((i + 1, t)) }
    }))
}

fn run(catalog_path: &str, lookups_path: &str) -> Result<u8, String> {
    let (mut rows, mut rejected) = (0, 0);
    let mut products: Vec<Product> = Vec::new();
    for (n, line) in read_lines(catalog_path, true)? {
        rows += 1;
        let cells: Vec<&str> = line.split(',').map(str::trim).collect();
        match load::<Product>(&cells) {
            Ok(p) => products.push(p),
            Err(r) => {
                rejected += 1;
                eprintln!("reject line={n} field={} rule={}", r.field, r.rule);
            }
        }
    }
    println!("rows={rows} valid={} rejected={rejected}", products.len());
    for (i, p) in top_n(&products, 3, Product::value).iter().enumerate() {
        println!("top rank={} sku={} value={}", i + 1, p.sku, p.value());
    }
    for (category, value) in group_sum(&products, |p| p.category.clone(), Product::value) {
        println!("category={category} value={value}");
    }
    // T7 - the lookups go through the cache; then the final report and the exit code
    let mut catalog = Catalog { items: products, scans: 0 };
    let mut cache: LruCache<Key, Product> = LruCache::new(3);
    for (_, token) in read_lines(lookups_path, false)? {
        let key = parse_key(&token);
        let cached = cache.get(&key).cloned();
        let found = match &cached {
            Some(p) => Some(p.clone()),
            None => catalog.find(&key).cloned(),
        };
        if cached.is_none() {
            if let Some(p) = &found {
                cache.put(key, p.clone());
            }
        }
        println!(
            "lookup={token} cache={} result={} sku={}",
            if cached.is_some() { "hit" } else { "miss" },
            if found.is_some() { "found" } else { "unknown" },
            found.as_ref().map_or("-", |p| p.sku.as_str())
        );
    }
    println!(
        "cache capacity=3 hits={} misses={} evictions={} scans={}",
        cache.hits, cache.misses, cache.evictions, catalog.scans
    );
    Ok(if rejected > 0 { 1 } else { 0 })
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 3 {
        eprintln!("usage: main CATALOG LOOKUPS");
        return ExitCode::from(2);
    }
    match run(&args[1], &args[2]) {
        Ok(code) => ExitCode::from(code),
        Err(path) => {
            eprintln!("cannot open {path}");
            ExitCode::from(2)
        }
    }
}
```

### Go

```go
package main

import (
	"bufio"
	"cmp"
	"fmt"
	"iter"
	"os"
	"reflect"
	"slices"
	"strconv"
	"strings"
	"unicode/utf8"
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

// T2 - the rules: a name and arguments in a tag string, interpreted at run time.
func shapeOK(mask, s string) bool {
	if len(s) != len(mask) {
		return false
	}
	for i := 0; i < len(mask); i++ {
		switch m, c := mask[i], s[i]; m {
		case 'A':
			if c < 'A' || c > 'Z' {
				return false
			}
		case '9':
			if c < '0' || c > '9' {
				return false
			}
		default:
			if m != c {
				return false
			}
		}
	}
	return true
}

func twoInts(args string) (lo, hi int64) {
	a, b, _ := strings.Cut(args, ":")
	lo, _ = strconv.ParseInt(a, 10, 64)
	hi, _ = strconv.ParseInt(b, 10, 64)
	return
}

// checkRule applies one tag rule to a value and returns whether it holds, plus the rule name.
func checkRule(spec string, v reflect.Value) (bool, string) {
	name, args, _ := strings.Cut(spec, "=")
	switch name {
	case "between":
		lo, hi := twoInts(args)
		n := v.Int()
		return lo <= n && n <= hi, name
	case "length":
		lo, hi := twoInts(args)
		n := int64(utf8.RuneCountInString(v.String()))
		return lo <= n && n <= hi, name
	case "shape":
		return shapeOK(args, v.String()), name
	}
	panic("unknown rule " + name)
}

type Reject struct{ Field, Rule string }

// setFrom converts raw text to the kind of dst (string or integer). It reports whether the text was valid.
func setFrom(dst reflect.Value, raw string) bool {
	switch dst.Kind() {
	case reflect.String:
		dst.SetString(raw)
	case reflect.Int:
		n, err := strconv.ParseInt(raw, 10, 64)
		if err != nil {
			return false
		}
		dst.SetInt(n)
	default:
		panic("unsupported kind " + dst.Kind().String())
	}
	return true
}

// Load is T3 - build a T from a row, knowing nothing about T except its fields and tags.
func Load[T any](row []string) (T, *Reject) {
	var out T
	v := reflect.ValueOf(&out).Elem()
	t := v.Type()
	if len(row) != t.NumField() {
		return out, &Reject{"-", "fields"}
	}
	for i := 0; i < t.NumField(); i++ {
		f, fv, raw := t.Field(i), v.Field(i), row[i]
		col := f.Tag.Get("col")
		target := fv
		if f.Type.Kind() == reflect.Pointer { // optional column: empty text leaves the pointer nil
			if raw == "" {
				continue
			}
			target = reflect.New(f.Type.Elem()).Elem()
		}
		if !setFrom(target, raw) {
			return out, &Reject{col, "type"}
		}
		for _, spec := range strings.Fields(f.Tag.Get("rule")) {
			if ok, name := checkRule(spec, target); !ok {
				return out, &Reject{col, name}
			}
		}
		if f.Type.Kind() == reflect.Pointer {
			fv.Set(target.Addr())
		}
	}
	return out, nil
}

// TopN is T4 - the n items with the largest key; ties keep the input order (the sort is stable).
func TopN[T any, K cmp.Ordered](items []T, n int, key func(T) K) []T {
	out := slices.Clone(items)
	slices.SortStableFunc(out, func(a, b T) int { return cmp.Compare(key(b), key(a)) })
	return out[:min(n, len(out))]
}

// Number is the constraint of the values we can add up: a TYPE SET, not a type.
type Number interface {
	~int | ~int64 | ~float64
}

// GroupSum is T4 - sum val(item) for each key(item).
func GroupSum[T any, K comparable, N Number](items []T, key func(T) K, val func(T) N) map[K]N {
	totals := map[K]N{}
	for _, item := range items {
		totals[key(item)] += val(item)
	}
	return totals
}

// Key is T5 - a lookup key is a position OR a SKU. Go has no union types: a sealed interface plays the role.
type Key interface{ isKey() }

type Pos int
type Sku string

func (Pos) isKey() {}
func (Sku) isKey() {}

func parseKey(token string) Key {
	if n, err := strconv.Atoi(token); err == nil && !strings.HasPrefix(token, "+") && !strings.HasPrefix(token, "-") {
		return Pos(n)
	}
	return Sku(token)
}

// Catalog is T5 - the slow source: every Find is a full scan, and is counted.
type Catalog struct {
	items []Product
	scans int
}

func (c *Catalog) Find(key Key) (Product, bool) {
	c.scans++
	switch k := key.(type) {
	case Pos:
		if i := int(k); i >= 1 && i <= len(c.items) {
			return c.items[i-1], true
		}
	case Sku:
		for _, p := range c.items {
			if p.SKU == string(k) {
				return p, true
			}
		}
	}
	return Product{}, false
}

// LruCache is T6 - a bounded cache that evicts the Least Recently Used entry. K must be comparable (a map key).
type LruCache[K comparable, V any] struct {
	capacity  int
	data      map[K]V
	order     []K // oldest first
	Hits      int
	Misses    int
	Evictions int
}

func NewLruCache[K comparable, V any](capacity int) *LruCache[K, V] {
	return &LruCache[K, V]{capacity: capacity, data: map[K]V{}}
}

func (c *LruCache[K, V]) touch(key K) {
	if i := slices.Index(c.order, key); i >= 0 {
		c.order = append(slices.Delete(c.order, i, i+1), key)
	}
}

func (c *LruCache[K, V]) Get(key K) (V, bool) {
	v, ok := c.data[key]
	if ok {
		c.touch(key)
		c.Hits++
	} else {
		c.Misses++
	}
	return v, ok
}

func (c *LruCache[K, V]) Put(key K, value V) {
	if _, exists := c.data[key]; exists {
		c.data[key] = value
		c.touch(key)
		return
	}
	c.data[key] = value
	c.order = append(c.order, key)
	if len(c.order) > c.capacity {
		delete(c.data, c.order[0])
		c.order = c.order[1:]
		c.Evictions++
	}
}

// readLines is T1 - lazy: an iter.Seq2 of (line_number, text); blank lines (and the header, if asked) are skipped.
func readLines(path string, skipHeader bool) (iter.Seq2[int, string], error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	return func(yield func(int, string) bool) {
		defer f.Close()
		sc := bufio.NewScanner(f)
		for n := 1; sc.Scan(); n++ {
			t := strings.TrimSpace(sc.Text())
			if t == "" || (skipHeader && n == 1) {
				continue
			}
			if !yield(n, t) {
				return
			}
		}
	}, nil
}

func run(catalogPath, lookupsPath string) int {
	lines, err := readLines(catalogPath, true)
	if err != nil {
		fmt.Fprintf(os.Stderr, "cannot open %s\n", catalogPath)
		return 2
	}
	lookups, err := readLines(lookupsPath, false)
	if err != nil {
		fmt.Fprintf(os.Stderr, "cannot open %s\n", lookupsPath)
		return 2
	}
	rows, rejected := 0, 0
	var products []Product
	for n, line := range lines {
		rows++
		cells := strings.Split(line, ",")
		for i := range cells {
			cells[i] = strings.TrimSpace(cells[i])
		}
		p, rej := Load[Product](cells)
		if rej != nil {
			rejected++
			fmt.Fprintf(os.Stderr, "reject line=%d field=%s rule=%s\n", n, rej.Field, rej.Rule)
			continue
		}
		products = append(products, p)
	}
	fmt.Printf("rows=%d valid=%d rejected=%d\n", rows, len(products), rejected)
	for i, p := range TopN(products, 3, Product.Value) {
		fmt.Printf("top rank=%d sku=%s value=%d\n", i+1, p.SKU, p.Value())
	}
	totals := GroupSum(products, func(p Product) string { return p.Category }, Product.Value)
	for _, category := range slices.Sorted(func(yield func(string) bool) {
		for k := range totals {
			if !yield(k) {
				return
			}
		}
	}) {
		fmt.Printf("category=%s value=%d\n", category, totals[category])
	}
	// T7 - the lookups go through the cache; then the final report and the exit code
	catalog := &Catalog{items: products}
	cache := NewLruCache[Key, Product](3)
	for _, token := range lookups {
		key := parseKey(token)
		cached, hit := cache.Get(key)
		found, ok := cached, hit
		if !hit {
			found, ok = catalog.Find(key)
			if ok {
				cache.Put(key, found)
			}
		}
		sku, result, status := "-", "unknown", "miss"
		if ok {
			sku, result = found.SKU, "found"
		}
		if hit {
			status = "hit"
		}
		fmt.Printf("lookup=%s cache=%s result=%s sku=%s\n", token, status, result, sku)
	}
	fmt.Printf("cache capacity=3 hits=%d misses=%d evictions=%d scans=%d\n", cache.Hits, cache.Misses, cache.Evictions, catalog.scans)
	if rejected > 0 {
		return 1
	}
	return 0
}

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage: main CATALOG LOOKUPS")
		os.Exit(2)
	}
	os.Exit(run(os.Args[1], os.Args[2]))
}
```
