# C06 — Tipi composti e modellazione dei dati (parte 5 di 5: appendice con i quattro programmi completi)

*Capitolo C06, file `C06-parte5-appendice.md`: leggere in ordine parte1 → parte5.*

## Appendice — I quattro programmi completi

Eseguibili con: `python3 main.py events.csv`, `php main.php events.csv`, `rustc --edition 2024 main.rs -o main_rs && ./main_rs events.csv`, `go build -o main_go . && ./main_go events.csv` (con `go.mod` che dichiara `go 1.24`). Tutti usano soltanto la libreria standard.

#### Python (`main.py`) — `python3 main.py events.csv`

```python
import sys
from dataclasses import dataclass


@dataclass(frozen=True)
class Money:
    """T3 - value object: an amount in euro cents. Immutable, compared by value."""
    cents: int

    def __add__(self, other: "Money") -> "Money":
        return Money(self.cents + other.cents)

    def __str__(self) -> str:
        return f"{self.cents // 100}.{self.cents % 100:02d}"


@dataclass(frozen=True)
class Created:
    amount: Money


@dataclass(frozen=True)
class Paid:
    amount: Money


@dataclass(frozen=True)
class Shipped:
    amount: Money
    tracking: str


@dataclass(frozen=True)
class Delivered:
    amount: Money


@dataclass(frozen=True)
class Cancelled:
    reason: str


State = Created | Paid | Shipped | Delivered | Cancelled  # sum type: exactly one of these


@dataclass(frozen=True)
class Create:
    amount: Money


@dataclass(frozen=True)
class Pay:
    pass


@dataclass(frozen=True)
class Ship:
    tracking: str


@dataclass(frozen=True)
class Deliver:
    pass


@dataclass(frozen=True)
class Cancel:
    reason: str


Event = Create | Pay | Ship | Deliver | Cancel


def read_rows(path):
    """T1 - lazy: yield (line_number, text), header skipped."""
    with open(path, encoding="utf-8") as f:
        next(f, None)
        for n, line in enumerate(f, start=2):
            line = line.strip()
            if line:
                yield n, line


def parse(line):
    """T2 - return ((order_id, event), "") or (None, reason)."""
    parts = [p.strip() for p in line.split(",")]
    if len(parts) != 3:
        return None, "fields"
    oid, name, arg = parts
    if not oid:
        return None, "fields"
    if name == "create":
        if not (arg.isascii() and arg.isdigit()) or int(arg) == 0:
            return None, "arg"
        return (oid, Create(Money(int(arg)))), ""
    if name == "pay":
        return (oid, Pay()), ""
    if name == "ship":
        return ((oid, Ship(arg)), "") if arg else (None, "arg")
    if name == "deliver":
        return (oid, Deliver()), ""
    if name == "cancel":
        return ((oid, Cancel(arg)), "") if arg else (None, "arg")
    return None, "event"


def apply(state, ev):
    """T4 - (state or None, event) -> (new state, "") or (None, reason)."""
    match state, ev:
        case None, Create(amount):
            return Created(amount), ""
        case None, _:
            return None, "unknown_order"
        case _, Create():
            return None, "duplicate"
        case Created(a), Pay():
            return Paid(a), ""
        case Paid(a), Ship(t):
            return Shipped(a, t), ""
        case Shipped(a, _), Deliver():
            return Delivered(a), ""
        case Created() | Paid(), Cancel(r):
            return Cancelled(r), ""
        case _:
            return None, "transition"


def describe(state):
    """T6 - (name, detail) for the report; one branch per variant."""
    match state:
        case Created(a):
            return "created", f"amount={a}"
        case Paid(a):
            return "paid", f"amount={a}"
        case Shipped(a, t):
            return "shipped", f"amount={a} tracking={t}"
        case Delivered(a):
            return "delivered", f"amount={a}"
        case Cancelled(r):
            return "cancelled", f"reason={r}"


NAMES = ("created", "paid", "shipped", "delivered", "cancelled")


def run(path):
    orders = {}  # T5 - order id -> current State
    events = applied = rejected = 0
    for n, line in read_rows(path):
        events += 1
        parsed, reason = parse(line)
        if parsed is None:
            rejected += 1
            print(f"reject line={n} reason={reason}", file=sys.stderr)
            continue
        oid, ev = parsed
        new, reason = apply(orders.get(oid), ev)
        if new is None:
            rejected += 1
            print(f"reject line={n} reason={reason}", file=sys.stderr)
            continue
        orders[oid] = new
        applied += 1
    # T7 - totals per variant and the final report
    counts = dict.fromkeys(NAMES, 0)
    delivered_total, open_total = Money(0), Money(0)
    lines = []
    for oid in sorted(orders):
        st = orders[oid]
        name, detail = describe(st)
        counts[name] += 1
        match st:
            case Delivered(a):
                delivered_total += a
            case Created(a) | Paid(a) | Shipped(a, _):
                open_total += a
        lines.append(f"order={oid} state={name} {detail}")
    print(f"events={events} applied={applied} rejected={rejected}")
    print("orders=" + str(len(orders)) + " " + " ".join(f"{k}={v}" for k, v in counts.items()))
    print(f"delivered_total={delivered_total} open_total={open_total}")
    print("\n".join(lines))
    return 1 if rejected else 0


def main():
    if len(sys.argv) != 2:
        print("usage: main.py FILE", file=sys.stderr)
        return 2
    try:
        return run(sys.argv[1])
    except FileNotFoundError:
        print(f"cannot open {sys.argv[1]}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

#### PHP (`main.php`) — `php main.php events.csv`

```php
<?php
declare(strict_types=1);

/** T3 - value object: an amount in euro cents. Immutable, compared by value. */
final readonly class Money
{
    public function __construct(public int $cents)
    {
    }

    public function add(Money $other): Money
    {
        return new Money($this->cents + $other->cents);
    }

    public function __toString(): string
    {
        return intdiv($this->cents, 100) . '.' . sprintf('%02d', $this->cents % 100);
    }
}

interface State
{
}

final readonly class Created implements State
{
    public function __construct(public Money $amount)
    {
    }
}

final readonly class Paid implements State
{
    public function __construct(public Money $amount)
    {
    }
}

final readonly class Shipped implements State
{
    public function __construct(public Money $amount, public string $tracking)
    {
    }
}

final readonly class Delivered implements State
{
    public function __construct(public Money $amount)
    {
    }
}

final readonly class Cancelled implements State
{
    public function __construct(public string $reason)
    {
    }
}

interface Event
{
}

final readonly class Create implements Event
{
    public function __construct(public Money $amount)
    {
    }
}

final readonly class Pay implements Event
{
}

final readonly class Ship implements Event
{
    public function __construct(public string $tracking)
    {
    }
}

final readonly class Deliver implements Event
{
}

final readonly class Cancel implements Event
{
    public function __construct(public string $reason)
    {
    }
}

/** T1 - lazy: yield line_number => text, header skipped. */
function readRows(string $path): Generator
{
    $f = @fopen($path, 'rb');
    if ($f === false) {
        throw new RuntimeException("cannot open $path");
    }
    try {
        fgets($f); // header
        $n = 1;
        while (($line = fgets($f)) !== false) {
            $n++;
            $line = trim($line);
            if ($line !== '') {
                yield $n => $line;
            }
        }
    } finally {
        fclose($f);
    }
}

/** T2 - return [[order_id, event], ''] or [null, reason]. */
function parse(string $line): array
{
    $parts = array_map('trim', explode(',', $line));
    if (count($parts) !== 3 || $parts[0] === '') {
        return [null, 'fields'];
    }
    [$oid, $name, $arg] = $parts;
    switch ($name) {
        case 'create':
            if (!ctype_digit($arg) || (int)$arg === 0) {
                return [null, 'arg'];
            }
            return [[$oid, new Create(new Money((int)$arg))], ''];
        case 'pay':
            return [[$oid, new Pay()], ''];
        case 'ship':
            return $arg !== '' ? [[$oid, new Ship($arg)], ''] : [null, 'arg'];
        case 'deliver':
            return [[$oid, new Deliver()], ''];
        case 'cancel':
            return $arg !== '' ? [[$oid, new Cancel($arg)], ''] : [null, 'arg'];
    }
    return [null, 'event'];
}

/** T4 - (state or null, event) -> [new state, ''] or [null, reason]. */
function apply(?State $s, Event $ev): array
{
    if ($s === null) {
        return $ev instanceof Create ? [new Created($ev->amount), ''] : [null, 'unknown_order'];
    }
    if ($ev instanceof Create) {
        return [null, 'duplicate'];
    }
    return match (true) {
        $s instanceof Created && $ev instanceof Pay => [new Paid($s->amount), ''],
        $s instanceof Paid && $ev instanceof Ship => [new Shipped($s->amount, $ev->tracking), ''],
        $s instanceof Shipped && $ev instanceof Deliver => [new Delivered($s->amount), ''],
        ($s instanceof Created || $s instanceof Paid) && $ev instanceof Cancel => [new Cancelled($ev->reason), ''],
        default => [null, 'transition'],
    };
}

/** T6 - [name, detail] for the report; one branch per variant. */
function describe(State $s): array
{
    return match (true) {
        $s instanceof Created => ['created', "amount=$s->amount"],
        $s instanceof Paid => ['paid', "amount=$s->amount"],
        $s instanceof Shipped => ['shipped', "amount=$s->amount tracking=$s->tracking"],
        $s instanceof Delivered => ['delivered', "amount=$s->amount"],
        $s instanceof Cancelled => ['cancelled', "reason=$s->reason"],
    };
}

function run(string $path): int
{
    $orders = []; // T5 - order id => current State
    $events = $applied = $rejected = 0;
    foreach (readRows($path) as $n => $line) {
        $events++;
        [$parsed, $reason] = parse($line);
        if ($parsed === null) {
            $rejected++;
            fwrite(STDERR, "reject line=$n reason=$reason\n");
            continue;
        }
        [$oid, $ev] = $parsed;
        [$new, $reason] = apply($orders[$oid] ?? null, $ev);
        if ($new === null) {
            $rejected++;
            fwrite(STDERR, "reject line=$n reason=$reason\n");
            continue;
        }
        $orders[$oid] = $new;
        $applied++;
    }
    // T7 - totals per variant and the final report
    $counts = ['created' => 0, 'paid' => 0, 'shipped' => 0, 'delivered' => 0, 'cancelled' => 0];
    $deliveredTotal = new Money(0);
    $openTotal = new Money(0);
    $lines = [];
    $ids = array_map('strval', array_keys($orders)); // numeric keys become int in PHP
    sort($ids, SORT_STRING);
    foreach ($ids as $oid) {
        $st = $orders[$oid];
        [$name, $detail] = describe($st);
        $counts[$name]++;
        if ($st instanceof Delivered) {
            $deliveredTotal = $deliveredTotal->add($st->amount);
        } elseif ($st instanceof Created || $st instanceof Paid || $st instanceof Shipped) {
            $openTotal = $openTotal->add($st->amount);
        }
        $lines[] = "order=$oid state=$name $detail";
    }
    echo "events=$events applied=$applied rejected=$rejected\n";
    $parts = [];
    foreach ($counts as $k => $v) {
        $parts[] = "$k=$v";
    }
    echo 'orders=' . count($orders) . ' ' . implode(' ', $parts) . "\n";
    echo "delivered_total=$deliveredTotal open_total=$openTotal\n";
    echo implode("\n", $lines) . "\n";
    return $rejected > 0 ? 1 : 0;
}

if ($argc !== 2) {
    fwrite(STDERR, "usage: main.php FILE\n");
    exit(2);
}
try {
    exit(run($argv[1]));
} catch (RuntimeException $e) {
    fwrite(STDERR, $e->getMessage() . "\n");
    exit(2);
}
```

#### Rust (`main.rs`) — `rustc --edition 2024 main.rs -o main_rs && ./main_rs events.csv`

```rust
use std::collections::HashMap;
use std::fmt;
use std::fs::File;
use std::io::{BufRead, BufReader};
use std::process::ExitCode;

/// T3 - value object: an amount in euro cents. Immutable, compared by value.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Money {
    cents: i64,
}

impl Money {
    fn add(self, other: Money) -> Money {
        Money { cents: self.cents + other.cents }
    }
}

impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}.{:02}", self.cents / 100, self.cents % 100)
    }
}

/// Sum type: exactly one variant, each with its own data.
enum State {
    Created { amount: Money },
    Paid { amount: Money },
    Shipped { amount: Money, tracking: String },
    Delivered { amount: Money },
    Cancelled { reason: String },
}

enum Event {
    Create(Money),
    Pay,
    Ship(String),
    Deliver,
    Cancel(String),
}

/// T1 - lazy: an iterator of (line_number, text), header skipped.
fn read_rows(path: &str) -> std::io::Result<impl Iterator<Item = (usize, String)>> {
    let f = File::open(path)?;
    Ok(BufReader::new(f)
        .lines()
        .enumerate()
        .skip(1)
        .filter_map(|(i, l)| {
            let t = l.ok()?.trim().to_string();
            if t.is_empty() { None } else { Some((i + 1, t)) }
        }))
}

/// T2 - Ok((order_id, event)) or Err(reason).
fn parse(line: &str) -> Result<(String, Event), &'static str> {
    let p: Vec<&str> = line.split(',').map(str::trim).collect();
    if p.len() != 3 || p[0].is_empty() {
        return Err("fields");
    }
    let (oid, arg) = (p[0].to_string(), p[2]);
    let ev = match p[1] {
        "create" => {
            if arg.is_empty() || !arg.bytes().all(|b| b.is_ascii_digit()) {
                return Err("arg");
            }
            match arg.parse::<i64>() {
                Ok(c) if c > 0 => Event::Create(Money { cents: c }),
                _ => return Err("arg"),
            }
        }
        "pay" => Event::Pay,
        "ship" if !arg.is_empty() => Event::Ship(arg.to_string()),
        "ship" => return Err("arg"),
        "deliver" => Event::Deliver,
        "cancel" if !arg.is_empty() => Event::Cancel(arg.to_string()),
        "cancel" => return Err("arg"),
        _ => return Err("event"),
    };
    Ok((oid, ev))
}

/// T4 - (current state or None, event) -> new state or the reason it is illegal.
fn apply(state: Option<&State>, ev: Event) -> Result<State, &'static str> {
    match (state, ev) {
        (None, Event::Create(amount)) => Ok(State::Created { amount }),
        (None, _) => Err("unknown_order"),
        (Some(_), Event::Create(_)) => Err("duplicate"),
        (Some(State::Created { amount }), Event::Pay) => Ok(State::Paid { amount: *amount }),
        (Some(State::Paid { amount }), Event::Ship(tracking)) => {
            Ok(State::Shipped { amount: *amount, tracking })
        }
        (Some(State::Shipped { amount, .. }), Event::Deliver) => {
            Ok(State::Delivered { amount: *amount })
        }
        (Some(State::Created { .. } | State::Paid { .. }), Event::Cancel(reason)) => {
            Ok(State::Cancelled { reason })
        }
        _ => Err("transition"),
    }
}

/// T6 - (name, detail) for the report; the compiler demands one arm per variant.
fn describe(s: &State) -> (&'static str, String) {
    match s {
        State::Created { amount } => ("created", format!("amount={amount}")),
        State::Paid { amount } => ("paid", format!("amount={amount}")),
        State::Shipped { amount, tracking } => ("shipped", format!("amount={amount} tracking={tracking}")),
        State::Delivered { amount } => ("delivered", format!("amount={amount}")),
        State::Cancelled { reason } => ("cancelled", format!("reason={reason}")),
    }
}

const NAMES: [&str; 5] = ["created", "paid", "shipped", "delivered", "cancelled"];

fn run(path: &str) -> std::io::Result<u8> {
    let mut orders: HashMap<String, State> = HashMap::new(); // T5 - order id -> State
    let (mut events, mut applied, mut rejected) = (0, 0, 0);
    for (n, line) in read_rows(path)? {
        events += 1;
        let (oid, ev) = match parse(&line) {
            Ok(x) => x,
            Err(reason) => {
                rejected += 1;
                eprintln!("reject line={n} reason={reason}");
                continue;
            }
        };
        match apply(orders.get(&oid), ev) {
            Ok(new) => {
                orders.insert(oid, new);
                applied += 1;
            }
            Err(reason) => {
                rejected += 1;
                eprintln!("reject line={n} reason={reason}");
            }
        }
    }
    // T7 - totals per variant and the final report
    let mut counts = [0usize; 5];
    let (mut delivered_total, mut open_total) = (Money { cents: 0 }, Money { cents: 0 });
    let mut ids: Vec<&String> = orders.keys().collect();
    ids.sort();
    let mut lines = Vec::new();
    for oid in ids {
        let st = &orders[oid];
        let (name, detail) = describe(st);
        counts[NAMES.iter().position(|n| *n == name).unwrap()] += 1;
        match st {
            State::Delivered { amount } => delivered_total = delivered_total.add(*amount),
            State::Created { amount } | State::Paid { amount } | State::Shipped { amount, .. } => {
                open_total = open_total.add(*amount)
            }
            State::Cancelled { .. } => {}
        }
        lines.push(format!("order={oid} state={name} {detail}"));
    }
    println!("events={events} applied={applied} rejected={rejected}");
    let summary: Vec<String> = NAMES.iter().zip(counts).map(|(k, v)| format!("{k}={v}")).collect();
    println!("orders={} {}", orders.len(), summary.join(" "));
    println!("delivered_total={delivered_total} open_total={open_total}");
    println!("{}", lines.join("\n"));
    Ok(if rejected > 0 { 1 } else { 0 })
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("usage: main FILE");
        return ExitCode::from(2);
    }
    match run(&args[1]) {
        Ok(code) => ExitCode::from(code),
        Err(_) => {
            eprintln!("cannot open {}", args[1]);
            ExitCode::from(2)
        }
    }
}
```

#### Go (`main.go`) — `go build -o main_go . && ./main_go events.csv`

```go
package main

import (
	"bufio"
	"fmt"
	"iter"
	"os"
	"sort"
	"strconv"
	"strings"
)

// Money is T3 - a value object: an amount in euro cents. Copied by value, compared with ==.
type Money struct{ cents int64 }

func (m Money) Add(o Money) Money { return Money{m.cents + o.cents} }

func (m Money) String() string { return fmt.Sprintf("%d.%02d", m.cents/100, m.cents%100) }

// State is a closed set of variants emulated with an interface: only types in
// this package can implement the unexported method.
type State interface{ isState() }

type Created struct{ Amount Money }
type Paid struct{ Amount Money }
type Shipped struct {
	Amount   Money
	Tracking string
}
type Delivered struct{ Amount Money }
type Cancelled struct{ Reason string }

func (Created) isState()   {}
func (Paid) isState()      {}
func (Shipped) isState()   {}
func (Delivered) isState() {}
func (Cancelled) isState() {}

type Event interface{ isEvent() }

type Create struct{ Amount Money }
type Pay struct{}
type Ship struct{ Tracking string }
type Deliver struct{}
type Cancel struct{ Reason string }

func (Create) isEvent()  {}
func (Pay) isEvent()     {}
func (Ship) isEvent()    {}
func (Deliver) isEvent() {}
func (Cancel) isEvent()  {}

// readRows is T1 - lazy: an iter.Seq2 of (line_number, text), header skipped.
func readRows(path string) (iter.Seq2[int, string], error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	return func(yield func(int, string) bool) {
		defer f.Close()
		sc := bufio.NewScanner(f)
		for n := 1; sc.Scan(); n++ {
			t := strings.TrimSpace(sc.Text())
			if n == 1 || t == "" {
				continue
			}
			if !yield(n, t) {
				return
			}
		}
	}, nil
}

// parse is T2 - returns the order id, the event, or a rejection reason.
func parse(line string) (string, Event, string) {
	p := strings.Split(line, ",")
	for i := range p {
		p[i] = strings.TrimSpace(p[i])
	}
	if len(p) != 3 || p[0] == "" {
		return "", nil, "fields"
	}
	oid, arg := p[0], p[2]
	switch p[1] {
	case "create":
		if arg == "" || strings.Trim(arg, "0123456789") != "" {
			return "", nil, "arg"
		}
		c, err := strconv.ParseInt(arg, 10, 64)
		if err != nil || c == 0 {
			return "", nil, "arg"
		}
		return oid, Create{Money{c}}, ""
	case "pay":
		return oid, Pay{}, ""
	case "ship":
		if arg == "" {
			return "", nil, "arg"
		}
		return oid, Ship{arg}, ""
	case "deliver":
		return oid, Deliver{}, ""
	case "cancel":
		if arg == "" {
			return "", nil, "arg"
		}
		return oid, Cancel{arg}, ""
	}
	return "", nil, "event"
}

// apply is T4 - (current state or nil, event) -> new state or the reason it is illegal.
func apply(s State, ev Event) (State, string) {
	if s == nil {
		if c, ok := ev.(Create); ok {
			return Created{c.Amount}, ""
		}
		return nil, "unknown_order"
	}
	if _, ok := ev.(Create); ok {
		return nil, "duplicate"
	}
	switch st := s.(type) {
	case Created:
		switch e := ev.(type) {
		case Pay:
			return Paid{st.Amount}, ""
		case Cancel:
			return Cancelled{e.Reason}, ""
		}
	case Paid:
		switch e := ev.(type) {
		case Ship:
			return Shipped{st.Amount, e.Tracking}, ""
		case Cancel:
			return Cancelled{e.Reason}, ""
		}
	case Shipped:
		if _, ok := ev.(Deliver); ok {
			return Delivered{st.Amount}, ""
		}
	}
	return nil, "transition"
}

// describe is T6 - (name, detail) for the report. Nothing forces one case per variant.
func describe(s State) (string, string) {
	switch st := s.(type) {
	case Created:
		return "created", fmt.Sprintf("amount=%s", st.Amount)
	case Paid:
		return "paid", fmt.Sprintf("amount=%s", st.Amount)
	case Shipped:
		return "shipped", fmt.Sprintf("amount=%s tracking=%s", st.Amount, st.Tracking)
	case Delivered:
		return "delivered", fmt.Sprintf("amount=%s", st.Amount)
	case Cancelled:
		return "cancelled", fmt.Sprintf("reason=%s", st.Reason)
	}
	panic("unreachable: unknown State variant")
}

var names = []string{"created", "paid", "shipped", "delivered", "cancelled"}

func run(path string) int {
	rows, err := readRows(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "cannot open %s\n", path)
		return 2
	}
	orders := map[string]State{} // T5 - order id -> current State
	var events, applied, rejected int
	for n, line := range rows {
		events++
		oid, ev, reason := parse(line)
		if reason == "" {
			var next State
			if next, reason = apply(orders[oid], ev); reason == "" {
				orders[oid] = next
				applied++
				continue
			}
		}
		rejected++
		fmt.Fprintf(os.Stderr, "reject line=%d reason=%s\n", n, reason)
	}
	// T7 - totals per variant and the final report
	counts := map[string]int{}
	var deliveredTotal, openTotal Money
	ids := make([]string, 0, len(orders))
	for oid := range orders {
		ids = append(ids, oid)
	}
	sort.Strings(ids)
	var lines []string
	for _, oid := range ids {
		st := orders[oid]
		name, detail := describe(st)
		counts[name]++
		switch s := st.(type) {
		case Delivered:
			deliveredTotal = deliveredTotal.Add(s.Amount)
		case Created:
			openTotal = openTotal.Add(s.Amount)
		case Paid:
			openTotal = openTotal.Add(s.Amount)
		case Shipped:
			openTotal = openTotal.Add(s.Amount)
		}
		lines = append(lines, fmt.Sprintf("order=%s state=%s %s", oid, name, detail))
	}
	fmt.Printf("events=%d applied=%d rejected=%d\n", events, applied, rejected)
	summary := make([]string, 0, len(names))
	for _, k := range names {
		summary = append(summary, fmt.Sprintf("%s=%d", k, counts[k]))
	}
	fmt.Printf("orders=%d %s\n", len(orders), strings.Join(summary, " "))
	fmt.Printf("delivered_total=%s open_total=%s\n", deliveredTotal, openTotal)
	fmt.Println(strings.Join(lines, "\n"))
	if rejected > 0 {
		return 1
	}
	return 0
}

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: main FILE")
		os.Exit(2)
	}
	os.Exit(run(os.Args[1]))
}
```

**Limiti dichiarati di questa bozza:** il comportamento di `dataclass` non frozen come chiave è riportato da una prova minima, non da test sistematici; il controllo di esaustività di Python è provato con mypy 2.4.0 `--strict` solo sull'esperimento e4, non sul programma intero; PHPStan/Psalm e i linter di esaustività per Go non sono stati eseguiti; il `readonly class` richiede PHP ≥ 8.2 (la baseline del corso è 8.4+); `sum type` per Go resta emulato, e nessuna libreria esterna è stata provata.
