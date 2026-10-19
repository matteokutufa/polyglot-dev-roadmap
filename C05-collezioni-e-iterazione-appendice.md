# C05 — Appendice: i quattro programmi completi

Eseguibili con: `python3 main.py orders.csv`, `php main.php orders.csv`, `rustc --edition 2024 main.rs -o main_rs && ./main_rs orders.csv`, `go run main.go orders.csv` (con `go.mod` che dichiara `go 1.24`). Tutti usano soltanto la libreria standard.

#### Python (`main.py`) — `python3 main.py orders.csv`

```python
import sys

CATALOG = ("A1", "B2", "C3", "D4", "E5", "F6")


def read_rows(path):
    """T1 - lazy: yield (line_number, text) one row at a time, header skipped."""
    with open(path, encoding="utf-8") as f:
        next(f, None)
        for n, line in enumerate(f, start=2):
            line = line.strip()
            if line:
                yield n, line


def parse(line):
    """T2 - return (record, "") or (None, reason)."""
    parts = [p.strip() for p in line.split(",")]
    if len(parts) != 4:
        return None, "fields"
    oid, customer, sku, qty = parts
    if not oid or not customer:
        return None, "empty"
    if sku not in CATALOG:
        return None, "sku"
    if not (qty.isascii() and qty.isdigit()) or int(qty) == 0:
        return None, "qty"
    return (oid, customer, sku, int(qty)), ""


def run(path):
    seen = set()           # T3 - order ids already counted
    qty_by_customer = {}   # T4 - customer -> total pieces
    used_skus = set()      # T4 - distinct SKUs ordered
    rows = valid = dup = rejected = 0
    for n, line in read_rows(path):
        rows += 1
        rec, reason = parse(line)
        if rec is None:
            rejected += 1
            print(f"reject line={n} reason={reason}", file=sys.stderr)
            continue
        oid, customer, sku, qty = rec
        if oid in seen:
            dup += 1
            continue
        seen.add(oid)
        valid += 1
        qty_by_customer[customer] = qty_by_customer.get(customer, 0) + qty
        used_skus.add(sku)
    # T5 - top 3: more pieces first, then alphabetical
    top = sorted(qty_by_customer.items(), key=lambda kv: (-kv[1], kv[0]))[:3]
    # T6 - catalog minus used, sorted
    unused = sorted(set(CATALOG) - used_skus)
    print(f"rows={rows} valid={valid} duplicates={dup} rejected={rejected}")
    print(f"customers={len(qty_by_customer)} skus={len(used_skus)}")
    print("top=" + ",".join(f"{c}:{q}" for c, q in top))
    print("unused_skus=" + ",".join(unused))
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

#### PHP (`main.php`) — `php main.php orders.csv`

```php
<?php
declare(strict_types=1);

const CATALOG = ['A1', 'B2', 'C3', 'D4', 'E5', 'F6'];

/** T1 - lazy: yield line_number => text one row at a time, header skipped. */
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

/** T2 - return [record, ''] or [null, reason]. */
function parse(string $line): array
{
    $parts = array_map('trim', explode(',', $line));
    if (count($parts) !== 4) {
        return [null, 'fields'];
    }
    [$oid, $customer, $sku, $qty] = $parts;
    if ($oid === '' || $customer === '') {
        return [null, 'empty'];
    }
    if (!in_array($sku, CATALOG, true)) {
        return [null, 'sku'];
    }
    if (!ctype_digit($qty) || (int)$qty === 0) {
        return [null, 'qty'];
    }
    return [[$oid, $customer, $sku, (int)$qty], ''];
}

function run(string $path): int
{
    $seen = [];          // T3 - order ids already counted (keys used as a set)
    $qtyByCustomer = []; // T4 - customer => total pieces
    $usedSkus = [];      // T4 - distinct SKUs ordered (keys used as a set)
    $rows = $valid = $dup = $rejected = 0;
    foreach (readRows($path) as $n => $line) {
        $rows++;
        [$rec, $reason] = parse($line);
        if ($rec === null) {
            $rejected++;
            fwrite(STDERR, "reject line=$n reason=$reason\n");
            continue;
        }
        [$oid, $customer, $sku, $qty] = $rec;
        if (isset($seen[$oid])) {
            $dup++;
            continue;
        }
        $seen[$oid] = true;
        $valid++;
        $qtyByCustomer[$customer] = ($qtyByCustomer[$customer] ?? 0) + $qty;
        $usedSkus[$sku] = true;
    }
    // T5 - top 3: more pieces first, then alphabetical
    $items = [];
    foreach ($qtyByCustomer as $c => $q) {
        $items[] = [(string)$c, $q];
    }
    usort($items, fn(array $a, array $b) => [$b[1], $a[0]] <=> [$a[1], $b[0]]);
    $top = array_slice($items, 0, 3);
    // T6 - catalog minus used, sorted
    $unused = array_diff(CATALOG, array_keys($usedSkus));
    sort($unused);
    echo "rows=$rows valid=$valid duplicates=$dup rejected=$rejected\n";
    echo 'customers=' . count($qtyByCustomer) . ' skus=' . count($usedSkus) . "\n";
    echo 'top=' . implode(',', array_map(fn(array $t) => "$t[0]:$t[1]", $top)) . "\n";
    echo 'unused_skus=' . implode(',', $unused) . "\n";
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

#### Rust (`main.rs`) — `rustc --edition 2024 main.rs -o main_rs && ./main_rs orders.csv`

```rust
use std::collections::{HashMap, HashSet};
use std::fs::File;
use std::io::{BufRead, BufReader};
use std::process::ExitCode;

const CATALOG: [&str; 6] = ["A1", "B2", "C3", "D4", "E5", "F6"];

struct Order {
    id: String,
    customer: String,
    sku: String,
    qty: u32,
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

/// T2 - Ok(order) or Err(reason).
fn parse(line: &str) -> Result<Order, &'static str> {
    let p: Vec<&str> = line.split(',').map(str::trim).collect();
    if p.len() != 4 {
        return Err("fields");
    }
    if p[0].is_empty() || p[1].is_empty() {
        return Err("empty");
    }
    if !CATALOG.contains(&p[2]) {
        return Err("sku");
    }
    if p[3].is_empty() || !p[3].bytes().all(|b| b.is_ascii_digit()) {
        return Err("qty");
    }
    let qty: u32 = p[3].parse().map_err(|_| "qty")?;
    if qty == 0 {
        return Err("qty");
    }
    Ok(Order { id: p[0].into(), customer: p[1].into(), sku: p[2].into(), qty })
}

fn run(path: &str) -> std::io::Result<u8> {
    let mut seen: HashSet<String> = HashSet::new(); // T3 - order ids already counted
    let mut qty_by_customer: HashMap<String, u32> = HashMap::new(); // T4
    let mut used_skus: HashSet<String> = HashSet::new(); // T4
    let (mut rows, mut valid, mut dup, mut rejected) = (0, 0, 0, 0);
    for (n, line) in read_rows(path)? {
        rows += 1;
        let o = match parse(&line) {
            Ok(o) => o,
            Err(reason) => {
                rejected += 1;
                eprintln!("reject line={n} reason={reason}");
                continue;
            }
        };
        if !seen.insert(o.id) {
            dup += 1;
            continue;
        }
        valid += 1;
        *qty_by_customer.entry(o.customer).or_insert(0) += o.qty;
        used_skus.insert(o.sku);
    }
    // T5 - top 3: more pieces first, then alphabetical
    let mut items: Vec<(&String, &u32)> = qty_by_customer.iter().collect();
    items.sort_by(|a, b| b.1.cmp(a.1).then(a.0.cmp(b.0)));
    let top: Vec<String> = items.iter().take(3).map(|(c, q)| format!("{c}:{q}")).collect();
    // T6 - catalog minus used, sorted
    let mut unused: Vec<&str> = CATALOG.iter().copied().filter(|s| !used_skus.contains(*s)).collect();
    unused.sort();
    println!("rows={rows} valid={valid} duplicates={dup} rejected={rejected}");
    println!("customers={} skus={}", qty_by_customer.len(), used_skus.len());
    println!("top={}", top.join(","));
    println!("unused_skus={}", unused.join(","));
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

#### Go (`main.go`) — `go run main.go orders.csv`

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

var catalog = []string{"A1", "B2", "C3", "D4", "E5", "F6"}

type order struct {
	id, customer, sku string
	qty               int
}

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

// parse is T2 - returns the order or a rejection reason.
func parse(line string) (order, string) {
	p := strings.Split(line, ",")
	for i := range p {
		p[i] = strings.TrimSpace(p[i])
	}
	if len(p) != 4 {
		return order{}, "fields"
	}
	if p[0] == "" || p[1] == "" {
		return order{}, "empty"
	}
	known := false
	for _, s := range catalog {
		known = known || s == p[2]
	}
	if !known {
		return order{}, "sku"
	}
	if p[3] == "" || strings.Trim(p[3], "0123456789") != "" {
		return order{}, "qty"
	}
	qty, err := strconv.Atoi(p[3])
	if err != nil || qty == 0 {
		return order{}, "qty"
	}
	return order{p[0], p[1], p[2], qty}, ""
}

func run(path string) int {
	rows, err := readRows(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "cannot open %s\n", path)
		return 2
	}
	seen := map[string]struct{}{}     // T3 - order ids already counted
	qtyByCustomer := map[string]int{} // T4 - customer -> total pieces
	usedSkus := map[string]struct{}{} // T4 - distinct SKUs ordered
	var nRows, valid, dup, rejected int
	for n, line := range rows {
		nRows++
		o, reason := parse(line)
		if reason != "" {
			rejected++
			fmt.Fprintf(os.Stderr, "reject line=%d reason=%s\n", n, reason)
			continue
		}
		if _, ok := seen[o.id]; ok {
			dup++
			continue
		}
		seen[o.id] = struct{}{}
		valid++
		qtyByCustomer[o.customer] += o.qty
		usedSkus[o.sku] = struct{}{}
	}
	// T5 - top 3: more pieces first, then alphabetical
	type kv struct {
		name string
		qty  int
	}
	items := make([]kv, 0, len(qtyByCustomer))
	for c, q := range qtyByCustomer {
		items = append(items, kv{c, q})
	}
	sort.Slice(items, func(i, j int) bool {
		if items[i].qty != items[j].qty {
			return items[i].qty > items[j].qty
		}
		return items[i].name < items[j].name
	})
	if len(items) > 3 {
		items = items[:3]
	}
	top := make([]string, 0, 3)
	for _, it := range items {
		top = append(top, fmt.Sprintf("%s:%d", it.name, it.qty))
	}
	// T6 - catalog minus used, sorted
	var unused []string
	for _, s := range catalog {
		if _, ok := usedSkus[s]; !ok {
			unused = append(unused, s)
		}
	}
	sort.Strings(unused)
	fmt.Printf("rows=%d valid=%d duplicates=%d rejected=%d\n", nRows, valid, dup, rejected)
	fmt.Printf("customers=%d skus=%d\n", len(qtyByCustomer), len(usedSkus))
	fmt.Println("top=" + strings.Join(top, ","))
	fmt.Println("unused_skus=" + strings.Join(unused, ","))
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

**Limiti dichiarati di questa bozza:** le misure di e7 sono state fatte in un ambiente condiviso; il comportamento di PHP sui decimali come chiave (e4) non è stato verificato con `error_reporting(E_ALL)`; la crescita delle slice Go oltre 256 elementi è descritta, non misurata; PHPStan/mypy non sono stati eseguiti su questo capitolo.
