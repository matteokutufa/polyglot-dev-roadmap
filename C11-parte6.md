> File C11-parte6.md (parte 6 di 6) — Appendice — i programmi completi … Appendice — i programmi completi

## Appendice — i programmi completi

Esecuzione: `jobs.csv` nella stessa directory; codice di uscita 0/1/2 come nel capitolo.

### Python (`main.py`) — `python3 main.py jobs.csv`

```python
"""Job runner: parse jobs, run them on a worker pool, cancel everything on a fatal job."""
import sys
import threading
import queue
from dataclasses import dataclass

WORKERS = 4
JOB_TIMEOUT_MS = 200
OPS = ("collatz", "io", "batch", "poison")


@dataclass(frozen=True)
class Job:
    id: int
    op: str
    arg: int


@dataclass(frozen=True)
class Outcome:
    job: Job
    status: str  # ok | timeout | fatal | cancelled
    detail: str = ""


def parse_jobs(lines):
    """T1 - validate every line; return (jobs, rejects) where rejects are (line_no, reason)."""
    jobs, rejects, seen = [], [], set()
    for no, raw in enumerate(lines, 1):
        line = raw.strip()
        if no == 1 or not line:
            continue  # header and blank lines
        parts = line.split(",")
        if len(parts) != 3:
            rejects.append((no, "fields"))
            continue
        sid, op, sarg = (p.strip() for p in parts)
        if not sid.isascii() or not sid.isdigit() or int(sid) == 0:
            rejects.append((no, "id"))
        elif int(sid) in seen:
            rejects.append((no, "duplicate"))
        elif op not in OPS:
            rejects.append((no, "op"))
        elif not sarg.isascii() or not sarg.isdigit():
            rejects.append((no, "arg"))
        else:
            seen.add(int(sid))
            jobs.append(Job(int(sid), op, int(sarg)))
    return jobs, rejects


def collatz_steps(n):
    """Number of Collatz steps to reach 1 (0 for n <= 1)."""
    steps = 0
    while n > 1:
        n = n // 2 if n % 2 == 0 else 3 * n + 1
        steps += 1
    return steps


class Shared:
    """State shared by all workers: the cancel flag, the results and the steps counter."""

    def __init__(self):
        self.cancel = threading.Event()  # T6 - set once, never cleared
        self.lock = threading.Lock()
        self.results = []
        self.steps_total = 0

    def add_steps(self, n):
        with self.lock:  # T5 - += on an int is read-modify-write, not atomic
            self.steps_total += n

    def record(self, outcome):
        with self.lock:
            self.results.append(outcome)


def run_job(job, shared):
    """T4 - execute one job; every wait is cancel-aware."""
    if shared.cancel.is_set():
        return Outcome(job, "cancelled")
    if job.op == "collatz":
        steps = collatz_steps(job.arg)
        shared.add_steps(steps)
        return Outcome(job, "ok", f"steps={steps}")
    if job.op == "io":
        wait_ms = min(job.arg, JOB_TIMEOUT_MS)
        if shared.cancel.wait(wait_ms / 1000):  # True if cancelled while waiting
            return Outcome(job, "cancelled")
        if job.arg > JOB_TIMEOUT_MS:
            return Outcome(job, "timeout", f"limit={JOB_TIMEOUT_MS}")
        return Outcome(job, "ok", f"waited={job.arg}")
    if job.op == "batch":  # long job without a timeout: only cancellation stops it
        if shared.cancel.wait(job.arg / 1000):
            return Outcome(job, "cancelled")
        return Outcome(job, "ok", f"waited={job.arg}")
    # poison: wait, then trip the cancel flag for everybody (T6)
    if shared.cancel.wait(job.arg / 1000):
        return Outcome(job, "cancelled")
    shared.cancel.set()
    return Outcome(job, "fatal")


def worker(jobs_q, shared):
    while True:
        job = jobs_q.get()
        if job is None:  # sentinel: no more work
            return
        shared.record(run_job(job, shared))


def run_pool(jobs):
    """T2, T3, T7 - queue the jobs, start W threads, wait for them."""
    shared = Shared()
    jobs_q = queue.Queue()
    for job in jobs:
        jobs_q.put(job)
    for _ in range(WORKERS):
        jobs_q.put(None)
    threads = [threading.Thread(target=worker, args=(jobs_q, shared)) for _ in range(WORKERS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return sorted(shared.results, key=lambda o: o.job.id), shared.steps_total


def report(results, steps_total, rejected):
    """T8 - stable, diff-able output; exit code is the contract."""
    count = {"ok": 0, "timeout": 0, "fatal": 0, "cancelled": 0}
    for o in results:
        count[o.status] += 1
        tail = f" {o.detail}" if o.detail else ""
        print(f"job {o.job.id} {o.job.op} {o.status}{tail}")
    print(f"jobs={len(results)} ok={count['ok']} timeout={count['timeout']} "
          f"fatal={count['fatal']} cancelled={count['cancelled']} rejected={rejected}")
    print(f"steps_total={steps_total} workers={WORKERS}")
    return 0 if count["ok"] == len(results) and rejected == 0 else 1


def main(argv):
    if len(argv) != 2:
        print("usage: main.py jobs.csv", file=sys.stderr)
        return 2
    try:
        with open(argv[1], encoding="utf-8") as f:
            lines = f.read().splitlines()
    except OSError as e:
        print(f"error: cannot read {argv[1]}", file=sys.stderr)
        return 2
    jobs, rejects = parse_jobs(lines)
    for no, reason in rejects:
        print(f"reject line {no}: {reason}", file=sys.stderr)
    results, steps_total = run_pool(jobs)
    return report(results, steps_total, len(rejects))


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

### PHP (`main.php`) — `php main.php jobs.csv`

```php
<?php
declare(strict_types=1);

// Job runner: parse jobs, run them on a pool of Fibers driven by a tiny event loop.
// PHP has no shared-memory threads: concurrency here is cooperative and single-threaded.

const WORKERS = 4;
const JOB_TIMEOUT_MS = 200;
const OPS = ['collatz', 'io', 'batch', 'poison'];

/** T1 - validate every line; return [jobs, rejects] with rejects as [line_no, reason]. */
function parseJobs(array $lines): array
{
    $jobs = [];
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
        [$sid, $op, $sarg] = $parts;
        $isDigits = fn(string $s): bool => preg_match('/^[0-9]+$/D', $s) === 1;
        if (!$isDigits($sid) || (int) $sid === 0) {
            $rejects[] = [$no, 'id'];
        } elseif (isset($seen[(int) $sid])) {
            $rejects[] = [$no, 'duplicate'];
        } elseif (!in_array($op, OPS, true)) {
            $rejects[] = [$no, 'op'];
        } elseif (!$isDigits($sarg)) {
            $rejects[] = [$no, 'arg'];
        } else {
            $seen[(int) $sid] = true;
            $jobs[] = ['id' => (int) $sid, 'op' => $op, 'arg' => (int) $sarg];
        }
    }
    return [$jobs, $rejects];
}

function collatzSteps(int $n): int
{
    $steps = 0;
    while ($n > 1) {
        $n = $n % 2 === 0 ? intdiv($n, 2) : 3 * $n + 1;
        $steps++;
    }
    return $steps;
}

/** A one-way flag shared by reference: set once, read by every fiber. */
final class Cancel
{
    public bool $set = false;
}

/**
 * Cancel-aware sleep. The fiber suspends and the scheduler resumes it when the
 * deadline passes or the flag is set. Returns true if cancelled.
 */
function sleepMs(int $ms, Cancel $cancel): bool
{
    if ($cancel->set) {
        return true;
    }
    $deadline = hrtime(true) + $ms * 1_000_000;
    Fiber::suspend([$deadline, $cancel]);
    return $cancel->set;
}

/** T4 - execute one job; every wait is cancel-aware. Returns [status, detail]. */
function runJob(array $job, Cancel $cancel, array &$shared): array
{
    if ($cancel->set) {
        return ['cancelled', ''];
    }
    switch ($job['op']) {
        case 'collatz':
            $steps = collatzSteps($job['arg']);
            $shared['steps_total'] += $steps; // T5 - safe: only one fiber runs at a time
            return ['ok', "steps=$steps"];
        case 'io':
            if (sleepMs(min($job['arg'], JOB_TIMEOUT_MS), $cancel)) {
                return ['cancelled', ''];
            }
            return $job['arg'] > JOB_TIMEOUT_MS
                ? ['timeout', 'limit=' . JOB_TIMEOUT_MS]
                : ['ok', 'waited=' . $job['arg']];
        case 'batch': // long job without a timeout: only cancellation stops it
            return sleepMs($job['arg'], $cancel) ? ['cancelled', ''] : ['ok', 'waited=' . $job['arg']];
        default: // poison: wait, then cancel everybody (T6)
            if (sleepMs($job['arg'], $cancel)) {
                return ['cancelled', ''];
            }
            $cancel->set = true;
            return ['fatal', ''];
    }
}

/** T2, T3, T7 - queue the jobs, start W fibers, run the event loop until all are done. */
function runPool(array $jobs): array
{
    $cancel = new Cancel();
    $shared = ['steps_total' => 0, 'results' => []];
    $queue = new SplQueue();
    foreach ($jobs as $job) {
        $queue->enqueue($job);
    }

    $workers = [];
    for ($w = 0; $w < WORKERS; $w++) {
        $workers[] = new Fiber(function () use ($queue, $cancel, &$shared): void {
            while (!$queue->isEmpty()) {
                $job = $queue->dequeue();
                [$status, $detail] = runJob($job, $cancel, $shared);
                $shared['results'][] = ['job' => $job, 'status' => $status, 'detail' => $detail];
            }
        });
    }

    // the event loop: each entry is [fiber, deadline|null, cancel|null]
    $waiting = [];
    foreach ($workers as $f) {
        $waiting[] = [$f, null, null];
    }
    while ($waiting) {
        $now = hrtime(true);
        $next = [];
        $earliest = null;
        foreach ($waiting as [$f, $deadline, $c]) {
            $ready = $deadline === null || $now >= $deadline || $c->set;
            if (!$ready) {
                $next[] = [$f, $deadline, $c];
                $earliest = $earliest === null ? $deadline : min($earliest, $deadline);
                continue;
            }
            $req = $f->isStarted() ? $f->resume() : $f->start();
            if (!$f->isTerminated()) {
                $next[] = [$f, $req[0], $req[1]];
                $earliest = ($earliest === null || $req[0] < $earliest) ? $req[0] : $earliest;
                if ($req[1]->set) {
                    $earliest = 0; // already cancelled: do not sleep, loop again
                }
            }
        }
        $waiting = $next;
        if ($waiting && $earliest !== null && $earliest > hrtime(true)) {
            usleep(max(0, intdiv($earliest - hrtime(true), 1000)));
        }
    }
    usort($shared['results'], fn($a, $b) => $a['job']['id'] <=> $b['job']['id']);
    return [$shared['results'], $shared['steps_total']];
}

/** T8 - stable, diff-able output; the exit code is the contract. */
function report(array $results, int $stepsTotal, int $rejected): int
{
    $count = ['ok' => 0, 'timeout' => 0, 'fatal' => 0, 'cancelled' => 0];
    foreach ($results as $o) {
        $count[$o['status']]++;
        $tail = $o['detail'] !== '' ? ' ' . $o['detail'] : '';
        echo "job {$o['job']['id']} {$o['job']['op']} {$o['status']}{$tail}\n";
    }
    echo 'jobs=' . count($results) . " ok={$count['ok']} timeout={$count['timeout']} "
        . "fatal={$count['fatal']} cancelled={$count['cancelled']} rejected=$rejected\n";
    echo "steps_total=$stepsTotal workers=" . WORKERS . "\n";
    return ($count['ok'] === count($results) && $rejected === 0) ? 0 : 1;
}

function main(array $argv): int
{
    if (count($argv) !== 2) {
        fwrite(STDERR, "usage: main.php jobs.csv\n");
        return 2;
    }
    $text = @file_get_contents($argv[1]);
    if ($text === false) {
        fwrite(STDERR, "error: cannot read {$argv[1]}\n");
        return 2;
    }
    $lines = explode("\n", rtrim(str_replace("\r\n", "\n", $text), "\n"));
    [$jobs, $rejects] = parseJobs($lines);
    foreach ($rejects as [$no, $reason]) {
        fwrite(STDERR, "reject line $no: $reason\n");
    }
    [$results, $stepsTotal] = runPool($jobs);
    return report($results, $stepsTotal, count($rejects));
}

exit(main($argv));
```

### Rust (`main.rs`) — `rustc --edition 2024 -O -o main main.rs && ./main jobs.csv`

```rust
//! Job runner: parse jobs, run them on a worker pool, cancel everything on a fatal job.
use std::collections::HashSet;
use std::process::ExitCode;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::mpsc;
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

const WORKERS: usize = 4;
const JOB_TIMEOUT_MS: u64 = 200;

#[derive(Debug, Clone)]
struct Job {
    id: u64,
    op: String,
    arg: u64,
}

#[derive(Debug)]
struct Outcome {
    job: Job,
    status: &'static str, // ok | timeout | fatal | cancelled
    detail: String,
}

/// T1 - validate every line; return the jobs and the rejects as (line_no, reason).
fn parse_jobs(lines: &[&str]) -> (Vec<Job>, Vec<(usize, &'static str)>) {
    let mut jobs = Vec::new();
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
        let digits = |s: &str| !s.is_empty() && s.bytes().all(|b| b.is_ascii_digit());
        let id = if digits(parts[0]) { parts[0].parse::<u64>().ok() } else { None };
        match id {
            None | Some(0) => rejects.push((no, "id")),
            Some(id) if seen.contains(&id) => rejects.push((no, "duplicate")),
            Some(id) => {
                if !["collatz", "io", "batch", "poison"].contains(&parts[1]) {
                    rejects.push((no, "op"));
                } else if !digits(parts[2]) {
                    rejects.push((no, "arg"));
                } else if let Ok(arg) = parts[2].parse::<u64>() {
                    seen.insert(id);
                    jobs.push(Job { id, op: parts[1].to_string(), arg });
                } else {
                    rejects.push((no, "arg")); // digits but does not fit in u64
                }
            }
        }
    }
    (jobs, rejects)
}

fn collatz_steps(mut n: u64) -> u64 {
    let mut steps = 0;
    while n > 1 {
        n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
        steps += 1;
    }
    steps
}

/// A one-way flag with a timed wait: set once, wakes every waiter.
struct Cancel {
    flag: Mutex<bool>,
    cv: Condvar,
}

impl Cancel {
    fn new() -> Self {
        Cancel { flag: Mutex::new(false), cv: Condvar::new() }
    }
    fn is_set(&self) -> bool {
        *self.flag.lock().unwrap()
    }
    fn set(&self) {
        *self.flag.lock().unwrap() = true;
        self.cv.notify_all(); // T6 - wake all the sleepers
    }
    /// Wait up to `d`; returns true if the flag was set (before or during the wait).
    fn wait(&self, d: Duration) -> bool {
        let guard = self.flag.lock().unwrap();
        let (guard, _) = self.cv.wait_timeout_while(guard, d, |set| !*set).unwrap();
        *guard
    }
}

/// T4 - execute one job; every wait is cancel-aware.
fn run_job(job: &Job, cancel: &Cancel, steps_total: &AtomicU64) -> (&'static str, String) {
    if cancel.is_set() {
        return ("cancelled", String::new());
    }
    match job.op.as_str() {
        "collatz" => {
            let steps = collatz_steps(job.arg);
            steps_total.fetch_add(steps, Ordering::Relaxed); // T5
            ("ok", format!("steps={steps}"))
        }
        "io" => {
            let wait_ms = job.arg.min(JOB_TIMEOUT_MS);
            if cancel.wait(Duration::from_millis(wait_ms)) {
                ("cancelled", String::new())
            } else if job.arg > JOB_TIMEOUT_MS {
                ("timeout", format!("limit={JOB_TIMEOUT_MS}"))
            } else {
                ("ok", format!("waited={}", job.arg))
            }
        }
        "batch" => {
            // long job without a timeout: only cancellation stops it
            if cancel.wait(Duration::from_millis(job.arg)) {
                ("cancelled", String::new())
            } else {
                ("ok", format!("waited={}", job.arg))
            }
        }
        _ => {
            // poison: wait, then cancel everybody (T6)
            if cancel.wait(Duration::from_millis(job.arg)) {
                ("cancelled", String::new())
            } else {
                cancel.set();
                ("fatal", String::new())
            }
        }
    }
}

/// T2, T3, T7 - queue the jobs, start W threads, collect the outcomes.
fn run_pool(jobs: Vec<Job>) -> (Vec<Outcome>, u64) {
    let cancel = Arc::new(Cancel::new());
    let steps_total = Arc::new(AtomicU64::new(0));

    let (job_tx, job_rx) = mpsc::channel::<Job>();
    let job_rx = Arc::new(Mutex::new(job_rx)); // Receiver is not Sync: share it behind a Mutex
    let (out_tx, out_rx) = mpsc::channel::<Outcome>();

    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx); // closing the channel is the "no more work" signal

    let mut handles = Vec::new();
    for _ in 0..WORKERS {
        let (job_rx, out_tx) = (Arc::clone(&job_rx), out_tx.clone());
        let (cancel, steps_total) = (Arc::clone(&cancel), Arc::clone(&steps_total));
        handles.push(thread::spawn(move || {
            loop {
                let next = job_rx.lock().unwrap().recv(); // lock released at the end of the statement
                let Ok(job) = next else { return };
                let (status, detail) = run_job(&job, &cancel, &steps_total);
                out_tx.send(Outcome { job, status, detail }).unwrap();
            }
        }));
    }
    drop(out_tx); // the main thread keeps no sender: the loop below ends when workers exit
    let mut results: Vec<Outcome> = out_rx.iter().collect();
    for h in handles {
        h.join().unwrap();
    }
    results.sort_by_key(|o| o.job.id);
    (results, steps_total.load(Ordering::Relaxed))
}

/// T8 - stable, diff-able output; the exit code is the contract.
fn report(results: &[Outcome], steps_total: u64, rejected: usize) -> u8 {
    let count = |s: &str| results.iter().filter(|o| o.status == s).count();
    for o in results {
        let tail = if o.detail.is_empty() { String::new() } else { format!(" {}", o.detail) };
        println!("job {} {} {}{}", o.job.id, o.job.op, o.status, tail);
    }
    println!(
        "jobs={} ok={} timeout={} fatal={} cancelled={} rejected={}",
        results.len(), count("ok"), count("timeout"), count("fatal"), count("cancelled"), rejected
    );
    println!("steps_total={steps_total} workers={WORKERS}");
    if count("ok") == results.len() && rejected == 0 { 0 } else { 1 }
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("usage: main jobs.csv");
        return ExitCode::from(2);
    }
    let Ok(text) = std::fs::read_to_string(&args[1]) else {
        eprintln!("error: cannot read {}", args[1]);
        return ExitCode::from(2);
    };
    let lines: Vec<&str> = text.lines().collect(); // lines() also strips a trailing \r
    let (jobs, rejects) = parse_jobs(&lines);
    for (no, reason) in &rejects {
        eprintln!("reject line {no}: {reason}");
    }
    let (results, steps_total) = run_pool(jobs);
    ExitCode::from(report(&results, steps_total, rejects.len()))
}
```

### Go (`main.go`, con `go.mod` che dichiara `go 1.24`) — `go build -o main . && ./main jobs.csv`

```go
// Job runner: parse jobs, run them on a worker pool, cancel everything on a fatal job.
package main

import (
	"context"
	"errors"
	"fmt"
	"os"
	"sort"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

const (
	workers      = 4
	jobTimeoutMS = 200
)

type Job struct {
	ID  int
	Op  string
	Arg int
}

type Outcome struct {
	Job    Job
	Status string // ok | timeout | fatal | cancelled
	Detail string
}

type Reject struct {
	Line   int
	Reason string
}

// parseJobs is T1: validate every line, collect rejects with their reason.
func parseJobs(lines []string) ([]Job, []Reject) {
	var jobs []Job
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
		sid, op, sarg := strings.TrimSpace(parts[0]), strings.TrimSpace(parts[1]), strings.TrimSpace(parts[2])
		id, idErr := parseUint(sid)
		arg, argErr := parseUint(sarg)
		switch {
		case idErr != nil || id == 0:
			rejects = append(rejects, Reject{no, "id"})
		case seen[id]:
			rejects = append(rejects, Reject{no, "duplicate"})
		case op != "collatz" && op != "io" && op != "batch" && op != "poison":
			rejects = append(rejects, Reject{no, "op"})
		case argErr != nil:
			rejects = append(rejects, Reject{no, "arg"})
		default:
			seen[id] = true
			jobs = append(jobs, Job{id, op, arg})
		}
	}
	return jobs, rejects
}

// parseUint accepts ASCII digits only (no sign, no underscores).
func parseUint(s string) (int, error) {
	if s == "" {
		return 0, errors.New("empty")
	}
	for _, r := range s {
		if r < '0' || r > '9' {
			return 0, errors.New("not a digit")
		}
	}
	return strconv.Atoi(s)
}

func collatzSteps(n uint64) uint64 {
	var steps uint64
	for n > 1 {
		if n%2 == 0 {
			n /= 2
		} else {
			n = 3*n + 1
		}
		steps++
	}
	return steps
}

// Shared is the state touched by all workers.
type Shared struct {
	cancel     context.CancelFunc // T6 - cancels the context every worker watches
	stepsTotal atomic.Uint64      // T5 - lock-free accumulator
	mu         sync.Mutex         // protects results
	results    []Outcome
}

func (s *Shared) record(o Outcome) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.results = append(s.results, o)
}

// wait blocks for d or until ctx is done; it reports whether ctx ended first.
func wait(ctx context.Context, d time.Duration) bool {
	t := time.NewTimer(d)
	defer t.Stop()
	select {
	case <-ctx.Done():
		return true
	case <-t.C:
		return false
	}
}

// runJob is T4: execute one job; every wait is cancel-aware.
func runJob(ctx context.Context, j Job, sh *Shared) Outcome {
	if ctx.Err() != nil {
		return Outcome{j, "cancelled", ""}
	}
	switch j.Op {
	case "collatz":
		steps := collatzSteps(uint64(j.Arg))
		sh.stepsTotal.Add(steps)
		return Outcome{j, "ok", fmt.Sprintf("steps=%d", steps)}
	case "io":
		// the per-job deadline is a child context of the pool context
		jctx, stop := context.WithTimeout(ctx, jobTimeoutMS*time.Millisecond)
		defer stop()
		if wait(jctx, time.Duration(j.Arg)*time.Millisecond) {
			if ctx.Err() != nil {
				return Outcome{j, "cancelled", ""}
			}
			return Outcome{j, "timeout", fmt.Sprintf("limit=%d", jobTimeoutMS)}
		}
		return Outcome{j, "ok", fmt.Sprintf("waited=%d", j.Arg)}
	case "batch": // long job without a timeout: only cancellation stops it
		if wait(ctx, time.Duration(j.Arg)*time.Millisecond) {
			return Outcome{j, "cancelled", ""}
		}
		return Outcome{j, "ok", fmt.Sprintf("waited=%d", j.Arg)}
	}
	// poison: wait, then cancel everybody (T6)
	if wait(ctx, time.Duration(j.Arg)*time.Millisecond) {
		return Outcome{j, "cancelled", ""}
	}
	sh.cancel()
	return Outcome{j, "fatal", ""}
}

// runPool is T2, T3, T7: queue the jobs, start the workers, wait for them.
func runPool(jobs []Job) ([]Outcome, uint64) {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	sh := &Shared{cancel: cancel}

	queue := make(chan Job, len(jobs))
	for _, j := range jobs {
		queue <- j
	}
	close(queue) // workers stop when the queue is drained

	var wg sync.WaitGroup
	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range queue {
				sh.record(runJob(ctx, j, sh))
			}
		}()
	}
	wg.Wait()

	sort.Slice(sh.results, func(a, b int) bool { return sh.results[a].Job.ID < sh.results[b].Job.ID })
	return sh.results, sh.stepsTotal.Load()
}

// report is T8: stable, diff-able output; the exit code is the contract.
func report(results []Outcome, stepsTotal uint64, rejected int) int {
	count := map[string]int{}
	for _, o := range results {
		count[o.Status]++
		tail := ""
		if o.Detail != "" {
			tail = " " + o.Detail
		}
		fmt.Printf("job %d %s %s%s\n", o.Job.ID, o.Job.Op, o.Status, tail)
	}
	fmt.Printf("jobs=%d ok=%d timeout=%d fatal=%d cancelled=%d rejected=%d\n",
		len(results), count["ok"], count["timeout"], count["fatal"], count["cancelled"], rejected)
	fmt.Printf("steps_total=%d workers=%d\n", stepsTotal, workers)
	if count["ok"] == len(results) && rejected == 0 {
		return 0
	}
	return 1
}

func main() { os.Exit(run(os.Args)) }

func run(args []string) int {
	if len(args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: main jobs.csv")
		return 2
	}
	data, err := os.ReadFile(args[1])
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: cannot read %s\n", args[1])
		return 2
	}
	text := strings.ReplaceAll(string(data), "\r\n", "\n")
	lines := strings.Split(strings.TrimSuffix(text, "\n"), "\n")
	jobs, rejects := parseJobs(lines)
	for _, r := range rejects {
		fmt.Fprintf(os.Stderr, "reject line %d: %s\n", r.Line, r.Reason)
	}
	results, steps := runPool(jobs)
	return report(results, steps, len(rejects))
}
```
