> File C12-parte4.md (parte 4 di 4) — capitolo C12, Moduli, package e dipendenze; le parti, lette in ordine, ricompongono capitoli/C12-moduli-package-dipendenze.md

### Python — `cd py && uv sync && uv run python -m salesrep ../sales.csv`

#### `pyproject.toml`

```toml
[project]
name = "salesrep"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["money>=1.3,<2"]

[tool.uv.sources]
money = { path = "libs/money" }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/salesrep"]
```

#### `libs/money/pyproject.toml`

```toml
[project]
name = "money"
version = "1.3.0"
requires-python = ">=3.12"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

#### `libs/money/src/money/__init__.py`

```python
"""money 1.3.0 - parse and format amounts as integer cents."""
from ._core import digits_only as _digits_only

__all__ = ["parse_cents", "format_cents", "__version__"]
__version__ = "1.3.0"


def parse_cents(text: str) -> int:
    """T0 - '12.5' -> 1250. Accepts N, N.D or N.DD; raises ValueError otherwise."""
    whole, dot, frac = text.partition(".")
    if not _digits_only(whole) or (dot and not (1 <= len(frac) <= 2 and _digits_only(frac))):
        raise ValueError(f"invalid amount: {text!r}")
    return int(whole) * 100 + int(frac.ljust(2, "0"))


def format_cents(cents: int) -> str:
    """1250 -> '12.50'."""
    return f"{cents // 100}.{cents % 100:02d}"
```

#### `libs/money/src/money/_core.py`

```python
"""Private helpers of the money package: not part of the public API."""


def digits_only(s: str) -> bool:
    """True for a non-empty string made of ASCII digits only."""
    return s != "" and s.isascii() and s.isdigit()
```

#### `src/salesrep/__init__.py`

```python
"""salesrep: a small sales report built from three modules."""
```

#### `src/salesrep/parsing.py`

```python
"""T2 - turn CSV lines into validated Sale records."""
from dataclasses import dataclass

from money import parse_cents


@dataclass(frozen=True)
class Sale:
    id: int
    region: str
    cents: int


def parse_sales(lines: list[str]) -> tuple[list[Sale], list[tuple[int, str]]]:
    """T2 - validate every line; return (sales, rejects) with rejects as (line_no, reason)."""
    sales: list[Sale] = []
    rejects: list[tuple[int, str]] = []
    seen: set[int] = set()
    for no, raw in enumerate(lines, 1):
        line = raw.strip()
        if no == 1 or not line:
            continue  # header and blank lines
        parts = [p.strip() for p in line.split(",")]
        if len(parts) != 3:
            rejects.append((no, "fields"))
            continue
        sid, region, amount = parts
        if not (sid.isascii() and sid.isdigit()) or int(sid) == 0:
            rejects.append((no, "id"))
        elif int(sid) in seen:
            rejects.append((no, "duplicate"))
        elif region == "":
            rejects.append((no, "region"))
        else:
            try:
                cents = parse_cents(amount)
            except ValueError:
                rejects.append((no, "amount"))
                continue
            seen.add(int(sid))
            sales.append(Sale(int(sid), region, cents))
    return sales, rejects
```

#### `src/salesrep/report.py`

```python
"""T3, T4 - aggregate by region and render the report."""
from collections import defaultdict

from money import format_cents

from .parsing import Sale


def totals_by_region(sales: list[Sale]) -> dict[str, tuple[int, int]]:
    """T3 - region -> (count, total_cents)."""
    acc: dict[str, list[int]] = defaultdict(lambda: [0, 0])
    for s in sales:
        acc[s.region][0] += 1
        acc[s.region][1] += s.cents
    return {r: (c, t) for r, (c, t) in acc.items()}


def render(totals: dict[str, tuple[int, int]], rows: int, rejected: int) -> list[str]:
    """T4 - stable output: regions sorted by name, then the summary lines."""
    out = [f"region {r} count={c} total={format_cents(t)}" for r, (c, t) in sorted(totals.items())]
    valid = sum(c for c, _ in totals.values())
    grand = sum(t for _, t in totals.values())
    out.append(f"rows={rows} valid={valid} rejected={rejected}")
    out.append(f"grand_total={format_cents(grand)}")
    return out
```

#### `src/salesrep/__main__.py`

```python
"""T1, T5 - read the file, run the pipeline, exit with the contract code."""
import sys

from .parsing import parse_sales
from .report import render, totals_by_region


def main(argv: list[str]) -> int:
    if len(argv) != 2:
        print("usage: python -m salesrep sales.csv", file=sys.stderr)
        return 2
    try:
        with open(argv[1], encoding="utf-8") as f:
            lines = f.read().splitlines()  # T1
    except OSError:
        print(f"error: cannot read {argv[1]}", file=sys.stderr)
        return 2
    sales, rejects = parse_sales(lines)
    for no, reason in rejects:
        print(f"reject line {no}: {reason}", file=sys.stderr)
    rows = max(len(lines) - 1, 0)
    print("\n".join(render(totals_by_region(sales), rows, len(rejects))))
    return 1 if rejects else 0  # T5


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

### PHP — `cd php/salesrep && composer install && php bin/main.php ../../sales.csv`

#### `money/composer.json`

```json
{
    "name": "example/money",
    "description": "Parse and format amounts as integer cents",
    "version": "1.3.0",
    "type": "library",
    "require": { "php": ">=8.2" },
    "autoload": { "psr-4": { "Example\\Money\\": "src/" } }
}
```

#### `money/src/Money.php`

```php
<?php
declare(strict_types=1);

namespace Example\Money;

/** money 1.3.0 - parse and format amounts as integer cents. */
final class Money
{
    /** T0 - '12.5' -> 1250, null when the text is not N, N.D or N.DD. */
    public static function parseCents(string $text): ?int
    {
        $parts = explode('.', $text, 2);
        $whole = $parts[0];
        $frac = $parts[1] ?? '';
        if (!self::digitsOnly($whole)) {
            return null;
        }
        if (count($parts) === 2 && !(strlen($frac) >= 1 && strlen($frac) <= 2 && self::digitsOnly($frac))) {
            return null;
        }
        return (int) $whole * 100 + (int) str_pad($frac, 2, '0');
    }

    /** 1250 -> '12.50'. */
    public static function formatCents(int $cents): string
    {
        return sprintf('%d.%02d', intdiv($cents, 100), $cents % 100);
    }

    /** Private helper: not part of the public API. */
    private static function digitsOnly(string $s): bool
    {
        return preg_match('/^[0-9]+$/D', $s) === 1;
    }
}
```

#### `salesrep/composer.json`

```json
{
    "name": "example/salesrep",
    "type": "project",
    "require": { "php": ">=8.2", "example/money": "^1.3" },
    "repositories": [
        { "type": "path", "url": "../money" },
        { "packagist.org": false }
    ],
    "autoload": { "psr-4": { "Sales\\": "src/" } }
}
```

#### `salesrep/src/Sale.php`

```php
<?php
declare(strict_types=1);

namespace Sales;

final class Sale
{
    public function __construct(
        public readonly int $id,
        public readonly string $region,
        public readonly int $cents,
    ) {
    }
}
```

#### `salesrep/src/Parsing.php`

```php
<?php
declare(strict_types=1);

namespace Sales;

use Example\Money\Money;

final class Parsing
{
    /** T2 - validate every line; return [sales, rejects] with rejects as [line_no, reason]. */
    public static function parseSales(array $lines): array
    {
        $sales = [];
        $rejects = [];
        $seen = [];
        foreach ($lines as $i => $raw) {
            $no = $i + 1;
            $line = trim($raw);
            if ($no === 1 || $line === '') {
                continue; // header and blank lines
            }
            $parts = array_map('trim', explode(',', $line));
            if (count($parts) !== 3) {
                $rejects[] = [$no, 'fields'];
                continue;
            }
            [$sid, $region, $amount] = $parts;
            if (preg_match('/^[0-9]+$/D', $sid) !== 1 || (int) $sid === 0) {
                $rejects[] = [$no, 'id'];
            } elseif (isset($seen[(int) $sid])) {
                $rejects[] = [$no, 'duplicate'];
            } elseif ($region === '') {
                $rejects[] = [$no, 'region'];
            } elseif (($cents = Money::parseCents($amount)) === null) {
                $rejects[] = [$no, 'amount'];
            } else {
                $seen[(int) $sid] = true;
                $sales[] = new Sale((int) $sid, $region, $cents);
            }
        }
        return [$sales, $rejects];
    }
}
```

#### `salesrep/src/Report.php`

```php
<?php
declare(strict_types=1);

namespace Sales;

use Example\Money\Money;

final class Report
{
    /** T3 - region => [count, total_cents]. */
    public static function totalsByRegion(array $sales): array
    {
        $acc = [];
        foreach ($sales as $s) {
            $acc[$s->region] ??= [0, 0];
            $acc[$s->region][0]++;
            $acc[$s->region][1] += $s->cents;
        }
        return $acc;
    }

    /** T4 - stable output: regions sorted by name, then the summary lines. */
    public static function render(array $totals, int $rows, int $rejected): array
    {
        ksort($totals, SORT_STRING);
        $out = [];
        $valid = 0;
        $grand = 0;
        foreach ($totals as $r => [$c, $t]) {
            $out[] = "region $r count=$c total=" . Money::formatCents($t);
            $valid += $c;
            $grand += $t;
        }
        $out[] = "rows=$rows valid=$valid rejected=$rejected";
        $out[] = 'grand_total=' . Money::formatCents($grand);
        return $out;
    }
}
```

#### `salesrep/bin/main.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php'; // generated by Composer from the PSR-4 map

use Sales\Parsing;
use Sales\Report;

function main(array $argv): int
{
    if (count($argv) !== 2) {
        fwrite(STDERR, "usage: php bin/main.php sales.csv\n");
        return 2;
    }
    $text = @file_get_contents($argv[1]); // T1
    if ($text === false) {
        fwrite(STDERR, "error: cannot read {$argv[1]}\n");
        return 2;
    }
    $lines = explode("\n", rtrim(str_replace("\r\n", "\n", $text), "\n"));
    [$sales, $rejects] = Parsing::parseSales($lines);
    foreach ($rejects as [$no, $reason]) {
        fwrite(STDERR, "reject line $no: $reason\n");
    }
    $rows = max(count($lines) - 1, 0);
    echo implode("\n", Report::render(Report::totalsByRegion($sales), $rows, count($rejects))), "\n";
    return $rejects ? 1 : 0; // T5
}

exit(main($argv));
```

### Rust — `cd rs && cargo build --release && ./target/release/salesrep ../sales.csv`

#### `Cargo.toml`

```toml
[workspace]
members = ["money", "salesrep"]
resolver = "3"
```

#### `money/Cargo.toml`

```toml
[package]
name = "money"
version = "1.3.0"
edition = "2024"
```

#### `money/src/lib.rs`

```rust
//! money 1.3.0 - parse and format amounts as integer cents.

/// The only error this crate can report.
#[derive(Debug, PartialEq)]
pub enum MoneyError {
    Invalid,
    Overflow,
}

/// T0 - "12.5" -> 1250. Accepts N, N.D or N.DD.
pub fn parse_cents(text: &str) -> Result<i64, MoneyError> {
    let (whole, frac) = match text.split_once('.') {
        Some((w, f)) => (w, Some(f)),
        None => (text, None),
    };
    if !digits_only(whole) {
        return Err(MoneyError::Invalid);
    }
    let frac_cents = match frac {
        None => 0,
        Some(f) if (1..=2).contains(&f.len()) && digits_only(f) => {
            f.parse::<i64>().unwrap() * if f.len() == 1 { 10 } else { 1 }
        }
        Some(_) => return Err(MoneyError::Invalid),
    };
    let whole: i64 = whole.parse().map_err(|_| MoneyError::Overflow)?;
    whole.checked_mul(100).and_then(|w| w.checked_add(frac_cents)).ok_or(MoneyError::Overflow)
}

/// 1250 -> "12.50".
pub fn format_cents(cents: i64) -> String {
    format!("{}.{:02}", cents / 100, cents % 100)
}

// private: not visible outside the crate
fn digits_only(s: &str) -> bool {
    !s.is_empty() && s.bytes().all(|b| b.is_ascii_digit())
}
```

#### `salesrep/Cargo.toml`

```toml
[package]
name = "salesrep"
version = "0.1.0"
edition = "2024"

[dependencies]
money = { path = "../money", version = "1.3" }
```

#### `salesrep/src/main.rs`

```rust
mod parsing; // declares the module: the compiler loads src/parsing.rs
mod report;

use std::process::ExitCode;

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("usage: salesrep sales.csv");
        return ExitCode::from(2);
    }
    let Ok(text) = std::fs::read_to_string(&args[1]) else { // T1
        eprintln!("error: cannot read {}", args[1]);
        return ExitCode::from(2);
    };
    let lines: Vec<&str> = text.lines().collect(); // lines() also strips a trailing \r
    let (sales, rejects) = parsing::parse_sales(&lines);
    for (no, reason) in &rejects {
        eprintln!("reject line {no}: {reason}");
    }
    let rows = lines.len().saturating_sub(1);
    let totals = report::totals_by_region(&sales);
    println!("{}", report::render(&totals, rows, rejects.len()).join("\n"));
    if rejects.is_empty() { ExitCode::SUCCESS } else { ExitCode::from(1) } // T5
}
```

#### `salesrep/src/parsing.rs`

```rust
use std::collections::HashSet;

use money::parse_cents;

#[derive(Debug, Clone)]
pub struct Sale {
    #[allow(dead_code)] // kept for symmetry with the other languages
    pub id: u64,
    pub region: String,
    pub cents: i64,
}

/// T2 - validate every line; return the sales and the rejects as (line_no, reason).
pub fn parse_sales(lines: &[&str]) -> (Vec<Sale>, Vec<(usize, &'static str)>) {
    let mut sales = Vec::new();
    let mut rejects = Vec::new();
    let mut seen = HashSet::new();
    for (i, raw) in lines.iter().enumerate() {
        let no = i + 1;
        let line = raw.trim();
        if no == 1 || line.is_empty() {
            continue; // header and blank lines
        }
        let parts: Vec<&str> = line.split(',').map(str::trim).collect();
        if parts.len() != 3 {
            rejects.push((no, "fields"));
            continue;
        }
        let id = if !parts[0].is_empty() && parts[0].bytes().all(|b| b.is_ascii_digit()) {
            parts[0].parse::<u64>().ok()
        } else {
            None
        };
        match id {
            None | Some(0) => rejects.push((no, "id")),
            Some(id) if seen.contains(&id) => rejects.push((no, "duplicate")),
            Some(_) if parts[1].is_empty() => rejects.push((no, "region")),
            Some(id) => match parse_cents(parts[2]) {
                Ok(cents) => {
                    seen.insert(id);
                    sales.push(Sale { id, region: parts[1].to_string(), cents });
                }
                Err(_) => rejects.push((no, "amount")),
            },
        }
    }
    (sales, rejects)
}
```

#### `salesrep/src/report.rs`

```rust
use std::collections::BTreeMap;

use money::format_cents;

use crate::parsing::Sale;

/// T3 - region -> (count, total_cents). BTreeMap keeps the keys sorted.
pub fn totals_by_region(sales: &[Sale]) -> BTreeMap<String, (u64, i64)> {
    let mut acc: BTreeMap<String, (u64, i64)> = BTreeMap::new();
    for s in sales {
        let e = acc.entry(s.region.clone()).or_insert((0, 0));
        e.0 += 1;
        e.1 += s.cents;
    }
    acc
}

/// T4 - stable output: regions sorted by name, then the summary lines.
pub fn render(totals: &BTreeMap<String, (u64, i64)>, rows: usize, rejected: usize) -> Vec<String> {
    let mut out: Vec<String> = totals
        .iter()
        .map(|(r, (c, t))| format!("region {r} count={c} total={}", format_cents(*t)))
        .collect();
    let valid: u64 = totals.values().map(|(c, _)| c).sum();
    let grand: i64 = totals.values().map(|(_, t)| t).sum();
    out.push(format!("rows={rows} valid={valid} rejected={rejected}"));
    out.push(format!("grand_total={}", format_cents(grand)));
    out
}
```

### Go — `cd go/salesrep && go build -o salesrep . && ./salesrep ../../sales.csv`

#### `money/go.mod`

```go
module example.com/money

go 1.24
```

#### `money/money.go`

```go
// Package money parses and formats amounts as integer cents (version 1.3.0).
package money

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
)

// ErrInvalid is returned for anything but N, N.D or N.DD.
var ErrInvalid = errors.New("invalid amount")

// ParseCents is T0: it converts "12.5" into 1250.
func ParseCents(text string) (int64, error) {
	whole, frac, hasDot := strings.Cut(text, ".")
	if !digitsOnly(whole) || (hasDot && (len(frac) < 1 || len(frac) > 2 || !digitsOnly(frac))) {
		return 0, ErrInvalid
	}
	w, err := strconv.ParseInt(whole, 10, 64)
	if err != nil {
		return 0, fmt.Errorf("amount %q: %w", text, err)
	}
	f := int64(0)
	if hasDot {
		f, _ = strconv.ParseInt((frac + "0")[:2], 10, 64)
	}
	return w*100 + f, nil
}

// FormatCents converts 1250 into "12.50".
func FormatCents(cents int64) string {
	return fmt.Sprintf("%d.%02d", cents/100, cents%100)
}

// digitsOnly starts with a lowercase letter: it is private to the package.
func digitsOnly(s string) bool {
	if s == "" {
		return false
	}
	for _, r := range s {
		if r < '0' || r > '9' {
			return false
		}
	}
	return true
}
```

#### `salesrep/go.mod`

```go
module example.com/salesrep

go 1.24

require example.com/money v1.3.0

// no proxy needed: the module lives next to this one
replace example.com/money => ../money
```

#### `salesrep/main.go`

```go
package main

import (
	"fmt"
	"os"
	"strings"

	"example.com/salesrep/internal/parsing"
	"example.com/salesrep/internal/report"
)

func main() { os.Exit(run(os.Args)) }

func run(args []string) int {
	if len(args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: salesrep sales.csv")
		return 2
	}
	data, err := os.ReadFile(args[1]) // T1
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: cannot read %s\n", args[1])
		return 2
	}
	text := strings.ReplaceAll(string(data), "\r\n", "\n")
	lines := strings.Split(strings.TrimSuffix(text, "\n"), "\n")
	sales, rejects := parsing.ParseSales(lines)
	for _, r := range rejects {
		fmt.Fprintf(os.Stderr, "reject line %d: %s\n", r.Line, r.Reason)
	}
	rows := max(len(lines)-1, 0)
	fmt.Println(strings.Join(report.Render(report.TotalsByRegion(sales), rows, len(rejects)), "\n"))
	if len(rejects) > 0 { // T5
		return 1
	}
	return 0
}
```

#### `salesrep/internal/parsing/parsing.go`

```go
// Package parsing turns CSV lines into validated sales. internal/ makes it
// importable only from inside example.com/salesrep.
package parsing

import (
	"strconv"
	"strings"

	"example.com/money"
)

type Sale struct {
	ID     int
	Region string
	Cents  int64
}

type Reject struct {
	Line   int
	Reason string
}

// ParseSales is T2: validate every line, collect rejects with their reason.
func ParseSales(lines []string) ([]Sale, []Reject) {
	var sales []Sale
	var rejects []Reject
	seen := map[int]bool{}
	for i, raw := range lines {
		no := i + 1
		line := strings.TrimSpace(raw)
		if no == 1 || line == "" {
			continue // header and blank lines
		}
		parts := strings.Split(line, ",")
		if len(parts) != 3 {
			rejects = append(rejects, Reject{no, "fields"})
			continue
		}
		sid, region, amount := strings.TrimSpace(parts[0]), strings.TrimSpace(parts[1]), strings.TrimSpace(parts[2])
		id, err := strconv.Atoi(sid)
		switch {
		case err != nil || id <= 0 || strings.ContainsAny(sid, "+-"):
			rejects = append(rejects, Reject{no, "id"})
		case seen[id]:
			rejects = append(rejects, Reject{no, "duplicate"})
		case region == "":
			rejects = append(rejects, Reject{no, "region"})
		default:
			cents, err := money.ParseCents(amount)
			if err != nil {
				rejects = append(rejects, Reject{no, "amount"})
				continue
			}
			seen[id] = true
			sales = append(sales, Sale{id, region, cents})
		}
	}
	return sales, rejects
}
```

#### `salesrep/internal/report/report.go`

```go
// Package report aggregates sales by region and renders the report.
package report

import (
	"fmt"
	"sort"

	"example.com/money"
	"example.com/salesrep/internal/parsing"
)

type Total struct {
	Count int
	Cents int64
}

// TotalsByRegion is T3.
func TotalsByRegion(sales []parsing.Sale) map[string]Total {
	acc := map[string]Total{}
	for _, s := range sales {
		t := acc[s.Region]
		t.Count++
		t.Cents += s.Cents
		acc[s.Region] = t
	}
	return acc
}

// Render is T4: regions sorted by name, then the summary lines.
func Render(totals map[string]Total, rows, rejected int) []string {
	names := make([]string, 0, len(totals))
	for r := range totals {
		names = append(names, r)
	}
	sort.Strings(names)
	var out []string
	valid, grand := 0, int64(0)
	for _, r := range names {
		t := totals[r]
		out = append(out, fmt.Sprintf("region %s count=%d total=%s", r, t.Count, money.FormatCents(t.Cents)))
		valid += t.Count
		grand += t.Cents
	}
	out = append(out, fmt.Sprintf("rows=%d valid=%d rejected=%d", rows, valid, rejected))
	out = append(out, "grand_total="+money.FormatCents(grand))
	return out
}
```
