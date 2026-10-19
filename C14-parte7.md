> File C14-parte7.md (parte 7 di 8) — C14 I/O, serializzazione, tempo; corso di programmazione multi-linguaggio

### PHP — `cd php && php main.php ../readings.ndjson`

#### `src/core.php`

```php
<?php
declare(strict_types=1);

namespace Coldchain;

use DateTimeImmutable;
use DateTimeZone;

const GAP_SECONDS = 30 * 60;      // T1 - silence longer than this is reported
const TEMP_MIN = -500;            // tenths of a degree Celsius: -50.0 ..
const TEMP_MAX = 600;             // .. 60.0
const BOM = "\xEF\xBB\xBF";       // UTF-8 byte order mark
const JSON_WS = " \t\r\n";        // the only whitespace JSON knows

function zone(): DateTimeZone      // the warehouse's civil time
{
    static $tz = null;
    return $tz ??= new DateTimeZone('Europe/Rome');
}

/** Why a line was rejected: a closed set, not free text. */
enum Reason: string
{
    case Encoding = 'encoding';    // not valid UTF-8
    case Json = 'json';            // not valid JSON
    case Field = 'field';          // valid JSON, wrong shape or type
    case Time = 'time';            // timestamp malformed or impossible
    case Range = 'range';          // temperature out of range or too precise
    case Duplicate = 'duplicate';  // same sensor and instant already seen
}

final class Reading
{
    public function __construct(
        public readonly string $sensor,
        public readonly int $at,      // UTC instant, seconds since 1970-01-01T00:00:00Z
        public readonly int $tenths,  // temperature in tenths of a degree Celsius
    ) {}
}

/** T2 - bytes -> text -> JSON value, or the reason why not. */
function decode(string $raw): mixed
{
    if (!mb_check_encoding($raw, 'UTF-8')) {
        return Reason::Encoding;
    }
    try {
        return json_decode($raw, false, 512, JSON_THROW_ON_ERROR);
    } catch (\JsonException) {
        return Reason::Json;
    }
}

/** T4 - strict RFC 3339 (no fraction) -> UTC epoch seconds, or null. */
function parseInstant(string $text): ?int
{
    $re = '/\A([0-9]{4})-([0-9]{2})-([0-9]{2})T([0-9]{2}):([0-9]{2}):([0-9]{2})(?:Z|[+-]([0-9]{2}):([0-9]{2}))\z/';
    if (preg_match($re, $text, $m) !== 1) {
        return null;
    }
    [$y, $mo, $d, $hh, $mi, $ss] = array_map('intval', array_slice($m, 1, 6));
    $oh = (int) ($m[7] ?? 0);
    $om = (int) ($m[8] ?? 0);
    if ($y < 1970 || $y > 2100 || $hh > 23 || $mi > 59 || $ss > 59 || $oh > 14 || $om > 59) {
        return null;
    }
    if (!checkdate($mo, $d, $y)) {  // new DateTimeImmutable('2026-02-30') would roll over to March
        return null;
    }
    return (new DateTimeImmutable($text))->getTimestamp();
}

/** A temperature in degrees -> tenths, only if in range with at most one decimal. */
function toTenths(int|float $x): ?int
{
    $scaled = $x * 10;
    if (!(TEMP_MIN <= $scaled && $scaled <= TEMP_MAX)) {  // false for NAN too
        return null;
    }
    $n = (int) round($scaled);
    return abs($scaled - $n) < 1e-6 ? $n : null;
}

/** T3 - a decoded JSON value -> Reading, or the first rule it breaks. */
function validate(mixed $value): Reading|Reason
{
    if (!($value instanceof \stdClass)) {
        return Reason::Field;
    }
    $o = (array) $value;
    $sensor = $o['sensor'] ?? null;
    $ts = $o['ts'] ?? null;
    $temp = $o['temp'] ?? null;
    if (!is_string($sensor) || preg_match('/\A[a-z0-9-]{1,32}\z/', $sensor) !== 1) {
        return Reason::Field;
    }
    if (!is_string($ts) || !(is_int($temp) || is_float($temp))) {
        return Reason::Field;
    }
    $at = parseInstant($ts);
    if ($at === null) {
        return Reason::Time;
    }
    $tenths = toTenths($temp);
    if ($tenths === null) {
        return Reason::Range;
    }
    return new Reading($sensor, $at, $tenths);
}

/** The local civil date (Y-m-d) of an instant. */
function localDate(int $at): string
{
    return (new DateTimeImmutable("@$at"))->setTimezone(zone())->format('Y-m-d');
}

/** T5 - length of a local day in hours: 23, 24 or 25. */
function dayHours(string $date): int
{
    $next = (new DateTimeImmutable($date . ' 00:00:00', new DateTimeZone('UTC')))->modify('+1 day')->format('Y-m-d');
    $start = (new DateTimeImmutable($date . ' 00:00:00', zone()))->getTimestamp();
    $end = (new DateTimeImmutable($next . ' 00:00:00', zone()))->getTimestamp();
    return intdiv($end - $start, 3600);
}

function localIso(int $at): string
{
    return (new DateTimeImmutable("@$at"))->setTimezone(zone())->format('c');
}

final class Report
{
    public int $readings = 0;
    /** @var list<Reading> */
    public array $valid = [];
    /** @var list<array{int,Reason}> */
    public array $rejects = [];
}

/** T6 - consume the stream one line at a time; blank lines are skipped. */
function run(iterable $lines): Report
{
    $rep = new Report();
    $seen = [];
    $n = 0;
    foreach ($lines as $raw) {
        $n++;
        if ($n === 1 && str_starts_with($raw, BOM)) {
            $raw = substr($raw, strlen(BOM));
        }
        if (trim($raw, JSON_WS) === '') {
            continue;
        }
        $rep->readings++;
        $value = decode($raw);
        $res = $value instanceof Reason ? $value : validate($value);
        if ($res instanceof Reason) {
            $rep->rejects[] = [$n, $res];
        } elseif (isset($seen[$res->sensor . "\0" . $res->at])) {
            $rep->rejects[] = [$n, Reason::Duplicate];
        } else {
            $seen[$res->sensor . "\0" . $res->at] = true;
            $rep->valid[] = $res;
        }
    }
    return $rep;
}

/** n / d rounded half away from zero (d > 0); integer division differs by language. */
function divRound(int $n, int $d): int
{
    $q = intdiv(2 * abs($n) + $d, 2 * $d);
    return $n < 0 ? -$q : $q;
}

/** @param list<Reading> $valid */
function daily(array $valid): array
{
    $groups = [];
    foreach ($valid as $r) {
        $groups[$r->sensor][localDate($r->at)][] = $r->tenths;
    }
    ksort($groups, SORT_STRING);
    $out = [];
    foreach ($groups as $sensor => $days) {
        ksort($days, SORT_STRING);
        foreach ($days as $date => $v) {
            $out[] = [
                'sensor' => (string) $sensor, 'date' => (string) $date, 'hours' => dayHours((string) $date),
                'count' => count($v), 'lo' => min($v), 'hi' => max($v),
                'avg_h' => divRound(array_sum($v) * 10, count($v)),
            ];
        }
    }
    return $out;
}

/** @param list<Reading> $valid */
function gaps(array $valid): array
{
    $by = [];
    foreach ($valid as $r) {
        $by[$r->sensor][] = $r->at;
    }
    ksort($by, SORT_STRING);
    $out = [];
    foreach ($by as $sensor => $times) {
        sort($times);
        for ($i = 1; $i < count($times); $i++) {
            $diff = $times[$i] - $times[$i - 1];
            if ($diff > GAP_SECONDS) {
                $out[] = [
                    'sensor' => (string) $sensor, 'from' => localIso($times[$i - 1]),
                    'to' => localIso($times[$i]), 'minutes' => intdiv($diff, 60),
                ];
            }
        }
    }
    return $out;
}

/** Integer in 10**-$decimals units -> decimal text, no floats involved. */
function fixed(int $n, int $decimals): string
{
    $p = 10 ** $decimals;
    return ($n < 0 ? '-' : '') . intdiv(abs($n), $p) . '.' . str_pad((string) (abs($n) % $p), $decimals, '0', STR_PAD_LEFT);
}

function renderText(Report $rep): string
{
    $out = [sprintf('readings=%d valid=%d rejected=%d', $rep->readings, count($rep->valid), count($rep->rejects))];
    foreach (daily($rep->valid) as $d) {
        $out[] = sprintf(
            'day sensor=%s date=%s hours=%d count=%d min=%s max=%s avg=%s',
            $d['sensor'], $d['date'], $d['hours'], $d['count'], fixed($d['lo'], 1), fixed($d['hi'], 1), fixed($d['avg_h'], 2),
        );
    }
    foreach (gaps($rep->valid) as $g) {
        $out[] = sprintf('gap sensor=%s from=%s to=%s minutes=%d', $g['sensor'], $g['from'], $g['to'], $g['minutes']);
    }
    return implode("\n", $out) . "\n";
}

function renderJson(Report $rep): string
{
    $doc = [
        'readings' => $rep->readings,
        'valid' => count($rep->valid),
        'rejected' => count($rep->rejects),
        'days' => array_map(fn($d) => [
            'sensor' => $d['sensor'], 'date' => $d['date'], 'hours' => $d['hours'], 'count' => $d['count'],
            'min_tenths' => $d['lo'], 'max_tenths' => $d['hi'], 'avg_hundredths' => $d['avg_h'],
        ], daily($rep->valid)),
        'gaps' => gaps($rep->valid),
        'rejects' => array_map(fn($r) => ['line' => $r[0], 'reason' => $r[1]->value], $rep->rejects),
    ];
    return json_encode($doc, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_THROW_ON_ERROR) . "\n";
}
```

#### `main.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . '/src/core.php';

use function Coldchain\{renderJson, renderText, run};

/** Yield the lines of a stream one at a time, newline included; false means a read error. */
function lines($fh): Generator
{
    while (($line = fgets($fh)) !== false) {
        yield $line;
    }
    if (!feof($fh)) {
        throw new RuntimeException('read error');
    }
}

/** T8 - CLI: stream the file (or stdin), report on stdout, rejects on stderr. */
function main(array $argv): int
{
    $args = array_values(array_filter(array_slice($argv, 1), fn($a) => $a !== '--json'));
    $asJson = count($args) !== count($argv) - 1;
    $path = $args[0] ?? '-';
    try {
        if ($path === '-') {
            $rep = run(lines(STDIN));
        } else {
            // a directory opens fine but cannot be read; compress.zlib:// is a stream wrapper
            $url = str_ends_with($path, '.gz') ? "compress.zlib://$path" : $path;
            $fh = is_file($path) ? @fopen($url, 'rb') : false;
            if ($fh === false) {
                throw new RuntimeException('open error');
            }
            $rep = run(lines($fh));
            fclose($fh);
        }
    } catch (RuntimeException) {
        fwrite(STDERR, "error: cannot read $path\n");
        return 2;
    }
    foreach ($rep->rejects as [$n, $reason]) {
        fwrite(STDERR, "line $n: rejected ({$reason->value})\n");
    }
    echo $asJson ? renderJson($rep) : renderText($rep);
    return count($rep->rejects) > 0 ? 1 : 0;
}

exit(main($argv));
```

### Rust — `cd rs && cargo run --release -- ../readings.ndjson`

#### `Cargo.toml`

```toml
[package]
name = "coldchain"
version = "0.1.0"
edition = "2024"

[dependencies]
chrono = "0.4"
chrono-tz = "0.10"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
flate2 = "1"
```

#### `src/lib.rs`

```rust
//! Cold-chain report: stream NDJSON readings, validate at the boundary, report per local day.
use std::collections::{BTreeMap, HashSet};
use std::io::{self, BufRead};

use chrono::{DateTime, NaiveDate, TimeZone};
use chrono_tz::Europe::Rome;
use serde::Serialize;
use serde_json::Value;

/// T1 - silence longer than this is reported (the zone is `chrono_tz::Europe::Rome`).
pub const GAP_SECONDS: i64 = 30 * 60;
/// Tenths of a degree Celsius: -50.0 .. 60.0.
pub const TEMP_MIN: i64 = -500;
pub const TEMP_MAX: i64 = 600;
/// UTF-8 byte order mark.
pub const BOM: &[u8] = b"\xEF\xBB\xBF";
/// The only whitespace JSON knows.
pub const JSON_WS: &[u8] = b" \t\r\n";

/// Why a line was rejected: a closed set, not free text.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Reason {
    Encoding,  // not valid UTF-8
    Json,      // not valid JSON
    Field,     // valid JSON, wrong shape or type
    Time,      // timestamp malformed or impossible
    Range,     // temperature out of range or too precise
    Duplicate, // same sensor and instant already seen
}

impl Reason {
    pub fn as_str(self) -> &'static str {
        match self {
            Reason::Encoding => "encoding",
            Reason::Json => "json",
            Reason::Field => "field",
            Reason::Time => "time",
            Reason::Range => "range",
            Reason::Duplicate => "duplicate",
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Reading {
    pub sensor: String,
    pub at: i64,     // UTC instant, seconds since 1970-01-01T00:00:00Z
    pub tenths: i64, // temperature in tenths of a degree Celsius
}

/// T2 - bytes -> text -> JSON value, or the reason why not.
pub fn decode(raw: &[u8]) -> Result<Value, Reason> {
    let text = std::str::from_utf8(raw).map_err(|_| Reason::Encoding)?;
    serde_json::from_str(text).map_err(|_| Reason::Json)
}

fn digits(b: &[u8], from: usize, to: usize) -> Option<u32> {
    let part = b.get(from..to)?;
    if !part.iter().all(u8::is_ascii_digit) {
        return None;
    }
    Some(part.iter().fold(0, |acc, d| acc * 10 + u32::from(d - b'0')))
}

/// T4 - strict RFC 3339 (no fraction) -> UTC epoch seconds, or None.
pub fn parse_instant(text: &str) -> Option<i64> {
    let b = text.as_bytes();
    let shape_ok = match b.len() {
        20 => b[19] == b'Z',
        25 => (b[19] == b'+' || b[19] == b'-') && b[22] == b':',
        _ => false,
    } && b[4] == b'-'
        && b[7] == b'-'
        && b[10] == b'T'
        && b[13] == b':'
        && b[16] == b':';
    if !shape_ok {
        return None;
    }
    let (y, hh, mi, ss) = (
        digits(b, 0, 4)?,
        digits(b, 11, 13)?,
        digits(b, 14, 16)?,
        digits(b, 17, 19)?,
    );
    digits(b, 5, 7)?;
    digits(b, 8, 10)?;
    let (oh, om) = if b.len() == 25 {
        (digits(b, 20, 22)?, digits(b, 23, 25)?)
    } else {
        (0, 0)
    };
    if !(1970..=2100).contains(&y) || hh > 23 || mi > 59 || ss > 59 || oh > 14 || om > 59 {
        return None;
    }
    // also checks the calendar (Feb 30); the leap second 23:59:60 is already excluded above
    DateTime::parse_from_rfc3339(text)
        .ok()
        .map(|dt| dt.timestamp())
}

fn valid_sensor(s: &str) -> bool {
    (1..=32).contains(&s.len())
        && s.bytes()
            .all(|c| c.is_ascii_lowercase() || c.is_ascii_digit() || c == b'-')
}

/// A temperature in degrees -> tenths, only if in range with at most one decimal.
pub fn to_tenths(x: f64) -> Option<i64> {
    let scaled = x * 10.0;
    if !(TEMP_MIN as f64 <= scaled && scaled <= TEMP_MAX as f64) {
        return None; // also true for NaN
    }
    let n = scaled.round();
    ((scaled - n).abs() < 1e-6).then_some(n as i64)
}

/// T3 - a decoded JSON value -> Reading, or the first rule it breaks.
pub fn validate(value: &Value) -> Result<Reading, Reason> {
    let o = value.as_object().ok_or(Reason::Field)?;
    let sensor = o
        .get("sensor")
        .and_then(Value::as_str)
        .filter(|s| valid_sensor(s))
        .ok_or(Reason::Field)?;
    let ts = o.get("ts").and_then(Value::as_str).ok_or(Reason::Field)?;
    let temp = o.get("temp").and_then(Value::as_f64).ok_or(Reason::Field)?; // None unless a JSON number
    let at = parse_instant(ts).ok_or(Reason::Time)?;
    let tenths = to_tenths(temp).ok_or(Reason::Range)?;
    Ok(Reading {
        sensor: sensor.to_string(),
        at,
        tenths,
    })
}

fn local(at: i64) -> DateTime<chrono_tz::Tz> {
    Rome.timestamp_opt(at, 0)
        .single()
        .expect("an instant has exactly one local time")
}

pub fn local_date(at: i64) -> NaiveDate {
    local(at).date_naive()
}

fn midnight(d: NaiveDate) -> i64 {
    // `earliest()` is None when the local midnight does not exist (not the case in Europe/Rome)
    let naive = d.and_hms_opt(0, 0, 0).expect("00:00:00 is a valid time");
    Rome.from_local_datetime(&naive)
        .earliest()
        .expect("midnight exists in Europe/Rome")
        .timestamp()
}

/// T5 - length of a local day in hours: 23, 24 or 25.
pub fn day_hours(d: NaiveDate) -> i64 {
    (midnight(d.succ_opt().expect("date in range")) - midnight(d)) / 3600
}

pub fn local_iso(at: i64) -> String {
    local(at).to_rfc3339()
}

#[derive(Debug, Default)]
pub struct Report {
    pub readings: usize,
    pub valid: Vec<Reading>,
    pub rejects: Vec<(usize, Reason)>,
}

/// T6 - consume the stream one line at a time; blank lines are skipped.
pub fn run(mut input: impl BufRead) -> io::Result<Report> {
    let mut rep = Report::default();
    let mut seen = HashSet::new();
    let mut buf = Vec::new();
    let mut n = 0;
    loop {
        buf.clear();
        if input.read_until(b'\n', &mut buf)? == 0 {
            break;
        }
        n += 1;
        let mut raw: &[u8] = &buf;
        if n == 1 && raw.starts_with(BOM) {
            raw = &raw[BOM.len()..];
        }
        if raw.iter().all(|b| JSON_WS.contains(b)) {
            continue;
        }
        rep.readings += 1;
        match decode(raw).and_then(|v| validate(&v)) {
            Err(reason) => rep.rejects.push((n, reason)),
            Ok(r) if !seen.insert((r.sensor.clone(), r.at)) => {
                rep.rejects.push((n, Reason::Duplicate))
            }
            Ok(r) => rep.valid.push(r),
        }
    }
    Ok(rep)
}

#[derive(Debug, Serialize)]
pub struct Day {
    pub sensor: String,
    pub date: String,
    pub hours: i64,
    pub count: usize,
    #[serde(rename = "min_tenths")]
    pub lo: i64,
    #[serde(rename = "max_tenths")]
    pub hi: i64,
    #[serde(rename = "avg_hundredths")]
    pub avg_h: i64, // mean in hundredths of a degree
}

#[derive(Debug, Serialize)]
pub struct Gap {
    pub sensor: String,
    #[serde(rename = "from")]
    pub start: String,
    #[serde(rename = "to")]
    pub end: String,
    pub minutes: i64,
}

/// n / d rounded half away from zero (d > 0); integer division differs by language.
pub fn div_round(n: i64, d: i64) -> i64 {
    let q = (2 * n.abs() + d) / (2 * d);
    if n < 0 { -q } else { q }
}

pub fn daily(valid: &[Reading]) -> Vec<Day> {
    let mut groups: BTreeMap<(&str, NaiveDate), Vec<i64>> = BTreeMap::new();
    for r in valid {
        groups
            .entry((r.sensor.as_str(), local_date(r.at)))
            .or_default()
            .push(r.tenths);
    }
    groups
        .into_iter()
        .map(|((s, d), v)| Day {
            sensor: s.to_string(),
            date: d.format("%Y-%m-%d").to_string(),
            hours: day_hours(d),
            count: v.len(),
            lo: *v.iter().min().unwrap(),
            hi: *v.iter().max().unwrap(),
            avg_h: div_round(v.iter().sum::<i64>() * 10, v.len() as i64),
        })
        .collect()
}

pub fn gaps(valid: &[Reading]) -> Vec<Gap> {
    let mut by_sensor: BTreeMap<&str, Vec<i64>> = BTreeMap::new();
    for r in valid {
        by_sensor.entry(r.sensor.as_str()).or_default().push(r.at);
    }
    let mut out = Vec::new();
    for (s, mut times) in by_sensor {
        times.sort_unstable();
        for w in times.windows(2) {
            if w[1] - w[0] > GAP_SECONDS {
                out.push(Gap {
                    sensor: s.to_string(),
                    start: local_iso(w[0]),
                    end: local_iso(w[1]),
                    minutes: (w[1] - w[0]) / 60,
                });
            }
        }
    }
    out
}

#[derive(Serialize)]
struct RejectOut {
    line: usize,
    reason: &'static str,
}

#[derive(Serialize)]
struct Doc {
    readings: usize,
    valid: usize,
    rejected: usize,
    days: Vec<Day>,
    gaps: Vec<Gap>,
    rejects: Vec<RejectOut>,
}

/// Integer in 10**-decimals units -> decimal text, no floats involved.
pub fn fixed(n: i64, decimals: u32) -> String {
    let p = 10_i64.pow(decimals);
    format!(
        "{}{}.{:0w$}",
        if n < 0 { "-" } else { "" },
        n.abs() / p,
        n.abs() % p,
        w = decimals as usize
    )
}

pub fn render_text(rep: &Report) -> String {
    let mut out = vec![format!(
        "readings={} valid={} rejected={}",
        rep.readings,
        rep.valid.len(),
        rep.rejects.len()
    )];
    for d in daily(&rep.valid) {
        out.push(format!(
            "day sensor={} date={} hours={} count={} min={} max={} avg={}",
            d.sensor,
            d.date,
            d.hours,
            d.count,
            fixed(d.lo, 1),
            fixed(d.hi, 1),
            fixed(d.avg_h, 2)
        ));
    }
    for g in gaps(&rep.valid) {
        out.push(format!(
            "gap sensor={} from={} to={} minutes={}",
            g.sensor, g.start, g.end, g.minutes
        ));
    }
    out.join("\n") + "\n"
}

pub fn render_json(rep: &Report) -> String {
    let doc = Doc {
        readings: rep.readings,
        valid: rep.valid.len(),
        rejected: rep.rejects.len(),
        days: daily(&rep.valid),
        gaps: gaps(&rep.valid),
        rejects: rep
            .rejects
            .iter()
            .map(|(n, r)| RejectOut {
                line: *n,
                reason: r.as_str(),
            })
            .collect(),
    };
    serde_json::to_string(&doc).expect("plain data always serializes") + "\n"
}
```

#### `src/main.rs`

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};
use std::process::ExitCode;

use coldchain::{Report, render_json, render_text, run};
use flate2::read::MultiGzDecoder;

/// A stream of bytes: stdin, a gzip file decompressed on the fly, or a plain file.
fn open_input(path: &str) -> io::Result<Box<dyn BufRead>> {
    Ok(match path {
        "-" => Box::new(io::stdin().lock()),
        p if p.ends_with(".gz") => Box::new(BufReader::new(MultiGzDecoder::new(File::open(p)?))),
        p => Box::new(BufReader::new(File::open(p)?)),
    })
}

fn read_report(path: &str) -> io::Result<Report> {
    run(open_input(path)?)
}

/// T8 - CLI: stream the file (or stdin), report on stdout, rejects on stderr.
fn main() -> ExitCode {
    let all: Vec<String> = std::env::args().skip(1).collect();
    let args: Vec<&String> = all.iter().filter(|a| *a != "--json").collect();
    let as_json = args.len() != all.len();
    let path = args.first().map_or("-", |s| s.as_str());
    let Ok(rep) = read_report(path) else {
        eprintln!("error: cannot read {path}");
        return ExitCode::from(2);
    };
    for (n, reason) in &rep.rejects {
        eprintln!("line {n}: rejected ({})", reason.as_str());
    }
    print!(
        "{}",
        if as_json {
            render_json(&rep)
        } else {
            render_text(&rep)
        }
    );
    ExitCode::from(if rep.rejects.is_empty() { 0 } else { 1 })
}
```

