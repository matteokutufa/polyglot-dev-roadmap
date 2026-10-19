## Appendice C02.A — Programmi completi dello scontrino

Salvare nella stessa cartella di `receipt.csv`. Exit code: `0` elaborazione completata, `1` file non leggibile o riga non valida, `2` argomento mancante. Compilazione di Rust: `rustc --edition 2024 -O receipt.rs`.

#### Python (receipt.py) — `python3 receipt.py receipt.csv`

```python
import sys
from dataclasses import dataclass

VAT_PERCENT = 22  # prices already include VAT


@dataclass(frozen=True)
class Line:
    name: str
    qty: int
    price_cents: int
    discount: int | None  # percent, or None when the field is empty


def all_digits(s: str) -> bool:
    return s != "" and s.isascii() and s.isdigit()


def parse_price(text: str) -> int:
    """'1.20' -> 120, '2.5' -> 250, '3' -> 300. No floats: money stays an integer."""
    whole, dot, frac = text.partition(".")
    if not all_digits(whole) or (dot != "" and not (all_digits(frac) and len(frac) <= 2)):
        raise ValueError(f'price "{text}" is not a decimal with at most 2 digits')
    return int(whole) * 100 + int(frac.ljust(2, "0"))


def parse_line(raw: str) -> Line:
    fields = raw.split(";")
    if len(fields) != 4:
        raise ValueError(f"expected 4 fields, got {len(fields)}")
    name, qty_text, price_text, discount_text = fields
    if name.strip() == "":
        raise ValueError("name is empty")
    if not all_digits(qty_text) or int(qty_text) == 0:
        raise ValueError(f'quantity "{qty_text}" is not a positive integer')
    discount = None  # absence of a value
    if discount_text != "":
        if not all_digits(discount_text) or int(discount_text) > 100:
            raise ValueError(f'discount "{discount_text}" is not an integer between 0 and 100')
        discount = int(discount_text)
    return Line(name.strip(), int(qty_text), parse_price(price_text), discount)


def line_total(line: Line) -> int:
    gross = line.qty * line.price_cents
    off = 0 if line.discount is None else (gross * line.discount + 50) // 100  # round half up
    return gross - off


def money(cents: int) -> str:
    return f"{cents // 100}.{cents % 100:02d}"


def main() -> int:
    if len(sys.argv) != 2:
        print("usage: receipt <file>", file=sys.stderr)
        return 2
    try:
        with open(sys.argv[1], encoding="utf-8") as f:
            raw_lines = f.read().splitlines()
    except OSError:
        print(f"fatal: cannot read {sys.argv[1]}", file=sys.stderr)
        return 1

    total = 0
    for n, raw in enumerate(raw_lines, start=1):
        if raw.strip() == "":
            continue
        try:
            line = parse_line(raw)
        except ValueError as e:
            print(f"line {n}: {e}", file=sys.stderr)
            return 1
        amount = line_total(line)
        total += amount
        suffix = "" if line.discount is None else f" (-{line.discount}%)"
        print(f"{line.name:<20} {line.qty:>3} x {money(line.price_cents):>6} = {money(amount):>7}{suffix}")

    vat = (total * VAT_PERCENT + (100 + VAT_PERCENT) // 2) // (100 + VAT_PERCENT)
    print("-" * 43)
    print(f"{'TOTAL':<20}{money(total):>23}")
    print(f"{'of which VAT ' + str(VAT_PERCENT) + '%':<20}{money(vat):>23}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

#### PHP (receipt.php) — `php receipt.php receipt.csv`

```php
<?php
declare(strict_types=1);

const VAT_PERCENT = 22; // prices already include VAT

final class Line
{
    public function __construct(
        public readonly string $name,
        public readonly int $qty,
        public readonly int $priceCents,
        public readonly ?int $discount, // percent, or null when the field is empty
    ) {}
}

function allDigits(string $s): bool
{
    return $s !== '' && ctype_digit($s); // ctype_digit: ASCII digits only
}

/** '1.20' -> 120, '2.5' -> 250, '3' -> 300. No floats: money stays an integer. */
function parsePrice(string $text): int
{
    [$whole, $frac] = array_pad(explode('.', $text, 2), 2, null);
    $hasDot = $frac !== null;
    if (!allDigits($whole) || ($hasDot && !(allDigits($frac) && strlen($frac) <= 2))) {
        throw new InvalidArgumentException("price \"$text\" is not a decimal with at most 2 digits");
    }
    return (int) $whole * 100 + (int) str_pad($frac ?? '', 2, '0');
}

function parseLine(string $raw): Line
{
    $fields = explode(';', $raw);
    if (count($fields) !== 4) {
        throw new InvalidArgumentException('expected 4 fields, got ' . count($fields));
    }
    [$name, $qtyText, $priceText, $discountText] = $fields;
    if (trim($name) === '') {
        throw new InvalidArgumentException('name is empty');
    }
    if (!allDigits($qtyText) || (int) $qtyText === 0) {
        throw new InvalidArgumentException("quantity \"$qtyText\" is not a positive integer");
    }
    $discount = null; // absence of a value
    if ($discountText !== '') {
        if (!allDigits($discountText) || (int) $discountText > 100) {
            throw new InvalidArgumentException("discount \"$discountText\" is not an integer between 0 and 100");
        }
        $discount = (int) $discountText;
    }
    return new Line(trim($name), (int) $qtyText, parsePrice($priceText), $discount);
}

function lineTotal(Line $line): int
{
    $gross = $line->qty * $line->priceCents;
    $off = $line->discount === null ? 0 : intdiv($gross * $line->discount + 50, 100); // round half up
    return $gross - $off;
}

function money(int $cents): string
{
    return intdiv($cents, 100) . '.' . str_pad((string) ($cents % 100), 2, '0', STR_PAD_LEFT);
}

/** printf("%-20s") counts bytes, so accented names would misalign: pad by characters. */
function padRight(string $s, int $width): string
{
    return $s . str_repeat(' ', max(0, $width - mb_strlen($s)));
}

function padLeft(string $s, int $width): string
{
    return str_repeat(' ', max(0, $width - mb_strlen($s))) . $s;
}

function main(array $argv): int
{
    if (count($argv) !== 2) {
        fwrite(STDERR, "usage: receipt <file>\n");
        return 2;
    }
    $content = @file_get_contents($argv[1]); // false on failure (legacy convention, see C09)
    if ($content === false) {
        fwrite(STDERR, "fatal: cannot read $argv[1]\n");
        return 1;
    }

    $total = 0;
    foreach (preg_split('/\r?\n/', rtrim($content, "\r\n")) as $i => $raw) {
        $n = $i + 1;
        if (trim($raw) === '') {
            continue;
        }
        try {
            $line = parseLine($raw);
        } catch (InvalidArgumentException $e) {
            fwrite(STDERR, "line $n: " . $e->getMessage() . "\n");
            return 1;
        }
        $amount = lineTotal($line);
        $total += $amount;
        $suffix = $line->discount === null ? '' : " (-{$line->discount}%)";
        echo padRight($line->name, 20), ' ', padLeft((string) $line->qty, 3), ' x ',
            padLeft(money($line->priceCents), 6), ' = ', padLeft(money($amount), 7), $suffix, "\n";
    }

    $vat = intdiv($total * VAT_PERCENT + intdiv(100 + VAT_PERCENT, 2), 100 + VAT_PERCENT);
    echo str_repeat('-', 43), "\n";
    echo padRight('TOTAL', 20), padLeft(money($total), 23), "\n";
    echo padRight('of which VAT ' . VAT_PERCENT . '%', 20), padLeft(money($vat), 23), "\n";
    return 0;
}

exit(main($argv));
```

#### Rust (receipt.rs) — `rustc --edition 2024 -O receipt.rs && ./receipt receipt.csv`

```rust
use std::env;
use std::fs;
use std::process::ExitCode;

const VAT_PERCENT: u64 = 22; // prices already include VAT

struct Line {
    name: String,
    qty: u64,
    price_cents: u64,
    discount: Option<u64>, // percent, or None when the field is empty
}

fn all_digits(s: &str) -> bool {
    !s.is_empty() && s.chars().all(|c| c.is_ascii_digit())
}

/// "1.20" -> 120, "2.5" -> 250, "3" -> 300. No floats: money stays an integer.
fn parse_price(text: &str) -> Result<u64, String> {
    let (whole, frac) = match text.split_once('.') {
        Some((w, f)) => (w, Some(f)),
        None => (text, None),
    };
    let frac_ok = match frac {
        Some(f) => all_digits(f) && f.len() <= 2,
        None => true,
    };
    if !all_digits(whole) || !frac_ok {
        return Err(format!("price \"{text}\" is not a decimal with at most 2 digits"));
    }
    let whole: u64 = whole.parse().map_err(|_| format!("price \"{text}\" is too large"))?;
    let cents: u64 = match frac {
        Some(f) => format!("{f:0<2}").parse().unwrap_or(0), // pad on the right: "5" -> "50"
        None => 0,
    };
    Ok(whole * 100 + cents)
}

fn parse_line(raw: &str) -> Result<Line, String> {
    let fields: Vec<&str> = raw.split(';').collect();
    if fields.len() != 4 {
        return Err(format!("expected 4 fields, got {}", fields.len()));
    }
    let (name, qty_text, price_text, discount_text) = (fields[0], fields[1], fields[2], fields[3]);
    if name.trim().is_empty() {
        return Err("name is empty".to_string());
    }
    let qty: u64 = if all_digits(qty_text) { qty_text.parse().unwrap_or(0) } else { 0 };
    if qty == 0 {
        return Err(format!("quantity \"{qty_text}\" is not a positive integer"));
    }
    let discount = if discount_text.is_empty() {
        None // absence of a value
    } else {
        match discount_text.parse::<u64>() {
            Ok(d) if all_digits(discount_text) && d <= 100 => Some(d),
            _ => {
                return Err(format!("discount \"{discount_text}\" is not an integer between 0 and 100"));
            }
        }
    };
    Ok(Line { name: name.trim().to_string(), qty, price_cents: parse_price(price_text)?, discount })
}

fn line_total(line: &Line) -> u64 {
    let gross = line.qty * line.price_cents;
    let off = match line.discount {
        None => 0,
        Some(pct) => (gross * pct + 50) / 100, // round half up
    };
    gross - off
}

fn money(cents: u64) -> String {
    format!("{}.{:02}", cents / 100, cents % 100)
}

fn main() -> ExitCode {
    let args: Vec<String> = env::args().collect();
    if args.len() != 2 {
        eprintln!("usage: receipt <file>");
        return ExitCode::from(2);
    }
    let content = match fs::read_to_string(&args[1]) {
        Ok(text) => text,
        Err(_) => {
            eprintln!("fatal: cannot read {}", args[1]);
            return ExitCode::from(1);
        }
    };

    let mut total: u64 = 0;
    for (i, raw) in content.lines().enumerate() {
        let n = i + 1;
        if raw.trim().is_empty() {
            continue;
        }
        let line = match parse_line(raw) {
            Ok(l) => l,
            Err(msg) => {
                eprintln!("line {n}: {msg}");
                return ExitCode::from(1);
            }
        };
        let amount = line_total(&line);
        total += amount;
        let suffix = match line.discount {
            None => String::new(),
            Some(pct) => format!(" (-{pct}%)"),
        };
        // {:<20} pads by characters (Unicode scalar values), not by bytes
        println!(
            "{:<20} {:>3} x {:>6} = {:>7}{}",
            line.name,
            line.qty,
            money(line.price_cents),
            money(amount),
            suffix
        );
    }

    let vat = (total * VAT_PERCENT + (100 + VAT_PERCENT) / 2) / (100 + VAT_PERCENT);
    println!("{}", "-".repeat(43));
    println!("{:<20}{:>23}", "TOTAL", money(total));
    println!("{:<20}{:>23}", format!("of which VAT {VAT_PERCENT}%"), money(vat));
    ExitCode::SUCCESS
}
```

#### Go (main.go, go.mod con `module example.com/receipt`) — `go run . receipt.csv`

```go
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

const vatPercent = 22 // prices already include VAT

type Line struct {
	Name       string
	Qty        int
	PriceCents int
	Discount   *int // percent; nil when the field is empty
}

func allDigits(s string) bool {
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

// parsePrice: "1.20" -> 120, "2.5" -> 250, "3" -> 300. No floats: money stays an integer.
func parsePrice(text string) (int, error) {
	whole, frac, hasDot := strings.Cut(text, ".")
	if !allDigits(whole) || (hasDot && !(allDigits(frac) && len(frac) <= 2)) {
		return 0, fmt.Errorf("price \"%s\" is not a decimal with at most 2 digits", text)
	}
	w, err := strconv.Atoi(whole)
	if err != nil {
		return 0, fmt.Errorf("price \"%s\" is too large", text)
	}
	c := 0
	if hasDot {
		c, _ = strconv.Atoi(frac + strings.Repeat("0", 2-len(frac))) // "5" -> "50"
	}
	return w*100 + c, nil
}

func parseLine(raw string) (Line, error) {
	fields := strings.Split(raw, ";")
	if len(fields) != 4 {
		return Line{}, fmt.Errorf("expected 4 fields, got %d", len(fields))
	}
	name, qtyText, priceText, discountText := fields[0], fields[1], fields[2], fields[3]
	if strings.TrimSpace(name) == "" {
		return Line{}, fmt.Errorf("name is empty")
	}
	qty := 0
	if allDigits(qtyText) {
		qty, _ = strconv.Atoi(qtyText)
	}
	if qty == 0 {
		return Line{}, fmt.Errorf("quantity \"%s\" is not a positive integer", qtyText)
	}
	var discount *int // nil = absence of a value
	if discountText != "" {
		d, err := strconv.Atoi(discountText)
		if !allDigits(discountText) || err != nil || d > 100 {
			return Line{}, fmt.Errorf("discount \"%s\" is not an integer between 0 and 100", discountText)
		}
		discount = &d
	}
	price, err := parsePrice(priceText)
	if err != nil {
		return Line{}, err
	}
	return Line{strings.TrimSpace(name), qty, price, discount}, nil
}

func lineTotal(l Line) int {
	gross := l.Qty * l.PriceCents
	off := 0
	if l.Discount != nil {
		off = (gross**l.Discount + 50) / 100 // round half up
	}
	return gross - off
}

func money(cents int) string {
	return fmt.Sprintf("%d.%02d", cents/100, cents%100)
}

func main() { os.Exit(run(os.Args)) }

func run(args []string) int {
	if len(args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: receipt <file>")
		return 2
	}
	data, err := os.ReadFile(args[1])
	if err != nil {
		fmt.Fprintf(os.Stderr, "fatal: cannot read %s\n", args[1])
		return 1
	}

	total := 0
	for i, raw := range strings.Split(strings.TrimRight(string(data), "\r\n"), "\n") {
		n := i + 1
		raw = strings.TrimSuffix(raw, "\r") // files written on Windows end lines with \r\n
		if strings.TrimSpace(raw) == "" {
			continue
		}
		line, err := parseLine(raw)
		if err != nil {
			fmt.Fprintf(os.Stderr, "line %d: %v\n", n, err)
			return 1
		}
		amount := lineTotal(line)
		total += amount
		suffix := ""
		if line.Discount != nil {
			suffix = fmt.Sprintf(" (-%d%%)", *line.Discount)
		}
		// %-20s pads by runes (Unicode code points), not by bytes
		fmt.Printf("%-20s %3d x %6s = %7s%s\n", line.Name, line.Qty, money(line.PriceCents), money(amount), suffix)
	}

	vat := (total*vatPercent + (100+vatPercent)/2) / (100 + vatPercent)
	fmt.Println(strings.Repeat("-", 43))
	fmt.Printf("%-20s%23s\n", "TOTAL", money(total))
	fmt.Printf("%-20s%23s\n", fmt.Sprintf("of which VAT %d%%", vatPercent), money(vat))
	return 0
}
```
