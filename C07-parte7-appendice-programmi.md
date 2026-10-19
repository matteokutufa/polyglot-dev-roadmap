(parte 7 di 7: appendice: i quattro programmi completi) — file originale: capitoli/C07-oggetti-interfacce-trait-composizione.md

## Appendice — I quattro programmi completi

Eseguibili con: `python3 main.py notifications.csv`, `php main.php notifications.csv`, `rustc --edition 2024 main.rs -o main_rs && ./main_rs notifications.csv`, `go build -o main_go . && ./main_go notifications.csv` (con `go.mod` che dichiara `go 1.24`). Tutti usano soltanto la libreria standard.

#### Python (`main.py`) — `python3 main.py notifications.csv`

```python
import sys
from collections.abc import Iterator
from dataclasses import dataclass
from typing import Protocol

CHANNELS = ("email", "sms", "webhook")

# Simulated remote endpoints: recipient -> number of timeouts before success (-1 = never reachable).
FAULTS = {
    "flaky@example.com": 2,
    "+391111111111": 2,
    "https://example.com/flaky": 2,
    "+392222222222": 4,
    "down@example.com": -1,
    "https://example.com/down": -1,
}


@dataclass(frozen=True)
class Notification:
    """T2 - value object: one parsed row."""
    id: str
    channel: str
    recipient: str
    message: str


@dataclass(frozen=True)
class Outcome:
    """Result of one send: err is "" on success; attempts counts the transport calls."""
    err: str
    attempts: int


class Transport:
    """T3 - simulated remote endpoint. The state is private: callers can only deliver() and read delivered."""

    def __init__(self) -> None:
        self._attempts: dict[str, int] = {}   # notification id -> calls so far
        self._delivered = 0

    @property
    def delivered(self) -> int:
        return self._delivered

    def deliver(self, nid: str, recipient: str) -> str:
        n = self._attempts.get(nid, 0) + 1
        self._attempts[nid] = n
        fault = FAULTS.get(recipient, 0)
        if fault < 0:
            return "unreachable"
        if n <= fault:
            return "timeout"
        self._delivered += 1
        return ""


class Sender(Protocol):
    """T4 - the contract: anything with these three methods is a Sender."""

    def name(self) -> str: ...
    def send(self, n: Notification) -> Outcome: ...
    def delivered(self) -> int: ...


class Email:
    def __init__(self) -> None:
        self._transport = Transport()

    def name(self) -> str:
        return "email"

    def delivered(self) -> int:
        return self._transport.delivered

    def send(self, n: Notification) -> Outcome:
        parts = n.recipient.split("@")
        if len(parts) != 2 or not parts[0] or "." not in parts[1] or " " in n.recipient:
            return Outcome("invalid_recipient", 1)
        return Outcome(self._transport.deliver(n.id, n.recipient), 1)


class Sms:
    MAX_LEN = 40

    def __init__(self) -> None:
        self._transport = Transport()

    def name(self) -> str:
        return "sms"

    def delivered(self) -> int:
        return self._transport.delivered

    def send(self, n: Notification) -> Outcome:
        digits = n.recipient[1:]
        if not (n.recipient.startswith("+") and 8 <= len(digits) <= 15
                and digits.isascii() and digits.isdigit()):
            return Outcome("invalid_recipient", 1)
        if len(n.message) > self.MAX_LEN:
            return Outcome("too_long", 1)
        return Outcome(self._transport.deliver(n.id, n.recipient), 1)


class Webhook:
    def __init__(self) -> None:
        self._transport = Transport()

    def name(self) -> str:
        return "webhook"

    def delivered(self) -> int:
        return self._transport.delivered

    def send(self, n: Notification) -> Outcome:
        r = n.recipient
        if not r.startswith("https://") or len(r) == len("https://") or " " in r:
            return Outcome("invalid_recipient", 1)
        return Outcome(self._transport.deliver(n.id, r), 1)


class Retrying:
    """T5 - decorator: it is a Sender and it HAS a Sender. Retries only transient errors."""
    TRANSIENT = ("timeout",)

    def __init__(self, inner: Sender, max_attempts: int):
        self._inner = inner
        self._max = max_attempts

    def name(self) -> str:
        return self._inner.name()

    def delivered(self) -> int:
        return self._inner.delivered()

    def send(self, n: Notification) -> Outcome:
        total = 0
        while True:
            o = self._inner.send(n)
            total += o.attempts
            if o.err not in self.TRANSIENT or total >= self._max:
                return Outcome(o.err, total)


class Audit:
    """T5 - decorator: logs every call to the wrapped Sender on stderr."""

    def __init__(self, inner: Sender):
        self._inner = inner

    def name(self) -> str:
        return self._inner.name()

    def delivered(self) -> int:
        return self._inner.delivered()

    def send(self, n: Notification) -> Outcome:
        o = self._inner.send(n)
        print(f"audit channel={self.name()} id={n.id} result={o.err or 'ok'}", file=sys.stderr)
        return o


def read_rows(path: str) -> Iterator[tuple[int, str]]:
    """T1 - lazy: yield (line_number, text), header skipped."""
    with open(path, encoding="utf-8") as f:
        next(f, None)
        for n, line in enumerate(f, start=2):
            line = line.strip()
            if line:
                yield n, line


def parse(line: str) -> tuple[Notification | None, str]:
    """T2 - return (Notification, "") or (None, reason)."""
    parts = [p.strip() for p in line.split(",", 3)]
    if len(parts) != 4 or not parts[0]:
        return None, "fields"
    nid, channel, recipient, message = parts
    if channel not in CHANNELS:
        return None, "channel"
    if not recipient:
        return None, "recipient"
    if not message:
        return None, "message"
    return Notification(nid, channel, recipient, message), ""


class Dispatcher:
    """T6 - routes a Notification to the Sender of its channel. It knows only the Sender contract."""

    def __init__(self, routes: dict[str, Sender]) -> None:
        self._routes = routes
        self._seen: set[str] = set()
        self._failed = dict.fromkeys(routes, 0)

    def failed(self, channel: str) -> int:
        return self._failed[channel]

    def dispatch(self, n: Notification) -> tuple[Outcome | None, str]:
        """Return (Outcome, "") or (None, reason)."""
        if n.id in self._seen:
            return None, "duplicate"
        self._seen.add(n.id)
        out = self._routes[n.channel].send(n)
        if out.err:
            self._failed[n.channel] += 1
        return out, ""


def build_routes() -> dict[str, Sender]:
    """Composition at the edge: each channel gets its own stack of wrappers."""
    return {
        "email": Retrying(Audit(Email()), 3),
        "sms": Retrying(Audit(Sms()), 2),
        "webhook": Retrying(Audit(Webhook()), 3),
    }


def run(path: str) -> int:
    routes = build_routes()
    disp = Dispatcher(routes)
    rows = sent = failed = rejected = 0
    for n, line in read_rows(path):
        rows += 1
        note, reason = parse(line)
        if note is None:
            rejected += 1
            print(f"reject line={n} reason={reason}", file=sys.stderr)
            continue
        out, reason = disp.dispatch(note)
        if out is None:
            rejected += 1
            print(f"reject line={n} reason={reason}", file=sys.stderr)
            continue
        if out.err:
            failed += 1
            print(f"id={note.id} channel={note.channel} status=failed reason={out.err} attempts={out.attempts}")
        else:
            sent += 1
            print(f"id={note.id} channel={note.channel} status=sent attempts={out.attempts}")
    # T7 - report and exit code
    print(f"rows={rows} sent={sent} failed={failed} rejected={rejected}")
    for ch in CHANNELS:
        print(f"channel={ch} delivered={routes[ch].delivered()} failed={disp.failed(ch)}")
    return 1 if failed or rejected else 0


def main() -> int:
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

#### PHP (`main.php`) — `php main.php notifications.csv`

```php
<?php
declare(strict_types=1);

const CHANNELS = ['email', 'sms', 'webhook'];

// Simulated remote endpoints: recipient => number of timeouts before success (-1 = never reachable).
const FAULTS = [
    'flaky@example.com' => 2,
    '+391111111111' => 2,
    'https://example.com/flaky' => 2,
    '+392222222222' => 4,
    'down@example.com' => -1,
    'https://example.com/down' => -1,
];

/** T2 - value object: one parsed row. */
final readonly class Notification
{
    public function __construct(
        public string $id,
        public string $channel,
        public string $recipient,
        public string $message,
    ) {
    }
}

/** Result of one send: err is '' on success; attempts counts the transport calls. */
final readonly class Outcome
{
    public function __construct(public string $err, public int $attempts)
    {
    }
}

/** T3 - simulated remote endpoint. The state is private: callers can only deliver() and read delivered(). */
final class Transport
{
    /** @var array<string,int> notification id => calls so far */
    private array $attempts = [];
    private int $delivered = 0;

    public function delivered(): int
    {
        return $this->delivered;
    }

    public function deliver(string $id, string $recipient): string
    {
        $n = ($this->attempts[$id] ?? 0) + 1;
        $this->attempts[$id] = $n;
        $fault = FAULTS[$recipient] ?? 0;
        if ($fault < 0) {
            return 'unreachable';
        }
        if ($n <= $fault) {
            return 'timeout';
        }
        $this->delivered++;
        return '';
    }
}

/** T4 - the contract: a class is a Sender only if it declares `implements Sender`. */
interface Sender
{
    public function name(): string;

    public function send(Notification $n): Outcome;

    public function delivered(): int;
}

final class Email implements Sender
{
    private Transport $transport;

    public function __construct()
    {
        $this->transport = new Transport();
    }

    public function name(): string
    {
        return 'email';
    }

    public function delivered(): int
    {
        return $this->transport->delivered();
    }

    public function send(Notification $n): Outcome
    {
        $parts = explode('@', $n->recipient);
        if (count($parts) !== 2 || $parts[0] === '' || !str_contains($parts[1], '.') || str_contains($n->recipient, ' ')) {
            return new Outcome('invalid_recipient', 1);
        }
        return new Outcome($this->transport->deliver($n->id, $n->recipient), 1);
    }
}

final class Sms implements Sender
{
    private const MAX_LEN = 40;
    private Transport $transport;

    public function __construct()
    {
        $this->transport = new Transport();
    }

    public function name(): string
    {
        return 'sms';
    }

    public function delivered(): int
    {
        return $this->transport->delivered();
    }

    public function send(Notification $n): Outcome
    {
        if (preg_match('/^\+[0-9]{8,15}\z/', $n->recipient) !== 1) {
            return new Outcome('invalid_recipient', 1);
        }
        if (mb_strlen($n->message) > self::MAX_LEN) {
            return new Outcome('too_long', 1);
        }
        return new Outcome($this->transport->deliver($n->id, $n->recipient), 1);
    }
}

final class Webhook implements Sender
{
    private Transport $transport;

    public function __construct()
    {
        $this->transport = new Transport();
    }

    public function name(): string
    {
        return 'webhook';
    }

    public function delivered(): int
    {
        return $this->transport->delivered();
    }

    public function send(Notification $n): Outcome
    {
        $r = $n->recipient;
        if (!str_starts_with($r, 'https://') || strlen($r) === strlen('https://') || str_contains($r, ' ')) {
            return new Outcome('invalid_recipient', 1);
        }
        return new Outcome($this->transport->deliver($n->id, $r), 1);
    }
}

/** T5 - decorator: it is a Sender and it HAS a Sender. Retries only transient errors. */
final class Retrying implements Sender
{
    private const TRANSIENT = ['timeout'];

    public function __construct(private readonly Sender $inner, private readonly int $maxAttempts)
    {
    }

    public function name(): string
    {
        return $this->inner->name();
    }

    public function delivered(): int
    {
        return $this->inner->delivered();
    }

    public function send(Notification $n): Outcome
    {
        $total = 0;
        while (true) {
            $o = $this->inner->send($n);
            $total += $o->attempts;
            if (!in_array($o->err, self::TRANSIENT, true) || $total >= $this->maxAttempts) {
                return new Outcome($o->err, $total);
            }
        }
    }
}

/** T5 - decorator: logs every call to the wrapped Sender on stderr. */
final class Audit implements Sender
{
    public function __construct(private readonly Sender $inner)
    {
    }

    public function name(): string
    {
        return $this->inner->name();
    }

    public function delivered(): int
    {
        return $this->inner->delivered();
    }

    public function send(Notification $n): Outcome
    {
        $o = $this->inner->send($n);
        fwrite(STDERR, "audit channel={$this->name()} id=$n->id result=" . ($o->err ?: 'ok') . "\n");
        return $o;
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

/** T2 - return [Notification, ''] or [null, reason]. */
function parse(string $line): array
{
    $parts = array_map('trim', explode(',', $line, 4));
    if (count($parts) !== 4 || $parts[0] === '') {
        return [null, 'fields'];
    }
    [$id, $channel, $recipient, $message] = $parts;
    if (!in_array($channel, CHANNELS, true)) {
        return [null, 'channel'];
    }
    if ($recipient === '') {
        return [null, 'recipient'];
    }
    if ($message === '') {
        return [null, 'message'];
    }
    return [new Notification($id, $channel, $recipient, $message), ''];
}

/** T6 - routes a Notification to the Sender of its channel. It knows only the Sender contract. */
final class Dispatcher
{
    /** @var array<string,true> */
    private array $seen = [];
    /** @var array<string,int> */
    private array $failed;

    /** @param array<string,Sender> $routes */
    public function __construct(private readonly array $routes)
    {
        $this->failed = array_fill_keys(array_keys($routes), 0);
    }

    public function failed(string $channel): int
    {
        return $this->failed[$channel];
    }

    /** @return array{0: ?Outcome, 1: string} */
    public function dispatch(Notification $n): array
    {
        if (isset($this->seen[$n->id])) {
            return [null, 'duplicate'];
        }
        $this->seen[$n->id] = true;
        $out = $this->routes[$n->channel]->send($n);
        if ($out->err !== '') {
            $this->failed[$n->channel]++;
        }
        return [$out, ''];
    }
}

/** Composition at the edge: each channel gets its own stack of wrappers. */
function buildRoutes(): array
{
    return [
        'email' => new Retrying(new Audit(new Email()), 3),
        'sms' => new Retrying(new Audit(new Sms()), 2),
        'webhook' => new Retrying(new Audit(new Webhook()), 3),
    ];
}

function run(string $path): int
{
    $routes = buildRoutes();
    $disp = new Dispatcher($routes);
    $rows = $sent = $failed = $rejected = 0;
    foreach (readRows($path) as $n => $line) {
        $rows++;
        [$note, $reason] = parse($line);
        if ($note === null) {
            $rejected++;
            fwrite(STDERR, "reject line=$n reason=$reason\n");
            continue;
        }
        [$out, $reason] = $disp->dispatch($note);
        if ($out === null) {
            $rejected++;
            fwrite(STDERR, "reject line=$n reason=$reason\n");
            continue;
        }
        if ($out->err !== '') {
            $failed++;
            echo "id=$note->id channel=$note->channel status=failed reason=$out->err attempts=$out->attempts\n";
        } else {
            $sent++;
            echo "id=$note->id channel=$note->channel status=sent attempts=$out->attempts\n";
        }
    }
    // T7 - report and exit code
    echo "rows=$rows sent=$sent failed=$failed rejected=$rejected\n";
    foreach (CHANNELS as $ch) {
        echo "channel=$ch delivered=" . $routes[$ch]->delivered() . ' failed=' . $disp->failed($ch) . "\n";
    }
    return ($failed > 0 || $rejected > 0) ? 1 : 0;
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

#### Rust (`main.rs`) — `rustc --edition 2024 main.rs -o main_rs && ./main_rs notifications.csv`

```rust
use std::collections::{HashMap, HashSet};
use std::fs::File;
use std::io::{BufRead, BufReader};
use std::process::ExitCode;

const CHANNELS: [&str; 3] = ["email", "sms", "webhook"];

// Simulated remote endpoints: recipient -> number of timeouts before success (-1 = never reachable).
const FAULTS: [(&str, i32); 6] = [
    ("flaky@example.com", 2),
    ("+391111111111", 2),
    ("https://example.com/flaky", 2),
    ("+392222222222", 4),
    ("down@example.com", -1),
    ("https://example.com/down", -1),
];

/// T2 - value object: one parsed row.
#[derive(Debug, Clone, PartialEq, Eq)]
struct Notification {
    id: String,
    channel: String,
    recipient: String,
    message: String,
}

/// Result of one send: `err` is None on success; `attempts` counts the transport calls.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Outcome {
    err: Option<&'static str>,
    attempts: u32,
}

/// T3 - simulated remote endpoint. The fields are private to the module.
mod remote {
    use super::FAULTS;
    use std::collections::HashMap;

    pub struct Transport {
        attempts: HashMap<String, i32>, // notification id -> calls so far
        delivered: u32,
    }

    impl Transport {
        pub fn new() -> Transport {
            Transport { attempts: HashMap::new(), delivered: 0 }
        }

        pub fn delivered(&self) -> u32 {
            self.delivered
        }

        pub fn deliver(&mut self, id: &str, recipient: &str) -> Option<&'static str> {
            let n = self.attempts.entry(id.to_string()).or_insert(0);
            *n += 1;
            let fault = FAULTS.iter().find(|(r, _)| *r == recipient).map_or(0, |(_, f)| *f);
            if fault < 0 {
                return Some("unreachable");
            }
            if *n <= fault {
                return Some("timeout");
            }
            self.delivered += 1;
            None
        }
    }
}
use remote::Transport;

/// T4 - the contract. A type is a Sender only through an explicit `impl Sender for ...`.
trait Sender {
    fn name(&self) -> &'static str;
    fn send(&mut self, n: &Notification) -> Outcome;
    fn delivered(&self) -> u32;
}

struct Email {
    transport: Transport,
}

impl Email {
    fn new() -> Email {
        Email { transport: Transport::new() }
    }
}

impl Sender for Email {
    fn name(&self) -> &'static str {
        "email"
    }

    fn delivered(&self) -> u32 {
        self.transport.delivered()
    }

    fn send(&mut self, n: &Notification) -> Outcome {
        let parts: Vec<&str> = n.recipient.split('@').collect();
        if parts.len() != 2 || parts[0].is_empty() || !parts[1].contains('.') || n.recipient.contains(' ') {
            return Outcome { err: Some("invalid_recipient"), attempts: 1 };
        }
        Outcome { err: self.transport.deliver(&n.id, &n.recipient), attempts: 1 }
    }
}

struct Sms {
    transport: Transport,
}

impl Sms {
    const MAX_LEN: usize = 40;

    fn new() -> Sms {
        Sms { transport: Transport::new() }
    }
}

impl Sender for Sms {
    fn name(&self) -> &'static str {
        "sms"
    }

    fn delivered(&self) -> u32 {
        self.transport.delivered()
    }

    fn send(&mut self, n: &Notification) -> Outcome {
        let digits = n.recipient.strip_prefix('+').unwrap_or("");
        if !(8..=15).contains(&digits.len()) || !digits.bytes().all(|b| b.is_ascii_digit()) {
            return Outcome { err: Some("invalid_recipient"), attempts: 1 };
        }
        if n.message.chars().count() > Self::MAX_LEN {
            return Outcome { err: Some("too_long"), attempts: 1 };
        }
        Outcome { err: self.transport.deliver(&n.id, &n.recipient), attempts: 1 }
    }
}

struct Webhook {
    transport: Transport,
}

impl Webhook {
    fn new() -> Webhook {
        Webhook { transport: Transport::new() }
    }
}

impl Sender for Webhook {
    fn name(&self) -> &'static str {
        "webhook"
    }

    fn delivered(&self) -> u32 {
        self.transport.delivered()
    }

    fn send(&mut self, n: &Notification) -> Outcome {
        let r = &n.recipient;
        if !r.starts_with("https://") || r.len() == "https://".len() || r.contains(' ') {
            return Outcome { err: Some("invalid_recipient"), attempts: 1 };
        }
        Outcome { err: self.transport.deliver(&n.id, r), attempts: 1 }
    }
}

/// T5 - decorator: it is a Sender and it HAS a Sender (static dispatch: S is known at compile time).
struct Retrying<S: Sender> {
    inner: S,
    max_attempts: u32,
}

impl<S: Sender> Sender for Retrying<S> {
    fn name(&self) -> &'static str {
        self.inner.name()
    }

    fn delivered(&self) -> u32 {
        self.inner.delivered()
    }

    fn send(&mut self, n: &Notification) -> Outcome {
        let mut total = 0;
        loop {
            let o = self.inner.send(n);
            total += o.attempts;
            let transient = o.err == Some("timeout");
            if !transient || total >= self.max_attempts {
                return Outcome { err: o.err, attempts: total };
            }
        }
    }
}

/// T5 - decorator: logs every call to the wrapped Sender on stderr.
struct Audit<S: Sender> {
    inner: S,
}

impl<S: Sender> Sender for Audit<S> {
    fn name(&self) -> &'static str {
        self.inner.name()
    }

    fn delivered(&self) -> u32 {
        self.inner.delivered()
    }

    fn send(&mut self, n: &Notification) -> Outcome {
        let o = self.inner.send(n);
        eprintln!("audit channel={} id={} result={}", self.name(), n.id, o.err.unwrap_or("ok"));
        o
    }
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

/// T2 - Ok(Notification) or Err(reason).
fn parse(line: &str) -> Result<Notification, &'static str> {
    let p: Vec<&str> = line.splitn(4, ',').map(str::trim).collect();
    if p.len() != 4 || p[0].is_empty() {
        return Err("fields");
    }
    if !CHANNELS.contains(&p[1]) {
        return Err("channel");
    }
    if p[2].is_empty() {
        return Err("recipient");
    }
    if p[3].is_empty() {
        return Err("message");
    }
    Ok(Notification {
        id: p[0].to_string(),
        channel: p[1].to_string(),
        recipient: p[2].to_string(),
        message: p[3].to_string(),
    })
}

/// T6 - routes a Notification to the Sender of its channel. It knows only the Sender contract
/// (dynamic dispatch: each route is a `Box<dyn Sender>`, a fat pointer to data + vtable).
struct Dispatcher {
    routes: HashMap<&'static str, Box<dyn Sender>>,
    seen: HashSet<String>,
    failed: HashMap<&'static str, u32>,
}

impl Dispatcher {
    fn new(routes: HashMap<&'static str, Box<dyn Sender>>) -> Dispatcher {
        let failed = routes.keys().map(|k| (*k, 0)).collect();
        Dispatcher { routes, seen: HashSet::new(), failed }
    }

    fn failed(&self, channel: &str) -> u32 {
        self.failed[channel]
    }

    fn delivered(&self, channel: &str) -> u32 {
        self.routes[channel].delivered()
    }

    fn dispatch(&mut self, n: &Notification) -> Result<Outcome, &'static str> {
        if !self.seen.insert(n.id.clone()) {
            return Err("duplicate");
        }
        let route = self.routes.get_mut(n.channel.as_str()).expect("channel validated by parse");
        let out = route.send(n);
        if out.err.is_some() {
            *self.failed.get_mut(n.channel.as_str()).unwrap() += 1;
        }
        Ok(out)
    }
}

/// Composition at the edge: each channel gets its own stack of wrappers.
fn build_routes() -> HashMap<&'static str, Box<dyn Sender>> {
    let mut routes: HashMap<&'static str, Box<dyn Sender>> = HashMap::new();
    routes.insert("email", Box::new(Retrying { inner: Audit { inner: Email::new() }, max_attempts: 3 }));
    routes.insert("sms", Box::new(Retrying { inner: Audit { inner: Sms::new() }, max_attempts: 2 }));
    routes.insert("webhook", Box::new(Retrying { inner: Audit { inner: Webhook::new() }, max_attempts: 3 }));
    routes
}

fn run(path: &str) -> std::io::Result<u8> {
    let mut disp = Dispatcher::new(build_routes());
    let (mut rows, mut sent, mut failed, mut rejected) = (0, 0, 0, 0);
    for (n, line) in read_rows(path)? {
        rows += 1;
        let note = match parse(&line) {
            Ok(x) => x,
            Err(reason) => {
                rejected += 1;
                eprintln!("reject line={n} reason={reason}");
                continue;
            }
        };
        let out = match disp.dispatch(&note) {
            Ok(o) => o,
            Err(reason) => {
                rejected += 1;
                eprintln!("reject line={n} reason={reason}");
                continue;
            }
        };
        match out.err {
            Some(reason) => {
                failed += 1;
                println!("id={} channel={} status=failed reason={} attempts={}", note.id, note.channel, reason, out.attempts);
            }
            None => {
                sent += 1;
                println!("id={} channel={} status=sent attempts={}", note.id, note.channel, out.attempts);
            }
        }
    }
    // T7 - report and exit code
    println!("rows={rows} sent={sent} failed={failed} rejected={rejected}");
    for ch in CHANNELS {
        println!("channel={} delivered={} failed={}", ch, disp.delivered(ch), disp.failed(ch));
    }
    Ok(if failed > 0 || rejected > 0 { 1 } else { 0 })
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

#### Go (`main.go`) — `go build -o main_go . && ./main_go notifications.csv`

```go
package main

import (
	"bufio"
	"fmt"
	"iter"
	"os"
	"strings"
	"unicode/utf8"
)

var channels = []string{"email", "sms", "webhook"}

// Simulated remote endpoints: recipient -> number of timeouts before success (-1 = never reachable).
var faults = map[string]int{
	"flaky@example.com":         2,
	"+391111111111":             2,
	"https://example.com/flaky": 2,
	"+392222222222":             4,
	"down@example.com":          -1,
	"https://example.com/down":  -1,
}

// Notification is T2 - a value object: one parsed row.
type Notification struct{ ID, Channel, Recipient, Message string }

// Outcome is the result of one send: Err is "" on success; Attempts counts the transport calls.
type Outcome struct {
	Err      string
	Attempts int
}

// Transport is T3 - a simulated remote endpoint. The fields are unexported: only this package can touch them.
type Transport struct {
	attempts  map[string]int // notification id -> calls so far
	delivered int
}

func NewTransport() *Transport { return &Transport{attempts: map[string]int{}} }

func (t *Transport) Delivered() int { return t.delivered }

// Deliver needs a pointer receiver: it modifies the Transport (e2).
func (t *Transport) Deliver(id, recipient string) string {
	t.attempts[id]++
	n := t.attempts[id]
	fault := faults[recipient]
	if fault < 0 {
		return "unreachable"
	}
	if n <= fault {
		return "timeout"
	}
	t.delivered++
	return ""
}

// Sender is T4 - the contract. Any type with these three methods satisfies it: no "implements" needed.
type Sender interface {
	Name() string
	Send(n Notification) Outcome
	Delivered() int
}

type Email struct{ transport *Transport }

func NewEmail() *Email { return &Email{transport: NewTransport()} }

func (e *Email) Name() string   { return "email" }
func (e *Email) Delivered() int { return e.transport.Delivered() }

func (e *Email) Send(n Notification) Outcome {
	parts := strings.Split(n.Recipient, "@")
	if len(parts) != 2 || parts[0] == "" || !strings.Contains(parts[1], ".") || strings.Contains(n.Recipient, " ") {
		return Outcome{"invalid_recipient", 1}
	}
	return Outcome{e.transport.Deliver(n.ID, n.Recipient), 1}
}

type Sms struct{ transport *Transport }

const smsMaxLen = 40

func NewSms() *Sms { return &Sms{transport: NewTransport()} }

func (s *Sms) Name() string   { return "sms" }
func (s *Sms) Delivered() int { return s.transport.Delivered() }

func (s *Sms) Send(n Notification) Outcome {
	digits := strings.TrimPrefix(n.Recipient, "+")
	if digits == n.Recipient || len(digits) < 8 || len(digits) > 15 || strings.Trim(digits, "0123456789") != "" {
		return Outcome{"invalid_recipient", 1}
	}
	if utf8.RuneCountInString(n.Message) > smsMaxLen {
		return Outcome{"too_long", 1}
	}
	return Outcome{s.transport.Deliver(n.ID, n.Recipient), 1}
}

type Webhook struct{ transport *Transport }

func NewWebhook() *Webhook { return &Webhook{transport: NewTransport()} }

func (w *Webhook) Name() string   { return "webhook" }
func (w *Webhook) Delivered() int { return w.transport.Delivered() }

func (w *Webhook) Send(n Notification) Outcome {
	r := n.Recipient
	if !strings.HasPrefix(r, "https://") || len(r) == len("https://") || strings.Contains(r, " ") {
		return Outcome{"invalid_recipient", 1}
	}
	return Outcome{w.transport.Deliver(n.ID, r), 1}
}

// Retrying is T5 - a decorator. It EMBEDS a Sender: Name and Delivered are promoted
// from the embedded value, and only Send is overridden.
type Retrying struct {
	Sender
	maxAttempts int
}

func (r Retrying) Send(n Notification) Outcome {
	total := 0
	for {
		o := r.Sender.Send(n)
		total += o.Attempts
		if o.Err != "timeout" || total >= r.maxAttempts {
			return Outcome{o.Err, total}
		}
	}
}

// Audit is T5 - a decorator: logs every call to the wrapped Sender on stderr.
type Audit struct{ Sender }

func (a Audit) Send(n Notification) Outcome {
	o := a.Sender.Send(n)
	res := o.Err
	if res == "" {
		res = "ok"
	}
	fmt.Fprintf(os.Stderr, "audit channel=%s id=%s result=%s\n", a.Name(), n.ID, res)
	return o
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

// parse is T2 - returns a Notification or a rejection reason.
func parse(line string) (Notification, string) {
	p := strings.SplitN(line, ",", 4)
	for i := range p {
		p[i] = strings.TrimSpace(p[i])
	}
	if len(p) != 4 || p[0] == "" {
		return Notification{}, "fields"
	}
	known := false
	for _, c := range channels {
		known = known || c == p[1]
	}
	switch {
	case !known:
		return Notification{}, "channel"
	case p[2] == "":
		return Notification{}, "recipient"
	case p[3] == "":
		return Notification{}, "message"
	}
	return Notification{p[0], p[1], p[2], p[3]}, ""
}

// Dispatcher is T6 - routes a Notification to the Sender of its channel. It knows only the Sender contract.
type Dispatcher struct {
	routes map[string]Sender
	seen   map[string]bool
	failed map[string]int
}

func NewDispatcher(routes map[string]Sender) *Dispatcher {
	return &Dispatcher{routes: routes, seen: map[string]bool{}, failed: map[string]int{}}
}

func (d *Dispatcher) Failed(channel string) int { return d.failed[channel] }

// Dispatch returns the outcome, or a rejection reason.
func (d *Dispatcher) Dispatch(n Notification) (out Outcome, reason string) {
	if d.seen[n.ID] {
		return Outcome{}, "duplicate"
	}
	d.seen[n.ID] = true
	out = d.routes[n.Channel].Send(n)
	if out.Err != "" {
		d.failed[n.Channel]++
	}
	return out, ""
}

// buildRoutes: composition at the edge - each channel gets its own stack of wrappers.
func buildRoutes() map[string]Sender {
	return map[string]Sender{
		"email":   Retrying{Audit{NewEmail()}, 3},
		"sms":     Retrying{Audit{NewSms()}, 2},
		"webhook": Retrying{Audit{NewWebhook()}, 3},
	}
}

func run(path string) int {
	rows, err := readRows(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "cannot open %s\n", path)
		return 2
	}
	routes := buildRoutes()
	disp := NewDispatcher(routes)
	var nrows, sent, failed, rejected int
	for n, line := range rows {
		nrows++
		note, reason := parse(line)
		var out Outcome
		if reason == "" {
			out, reason = disp.Dispatch(note)
		}
		if reason != "" {
			rejected++
			fmt.Fprintf(os.Stderr, "reject line=%d reason=%s\n", n, reason)
			continue
		}
		if out.Err != "" {
			failed++
			fmt.Printf("id=%s channel=%s status=failed reason=%s attempts=%d\n", note.ID, note.Channel, out.Err, out.Attempts)
		} else {
			sent++
			fmt.Printf("id=%s channel=%s status=sent attempts=%d\n", note.ID, note.Channel, out.Attempts)
		}
	}
	// T7 - report and exit code
	fmt.Printf("rows=%d sent=%d failed=%d rejected=%d\n", nrows, sent, failed, rejected)
	for _, ch := range channels {
		fmt.Printf("channel=%s delivered=%d failed=%d\n", ch, routes[ch].Delivered(), disp.Failed(ch))
	}
	if failed > 0 || rejected > 0 {
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

**Limiti dichiarati di questa bozza:** la rete è simulata (nessun socket, nessun SMTP, nessun HTTP: la lettura dei limiti reali — timeout, TLS, rate limit — è in C16); il benchmark di e10 è stato eseguito in un ambiente condiviso e vale come ordine di grandezza; il comportamento dei generics di Go riportato in e10 dipende dalla versione del compilatore (1.24.7 qui, baseline del corso 1.25+: da riverificare); PHPStan/Psalm e `staticcheck` non sono stati eseguiti; `mypy --strict` è stato eseguito solo sul programma `main.py` e sull'esperimento e6; il comportamento di `trim()` PHP su altri spazi Unicode oltre a U+00A0 non è stato provato.
