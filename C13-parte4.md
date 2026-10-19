> File C13-parte4.md (parte 4 di 5) — C13 Testing e qualità, corso di programmazione multi-linguaggio (PHP, Python, Rust, Go)

### Python — `cd py && python3 -m parking ../tickets.csv && python3 -m unittest discover -s tests -t . && python3 -m pytest tests/test_props.py`

#### `parking/core.py`

```python
"""Parking fee calculator: pure functions plus one injected dependency."""
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Protocol

FREE_MINUTES = 15       # no charge up to and including this many minutes
DAY_MINUTES = 1440      # one parking "day"
CAP_HOURS = 8           # daily cap = 8 hours at the zone rate


class RateSource(Protocol):
    """Where hourly rates (in cents) come from."""

    def rate(self, zone: str) -> int | None: ...


class StaticRates:
    def __init__(self, table: dict[str, int]) -> None:
        self._table = dict(table)

    def rate(self, zone: str) -> int | None:
        return self._table.get(zone)


def parse_time(text: str) -> int | None:
    """T2 - 'HH:MM' -> minutes since midnight, or None if malformed."""
    parts = text.split(":")
    if len(parts) != 2 or any(len(p) != 2 or not (p.isascii() and p.isdigit()) for p in parts):
        return None
    hh, mm = int(parts[0]), int(parts[1])
    if not (0 <= hh <= 23 and 0 <= mm <= 59):
        return None
    return hh * 60 + mm


def duration(t_in: int, t_out: int) -> int:
    """T3 - minutes parked; an exit earlier than the entry means next day."""
    d = t_out - t_in
    return d if d >= 0 else d + DAY_MINUTES


def fee(minutes: int, rate: int) -> int:
    """T4 - fee in cents: first 15 min free, started hours, cap per day."""
    cap = CAP_HOURS * rate
    days, rem = divmod(minutes, DAY_MINUTES)
    part = 0 if rem <= FREE_MINUTES else rate * -(-rem // 60)
    return days * cap + min(cap, part)


@dataclass(frozen=True)
class Ticket:
    id: str
    zone: str
    minutes: int
    cents: int


def parse_line(line: str, rates: RateSource) -> Ticket | str:
    """T5 - one CSV line -> Ticket, or the reject reason as a string."""
    fields = [f.strip() for f in line.split(",")]
    if len(fields) != 4 or not fields[0] or not fields[1]:
        return "fields"
    t_in, t_out = parse_time(fields[2]), parse_time(fields[3])
    if t_in is None or t_out is None:
        return "time"
    rate = rates.rate(fields[1])
    if rate is None:
        return "zone"
    minutes = duration(t_in, t_out)
    return Ticket(fields[0], fields[1], minutes, fee(minutes, rate))


@dataclass
class Report:
    tickets: int = 0
    valid: list[Ticket] = field(default_factory=list)
    rejects: list[tuple[int, str]] = field(default_factory=list)


def run(lines: list[str], rates: RateSource) -> Report:
    """T6 - process every non-empty line; ids must be unique."""
    rep, seen = Report(), set()
    for n, raw in enumerate(lines, 1):
        line = raw.strip()
        if not line:
            continue
        rep.tickets += 1
        res = parse_line(line, rates)
        if isinstance(res, str):
            rep.rejects.append((n, res))
        elif res.id in seen:
            rep.rejects.append((n, "duplicate"))
        else:
            seen.add(res.id)
            rep.valid.append(res)
    return rep
```

#### `parking/__main__.py`

```python
import sys

from .core import StaticRates, run

RATES = {"A": 120, "B": 80, "C": 200}


def main(argv: list[str]) -> int:
    """T7 - CLI: report on stdout, rejects on stderr, exit code as contract."""
    try:
        with open(argv[1], encoding="utf-8") as fh:
            lines = fh.read().splitlines()
    except OSError:
        print(f"error: cannot read {argv[1]}", file=sys.stderr)
        return 2
    rep = run(lines, StaticRates(RATES))
    for n, reason in rep.rejects:
        print(f"line {n}: rejected ({reason})", file=sys.stderr)
    total = sum(t.cents for t in rep.valid)
    print(f"tickets={rep.tickets} valid={len(rep.valid)} rejected={len(rep.rejects)} total={total}")
    for zone in sorted({t.zone for t in rep.valid}):
        zt = [t for t in rep.valid if t.zone == zone]
        print(f"zone={zone} count={len(zt)} total={sum(t.cents for t in zt)}")
    return 1 if rep.rejects else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

#### `tests/test_core.py`

```python
import unittest

from parking.core import (DAY_MINUTES, StaticRates, Ticket, duration, fee,
                          parse_line, parse_time, run)


class SpyRates:
    """Test double: records every lookup (spy) and answers from a table (stub)."""

    def __init__(self, table):
        self.table, self.calls = table, []

    def rate(self, zone):
        self.calls.append(zone)
        return self.table.get(zone)


class FeeTest(unittest.TestCase):
    def test_table(self):
        # (minutes, rate, expected cents): boundaries first
        cases = [
            (0, 120, 0), (15, 120, 0), (16, 120, 120), (60, 120, 120),
            (61, 120, 240), (480, 120, 960), (481, 120, 960),  # daily cap 8*120
            (1439, 120, 960), (1440, 120, 960), (1455, 120, 960),
            (1456, 120, 1080), (2880, 80, 1280),
        ]
        for minutes, rate, want in cases:
            with self.subTest(minutes=minutes, rate=rate):
                self.assertEqual(fee(minutes, rate), want)


class TimeTest(unittest.TestCase):
    def test_valid(self):
        self.assertEqual(parse_time("00:00"), 0)
        self.assertEqual(parse_time("23:59"), 1439)

    def test_invalid(self):
        for bad in ["", "9:05", "09:5", "24:00", "12:60", "12-30", "ab:cd", " 9:05", "09:05:00", "+9:05"]:
            with self.subTest(text=bad):
                self.assertIsNone(parse_time(bad))

    def test_duration_wraps_midnight(self):
        self.assertEqual(duration(22 * 60, 6 * 60), 8 * 60)
        self.assertEqual(duration(600, 600), 0)


class RunTest(unittest.TestCase):
    def test_reject_reasons(self):
        spy = SpyRates({"A": 120})
        rep = run(["t1,A,08:00,09:00", "t2,Z,08:00,09:00", "t3,A,8:00,09:00",
                   "t4,A,08:00", "t1,A,10:00,11:00", "", "  "], spy)
        self.assertEqual(rep.tickets, 5)
        self.assertEqual(rep.valid, [Ticket("t1", "A", 60, 120)])
        self.assertEqual(rep.rejects, [(2, "zone"), (3, "time"), (4, "fields"), (5, "duplicate")])

    def test_rate_source_is_asked_only_for_well_formed_lines(self):
        spy = SpyRates({"A": 120})
        run(["t1,A,08:00,09:00", "t2,B,99:00,09:00"], spy)
        self.assertEqual(spy.calls, ["A"])  # the malformed line never reaches the source

    def test_parse_line_with_static_rates(self):
        t = parse_line("x, B ,10:00,12:30", StaticRates({"B": 80}))
        self.assertEqual(t, Ticket("x", "B", 150, 240))


if __name__ == "__main__":
    unittest.main()
```

#### `tests/test_props.py`

```python
from hypothesis import given, strategies as st

from parking.core import CAP_HOURS, DAY_MINUTES, FREE_MINUTES, duration, fee, parse_time

minutes = st.integers(min_value=0, max_value=10 * DAY_MINUTES)
rates = st.integers(min_value=1, max_value=1000)


@given(minutes, rates)
def test_free_window(m, r):
    # P1: parking up to the free window costs nothing
    if m <= FREE_MINUTES:
        assert fee(m, r) == 0


@given(minutes, minutes, rates)
def test_monotonic(a, b, r):
    # P2: staying longer never costs less
    lo, hi = sorted((a, b))
    assert fee(lo, r) <= fee(hi, r)


@given(minutes, rates)
def test_cap_per_day(m, r):
    # P3: never more than one daily cap per started day
    days_started = -(-m // DAY_MINUTES)
    assert fee(m, r) <= days_started * CAP_HOURS * r


@given(st.integers(0, 1439), st.integers(0, 1439))
def test_duration_range(t_in, t_out):
    # P4: a duration is always within one day
    assert 0 <= duration(t_in, t_out) < DAY_MINUTES


@given(st.text())
def test_parse_time_accepts_only_hh_mm(s):
    # P5: if it parses, it had the exact shape HH:MM in ASCII digits
    v = parse_time(s)
    if v is not None:
        assert len(s) == 5 and s[2] == ":" and s.replace(":", "").isascii() and s.replace(":", "").isdigit()
        assert f"{v // 60:02d}:{v % 60:02d}" == s
```

#### `tests/test_cli.py`

```python
import io
import contextlib
import pathlib
import tempfile
import unittest

from parking.__main__ import main

CSV = "a,B,08:00,09:00\nb,A,08:00,09:30\nc,A,08:00\nd,B,08:00,08:20\n"


def call(argv):
    out, err = io.StringIO(), io.StringIO()
    with contextlib.redirect_stdout(out), contextlib.redirect_stderr(err):
        code = main(argv)
    return code, out.getvalue(), err.getvalue()


class CliTest(unittest.TestCase):
    def test_golden_output_and_exit_code(self):
        with tempfile.TemporaryDirectory() as d:
            p = pathlib.Path(d, "t.csv")
            p.write_text(CSV)
            code, out, err = call(["prog", str(p)])
        self.assertEqual(code, 1)
        self.assertEqual(out, "tickets=4 valid=3 rejected=1 total=400\nzone=A count=1 total=240\nzone=B count=2 total=160\n")
        self.assertEqual(err, "line 3: rejected (fields)\n")

    def test_missing_file(self):
        code, out, err = call(["prog", "/nonexistent/x.csv"])
        self.assertEqual((code, out, err), (2, "", "error: cannot read /nonexistent/x.csv\n"))
```

### PHP — `cd php && php main.php ../tickets.csv && php tests/run.php`

#### `src/core.php`

```php
<?php
declare(strict_types=1);

namespace Parking;

const FREE_MINUTES = 15;   // no charge up to and including this many minutes
const DAY_MINUTES = 1440;  // one parking "day"
const CAP_HOURS = 8;       // daily cap = 8 hours at the zone rate

interface RateSource
{
    public function rate(string $zone): ?int;
}

final class StaticRates implements RateSource
{
    /** @param array<string,int> $table */
    public function __construct(private array $table) {}

    public function rate(string $zone): ?int
    {
        return $this->table[$zone] ?? null;
    }
}

/** T2 - 'HH:MM' -> minutes since midnight, or null if malformed. */
function parseTime(string $text): ?int
{
    if (preg_match('/\A(\d\d):(\d\d)\z/', $text, $m) !== 1) {
        return null;
    }
    $hh = (int) $m[1];
    $mm = (int) $m[2];
    if ($hh > 23 || $mm > 59) {
        return null;
    }
    return $hh * 60 + $mm;
}

/** T3 - minutes parked; an exit earlier than the entry means next day. */
function duration(int $in, int $out): int
{
    $d = $out - $in;
    return $d >= 0 ? $d : $d + DAY_MINUTES;
}

/** T4 - fee in cents: first 15 min free, started hours, cap per day. */
function fee(int $minutes, int $rate): int
{
    $cap = CAP_HOURS * $rate;
    $days = intdiv($minutes, DAY_MINUTES);
    $rem = $minutes % DAY_MINUTES;
    $part = $rem <= FREE_MINUTES ? 0 : $rate * intdiv($rem + 59, 60);
    return $days * $cap + min($cap, $part);
}

final class Ticket
{
    public function __construct(
        public readonly string $id,
        public readonly string $zone,
        public readonly int $minutes,
        public readonly int $cents,
    ) {}
}

/** T5 - one CSV line -> Ticket, or the reject reason as a string. */
function parseLine(string $line, RateSource $rates): Ticket|string
{
    $f = array_map('trim', explode(',', $line));
    if (count($f) !== 4 || $f[0] === '' || $f[1] === '') {
        return 'fields';
    }
    $in = parseTime($f[2]);
    $out = parseTime($f[3]);
    if ($in === null || $out === null) {
        return 'time';
    }
    $rate = $rates->rate($f[1]);
    if ($rate === null) {
        return 'zone';
    }
    $minutes = duration($in, $out);
    return new Ticket($f[0], $f[1], $minutes, fee($minutes, $rate));
}

final class Report
{
    public int $tickets = 0;
    /** @var list<Ticket> */
    public array $valid = [];
    /** @var list<array{int,string}> */
    public array $rejects = [];
}

/** T6 - process every non-empty line; ids must be unique. */
function run(array $lines, RateSource $rates): Report
{
    $rep = new Report();
    $seen = [];
    foreach ($lines as $i => $raw) {
        $line = trim($raw);
        if ($line === '') {
            continue;
        }
        $n = $i + 1;
        $rep->tickets++;
        $res = parseLine($line, $rates);
        if (is_string($res)) {
            $rep->rejects[] = [$n, $res];
        } elseif (isset($seen[$res->id])) {
            $rep->rejects[] = [$n, 'duplicate'];
        } else {
            $seen[$res->id] = true;
            $rep->valid[] = $res;
        }
    }
    return $rep;
}
```

#### `main.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . '/src/core.php';

use Parking\StaticRates;

use function Parking\run;

/** T7 - CLI: report on stdout, rejects on stderr, exit code as contract. */
function main(array $argv): int
{
    $text = @file_get_contents($argv[1] ?? '');
    if ($text === false) {
        fwrite(STDERR, "error: cannot read " . ($argv[1] ?? '') . "\n");
        return 2;
    }
    $lines = preg_split('/\r\n|\n|\r/', $text);
    $rep = run($lines, new StaticRates(['A' => 120, 'B' => 80, 'C' => 200]));
    foreach ($rep->rejects as [$n, $reason]) {
        fwrite(STDERR, "line $n: rejected ($reason)\n");
    }
    $total = array_sum(array_map(fn($t) => $t->cents, $rep->valid));
    printf("tickets=%d valid=%d rejected=%d total=%d\n", $rep->tickets, count($rep->valid), count($rep->rejects), $total);
    $zones = array_unique(array_map(fn($t) => $t->zone, $rep->valid));
    sort($zones);
    foreach ($zones as $z) {
        $zt = array_filter($rep->valid, fn($t) => $t->zone === $z);
        printf("zone=%s count=%d total=%d\n", $z, count($zt), array_sum(array_map(fn($t) => $t->cents, $zt)));
    }
    return count($rep->rejects) > 0 ? 1 : 0;
}

exit(main($argv));
```

#### `tests/run.php`

```php
<?php
declare(strict_types=1);

// PHPUnit is not installable here (Packagist is blocked): a 30-line harness shows the mechanics.
require __DIR__ . '/../src/core.php';

use Parking\{RateSource, StaticRates, Ticket};

use function Parking\{duration, fee, parseLine, parseTime, run};

$failures = 0;
$count = 0;

function check(string $name, mixed $got, mixed $want): void
{
    global $failures, $count;
    $count++;
    if ($got !== $want) {
        $failures++;
        fwrite(STDERR, "FAIL $name: got " . var_export($got, true) . ", want " . var_export($want, true) . "\n");
    }
}

/** Test double: records every lookup (spy) and answers from a table (stub). */
final class SpyRates implements RateSource
{
    public array $calls = [];

    public function __construct(private array $table) {}

    public function rate(string $zone): ?int
    {
        $this->calls[] = $zone;
        return $this->table[$zone] ?? null;
    }
}

$cases = [
    [0, 120, 0], [15, 120, 0], [16, 120, 120], [60, 120, 120], [61, 120, 240],
    [480, 120, 960], [481, 120, 960], [1439, 120, 960], [1440, 120, 960],
    [1455, 120, 960], [1456, 120, 1080], [2880, 80, 1280],
];
foreach ($cases as [$m, $r, $want]) {
    check("fee($m,$r)", fee($m, $r), $want);
}

check('00:00', parseTime('00:00'), 0);
check('23:59', parseTime('23:59'), 1439);
foreach (['', '9:05', '09:5', '24:00', '12:60', '12-30', 'ab:cd', ' 9:05', '09:05:00', '+9:05', "09:05\n"] as $bad) {
    check('bad ' . json_encode($bad), parseTime($bad), null);
}
check('wrap', duration(22 * 60, 6 * 60), 8 * 60);

$spy = new SpyRates(['A' => 120]);
$rep = run(['t1,A,08:00,09:00', 't2,Z,08:00,09:00', 't3,A,8:00,09:00', 't4,A,08:00', 't1,A,10:00,11:00', '', '  '], $spy);
check('tickets', $rep->tickets, 5);
check('valid', count($rep->valid), 1);
check('rejects', $rep->rejects, [[2, 'zone'], [3, 'time'], [4, 'fields'], [5, 'duplicate']]);
$spy2 = new SpyRates(['A' => 120]);
run(['t1,A,08:00,09:00', 't2,B,99:00,09:00'], $spy2);
check('spy', $spy2->calls, ['A']);
$t = parseLine('x, B ,10:00,12:30', new StaticRates(['B' => 80]));
check('static', [$t->id, $t->minutes, $t->cents], ['x', 150, 240]);

// Property tests by hand: a seeded generator, so a failure can be replayed.
mt_srand(12345);
for ($i = 0; $i < 2000; $i++) {
    $r = mt_rand(1, 1000);
    $a = mt_rand(0, 14400);
    $b = mt_rand(0, 14400);
    [$lo, $hi] = $a <= $b ? [$a, $b] : [$b, $a];
    check("P2 seed-iter $i", fee($lo, $r) <= fee($hi, $r), true);
    check("P3 seed-iter $i", fee($a, $r) <= intdiv($a + 1439, 1440) * 8 * $r, true);
    if ($a <= 15) {
        check("P1 seed-iter $i", fee($a, $r), 0);
    }
}

echo "$count checks, $failures failed\n";
exit($failures > 0 ? 1 : 0);
```

### Rust — `cd rs && cargo run --release -- ../tickets.csv && cargo test`

#### `Cargo.toml`

```toml
[package]
name = "parking"
version = "0.1.0"
edition = "2024"

[dev-dependencies]
proptest = "1"
```

#### `src/lib.rs`

```rust
//! Parking fee calculator: pure functions plus one injected dependency.
use std::collections::{BTreeMap, HashMap, HashSet};

pub const FREE_MINUTES: u32 = 15; // no charge up to and including this many minutes
pub const DAY_MINUTES: u32 = 1440; // one parking "day"
pub const CAP_HOURS: u32 = 8; // daily cap = 8 hours at the zone rate

/// Where hourly rates (in cents) come from.
pub trait RateSource {
    fn rate(&self, zone: &str) -> Option<u32>;
}

pub struct StaticRates(pub HashMap<String, u32>);

impl RateSource for StaticRates {
    fn rate(&self, zone: &str) -> Option<u32> {
        self.0.get(zone).copied()
    }
}

/// T2 - 'HH:MM' -> minutes since midnight, or None if malformed.
pub fn parse_time(text: &str) -> Option<u32> {
    let (h, m) = text.split_once(':')?;
    let two_digits = |s: &str| s.len() == 2 && s.bytes().all(|b| b.is_ascii_digit());
    if !two_digits(h) || !two_digits(m) {
        return None;
    }
    let (hh, mm): (u32, u32) = (h.parse().ok()?, m.parse().ok()?);
    if hh > 23 || mm > 59 {
        return None;
    }
    Some(hh * 60 + mm)
}

/// T3 - minutes parked; an exit earlier than the entry means next day.
pub fn duration(t_in: u32, t_out: u32) -> u32 {
    if t_out >= t_in {
        t_out - t_in
    } else {
        t_out + DAY_MINUTES - t_in
    }
}

/// T4 - fee in cents: first 15 min free, started hours, cap per day.
pub fn fee(minutes: u32, rate: u32) -> u32 {
    let cap = CAP_HOURS * rate;
    let (days, rem) = (minutes / DAY_MINUTES, minutes % DAY_MINUTES);
    let part = if rem <= FREE_MINUTES {
        0
    } else {
        rate * rem.div_ceil(60)
    };
    days * cap + part.min(cap)
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Ticket {
    pub id: String,
    pub zone: String,
    pub minutes: u32,
    pub cents: u32,
}

/// T5 - one CSV line -> Ticket, or the reject reason.
pub fn parse_line(line: &str, rates: &dyn RateSource) -> Result<Ticket, &'static str> {
    let f: Vec<&str> = line.split(',').map(str::trim).collect();
    if f.len() != 4 || f[0].is_empty() || f[1].is_empty() {
        return Err("fields");
    }
    let (Some(t_in), Some(t_out)) = (parse_time(f[2]), parse_time(f[3])) else {
        return Err("time");
    };
    let rate = rates.rate(f[1]).ok_or("zone")?;
    let minutes = duration(t_in, t_out);
    Ok(Ticket {
        id: f[0].into(),
        zone: f[1].into(),
        minutes,
        cents: fee(minutes, rate),
    })
}

#[derive(Debug, Default)]
pub struct Report {
    pub tickets: usize,
    pub valid: Vec<Ticket>,
    pub rejects: Vec<(usize, &'static str)>,
}

/// T6 - process every non-empty line; ids must be unique.
pub fn run(lines: &[&str], rates: &dyn RateSource) -> Report {
    let mut rep = Report::default();
    let mut seen = HashSet::new();
    for (i, raw) in lines.iter().enumerate() {
        let line = raw.trim();
        if line.is_empty() {
            continue;
        }
        rep.tickets += 1;
        match parse_line(line, rates) {
            Err(reason) => rep.rejects.push((i + 1, reason)),
            Ok(t) if !seen.insert(t.id.clone()) => rep.rejects.push((i + 1, "duplicate")),
            Ok(t) => rep.valid.push(t),
        }
    }
    rep
}

pub fn per_zone(rep: &Report) -> BTreeMap<&str, (usize, u32)> {
    let mut m: BTreeMap<&str, (usize, u32)> = BTreeMap::new();
    for t in &rep.valid {
        let e = m.entry(t.zone.as_str()).or_default();
        e.0 += 1;
        e.1 += t.cents;
    }
    m
}

#[cfg(test)]
mod tests;
```

#### `src/main.rs`

```rust
use std::collections::HashMap;
use std::process::ExitCode;

use parking::{StaticRates, per_zone, run};

/// T7 - CLI: report on stdout, rejects on stderr, exit code as contract.
fn main() -> ExitCode {
    let path = std::env::args().nth(1).unwrap_or_default();
    let Ok(text) = std::fs::read_to_string(&path) else {
        eprintln!("error: cannot read {path}");
        return ExitCode::from(2);
    };
    let lines: Vec<&str> = text.lines().collect();
    let rates = StaticRates(HashMap::from([
        ("A".into(), 120),
        ("B".into(), 80),
        ("C".into(), 200),
    ]));
    let rep = run(&lines, &rates);
    for (n, reason) in &rep.rejects {
        eprintln!("line {n}: rejected ({reason})");
    }
    let total: u32 = rep.valid.iter().map(|t| t.cents).sum();
    println!(
        "tickets={} valid={} rejected={} total={}",
        rep.tickets,
        rep.valid.len(),
        rep.rejects.len(),
        total
    );
    for (zone, (count, sum)) in per_zone(&rep) {
        println!("zone={zone} count={count} total={sum}");
    }
    ExitCode::from(if rep.rejects.is_empty() { 0 } else { 1 })
}
```

#### `src/tests.rs`

```rust
use super::*;

use std::cell::RefCell;

/// Test double: records every lookup (spy) and answers from a table (stub).
struct SpyRates {
    table: HashMap<String, u32>,
    calls: RefCell<Vec<String>>,
}

impl RateSource for SpyRates {
    fn rate(&self, zone: &str) -> Option<u32> {
        self.calls.borrow_mut().push(zone.to_string());
        self.table.get(zone).copied()
    }
}

fn spy(pairs: &[(&str, u32)]) -> SpyRates {
    SpyRates {
        table: pairs.iter().map(|(z, r)| (z.to_string(), *r)).collect(),
        calls: RefCell::new(vec![]),
    }
}

#[test]
fn fee_table() {
    // (minutes, rate, expected cents): boundaries first
    let cases = [
        (0, 120, 0),
        (15, 120, 0),
        (16, 120, 120),
        (60, 120, 120),
        (61, 120, 240),
        (480, 120, 960),
        (481, 120, 960),
        (1439, 120, 960),
        (1440, 120, 960),
        (1455, 120, 960),
        (1456, 120, 1080),
        (2880, 80, 1280),
    ];
    for (minutes, rate, want) in cases {
        assert_eq!(fee(minutes, rate), want, "fee({minutes}, {rate})");
    }
}

#[test]
fn time_valid_and_invalid() {
    assert_eq!(parse_time("00:00"), Some(0));
    assert_eq!(parse_time("23:59"), Some(1439));
    for bad in [
        "", "9:05", "09:5", "24:00", "12:60", "12-30", "ab:cd", " 9:05", "09:05:00", "+9:05",
    ] {
        assert_eq!(parse_time(bad), None, "{bad:?}");
    }
}

#[test]
fn duration_wraps_midnight() {
    assert_eq!(duration(22 * 60, 6 * 60), 8 * 60);
    assert_eq!(duration(600, 600), 0);
}

#[test]
fn reject_reasons() {
    let rates = spy(&[("A", 120)]);
    let rep = run(
        &[
            "t1,A,08:00,09:00",
            "t2,Z,08:00,09:00",
            "t3,A,8:00,09:00",
            "t4,A,08:00",
            "t1,A,10:00,11:00",
            "",
            "  ",
        ],
        &rates,
    );
    assert_eq!(rep.tickets, 5);
    assert_eq!(
        rep.valid,
        vec![Ticket {
            id: "t1".into(),
            zone: "A".into(),
            minutes: 60,
            cents: 120
        }]
    );
    assert_eq!(
        rep.rejects,
        vec![(2, "zone"), (3, "time"), (4, "fields"), (5, "duplicate")]
    );
}

#[test]
fn rate_source_is_asked_only_for_well_formed_lines() {
    let rates = spy(&[("A", 120)]);
    run(&["t1,A,08:00,09:00", "t2,B,99:00,09:00"], &rates);
    assert_eq!(*rates.calls.borrow(), vec!["A".to_string()]);
}
```

#### `tests/props.rs`

```rust
use parking::{CAP_HOURS, DAY_MINUTES, FREE_MINUTES, duration, fee, parse_time};
use proptest::prelude::*;

proptest! {
    // P1: parking up to the free window costs nothing
    #[test]
    fn free_window(m in 0..=FREE_MINUTES, r in 1u32..1000) {
        prop_assert_eq!(fee(m, r), 0);
    }

    // P2: staying longer never costs less
    #[test]
    fn monotonic(a in 0u32..14400, b in 0u32..14400, r in 1u32..1000) {
        let (lo, hi) = if a <= b { (a, b) } else { (b, a) };
        prop_assert!(fee(lo, r) <= fee(hi, r));
    }

    // P3: never more than one daily cap per started day
    #[test]
    fn cap_per_day(m in 0u32..14400, r in 1u32..1000) {
        prop_assert!(fee(m, r) <= m.div_ceil(DAY_MINUTES) * CAP_HOURS * r);
    }

    // P4: a duration is always within one day
    #[test]
    fn duration_range(a in 0u32..1440, b in 0u32..1440) {
        prop_assert!(duration(a, b) < DAY_MINUTES);
    }

    // P5: if it parses, it had the exact shape HH:MM and round-trips
    #[test]
    fn parse_time_round_trip(s in "\\PC{0,8}") {
        if let Some(v) = parse_time(&s) {
            prop_assert_eq!(format!("{:02}:{:02}", v / 60, v % 60), s);
        }
    }
}
```

