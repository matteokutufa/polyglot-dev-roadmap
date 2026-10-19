> File C10-parte7.md (parte 7 di 7) — Capitolo C10, Modelli di memoria

## Appendice — Il programma completo

### Python (`main.py`) — `python3 main.py ops.txt`

```python
import sys
import weakref


class Node:
    """T2 - a file or a directory. Children are OWNED (strong references); the parent link is WEAK."""

    __slots__ = ("name", "size", "is_dir", "children", "parent", "__weakref__")

    def __init__(self, name: str, is_dir: bool, size: int = 0, parent: "Node | None" = None) -> None:
        self.name = name
        self.is_dir = is_dir
        self.size = size
        self.children: dict[str, Node] = {}
        self.parent: weakref.ref[Node] | None = weakref.ref(parent) if parent is not None else None


def split_path(path: str) -> list[str] | None:
    """T1 - '/a/b' -> ['a', 'b'], '/' -> []; None if the path is not valid."""
    if not path.startswith("/"):
        return None
    if path == "/":
        return []
    parts = path[1:].split("/")
    if any(p in ("", ".", "..") for p in parts):
        return None
    return parts


def lookup(root: Node, parts: list[str]) -> Node | None:
    node = root
    for p in parts:
        nxt = node.children.get(p)
        if nxt is None:
            return None
        node = nxt
    return node


def full_path(node: Node) -> str:
    """T6 - walk the parent links up to the root."""
    names: list[str] = []
    cur: Node | None = node
    while cur is not None and cur.parent is not None:
        names.append(cur.name)
        cur = cur.parent()
    return "/" + "/".join(reversed(names))


def du(node: Node) -> tuple[int, int]:
    """T4 - (total size, number of nodes) of a subtree, by recursion."""
    size, count = node.size, 1
    for child in node.children.values():
        s, c = du(child)
        size += s
        count += c
    return size, count


def deep_copy(src: Node, parent: Node | None = None) -> Node:
    """T5 - an independent copy of a subtree: every node is new, every parent link is re-created."""
    copy = Node(src.name, src.is_dir, src.size, parent)
    for name, child in src.children.items():
        copy.children[name] = deep_copy(child, copy)
    return copy


class Drive:
    def __init__(self) -> None:
        self.root = Node("", True)
        self.snapshots: dict[str, Node] = {}
        self.bookmarks: dict[str, weakref.ref[Node]] = {}

    def create(self, path: str, is_dir: bool, size: int) -> str | None:
        """T3 - mkdir and add share the same checks; returns the reason of a refusal, or None."""
        parts = split_path(path)
        if parts is None:
            return "path"
        if not parts:
            return "exists"
        parent = lookup(self.root, parts[:-1])
        if parent is None:
            return "no_parent"
        if not parent.is_dir:
            return "not_dir"
        if parts[-1] in parent.children:
            return "exists"
        parent.children[parts[-1]] = Node(parts[-1], is_dir, size, parent)
        return None

    def remove(self, path: str) -> str | None:
        """T3 - remove a file or a whole subtree."""
        parts = split_path(path)
        if parts is None:
            return "path"
        if not parts:
            return "root"
        parent = lookup(self.root, parts[:-1])
        if parent is None or parts[-1] not in parent.children:
            return "no_such"
        del parent.children[parts[-1]]  # the subtree loses its owner: it is freed right here
        return None

    def bookmark(self, name: str, path: str) -> str | None:
        """T6 - a bookmark is a WEAK handle: it finds the node again without keeping it alive."""
        parts = split_path(path)
        if parts is None:
            return "path"
        node = lookup(self.root, parts)
        if node is None:
            return "no_such"
        self.bookmarks[name] = weakref.ref(node)  # a weak handle: it does not keep the node alive
        return None

    def show(self, name: str) -> str | None:
        ref = self.bookmarks.get(name)
        if ref is None:
            return "no_bookmark"
        node = ref()
        print(f"show {name} alive path={full_path(node)}" if node is not None else f"show {name} gone")
        return None


def parse_size(raw: str) -> int | None:
    return int(raw) if raw.isascii() and raw.isdigit() and len(raw) <= 12 else None


def execute(d: Drive, tokens: list[str]) -> str | None:
    """T1 - one command; returns the reason of a refusal, or None."""
    match tokens:
        case ["mkdir", path]:
            return d.create(path, True, 0)
        case ["add", path, raw]:
            if split_path(path) is None:
                return "path"
            size = parse_size(raw)
            return "size" if size is None else d.create(path, False, size)
        case ["rm", path]:
            return d.remove(path)
        case ["du", path]:
            parts = split_path(path)
            if parts is None:
                return "path"
            node = lookup(d.root, parts)
            if node is None:
                return "no_such"
            size, count = du(node)
            print(f"du {path} size={size} nodes={count}")
            return None
        case ["snapshot", name]:
            if name in d.snapshots:
                return "dup_snapshot"
            d.snapshots[name] = deep_copy(d.root)
            return None
        case ["rollback", name]:
            snap = d.snapshots.get(name)
            if snap is None:
                return "no_snapshot"
            d.root = deep_copy(snap)  # the old tree loses its owner: it is freed right here
            return None
        case ["bookmark", name, path]:
            return d.bookmark(name, path)
        case ["show", name]:
            return d.show(name)
        case _:
            return "syntax"


def run(path: str) -> int:
    d = Drive()
    ops = applied = rejected = 0
    with open(path, encoding="utf-8") as f:
        for n, line in enumerate(f, start=1):
            line = line.strip()
            if not line or line.startswith("#"):
                continue
            ops += 1
            tokens = line.split()
            reason = execute(d, tokens)
            if reason is None:
                applied += 1
            else:
                rejected += 1
                print(f"reject line={n} op={tokens[0]} reason={reason}", file=sys.stderr)
    # T7 - the summary and the exit code
    alive = sum(1 for r in d.bookmarks.values() if r() is not None)
    print(f"ops={ops} applied={applied} rejected={rejected} nodes={du(d.root)[1]} "
          f"snapshots={len(d.snapshots)} bookmarks_alive={alive} bookmarks_gone={len(d.bookmarks) - alive}")
    return 1 if rejected else 0


def main() -> int:
    if len(sys.argv) != 2:
        print("usage: main.py OPS", file=sys.stderr)
        return 2
    try:
        return run(sys.argv[1])
    except FileNotFoundError as e:
        print(f"cannot open {e.filename}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

### PHP (`main.php`) — `php main.php ops.txt`

```php
<?php
declare(strict_types=1);

/** T2 - a file or a directory. Children are OWNED (array of strong references); the parent link is WEAK. */
final class Node
{
    /** @var array<string, Node> */
    public array $children = [];
    public ?WeakReference $parent;

    public function __construct(
        public readonly string $name,
        public readonly bool $isDir,
        public readonly int $size = 0,
        ?Node $parent = null,
    ) {
        $this->parent = $parent !== null ? WeakReference::create($parent) : null;
    }
}

/** T1 - '/a/b' -> ['a', 'b'], '/' -> []; null if the path is not valid. */
function splitPath(string $path): ?array
{
    if (!str_starts_with($path, '/')) {
        return null;
    }
    if ($path === '/') {
        return [];
    }
    $parts = explode('/', substr($path, 1));
    foreach ($parts as $p) {
        if ($p === '' || $p === '.' || $p === '..') {
            return null;
        }
    }
    return $parts;
}

function lookup(Node $root, array $parts): ?Node
{
    $node = $root;
    foreach ($parts as $p) {
        $node = $node->children[$p] ?? null;
        if ($node === null) {
            return null;
        }
    }
    return $node;
}

/** T6 - walk the parent links up to the root. */
function fullPath(Node $node): string
{
    $names = [];
    $cur = $node;
    while ($cur !== null && $cur->parent !== null) {
        $names[] = $cur->name;
        $cur = $cur->parent->get();
    }
    return '/' . implode('/', array_reverse($names));
}

/** T4 - [total size, number of nodes] of a subtree, by recursion. */
function du(Node $node): array
{
    $size = $node->size;
    $count = 1;
    foreach ($node->children as $child) {
        [$s, $c] = du($child);
        $size += $s;
        $count += $c;
    }
    return [$size, $count];
}

/** T5 - an independent copy of a subtree: every node is new, every parent link is re-created. */
function deepCopy(Node $src, ?Node $parent = null): Node
{
    $copy = new Node($src->name, $src->isDir, $src->size, $parent);
    foreach ($src->children as $name => $child) {
        $copy->children[$name] = deepCopy($child, $copy);
    }
    return $copy;
}

final class Drive
{
    public Node $root;
    /** @var array<string, Node> */
    public array $snapshots = [];
    /** @var array<string, WeakReference<Node>> */
    public array $bookmarks = [];

    public function __construct()
    {
        $this->root = new Node('', true);
    }

    /** T3 - mkdir and add share the same checks; returns the reason of a refusal, or null. */
    public function create(string $path, bool $isDir, int $size): ?string
    {
        $parts = splitPath($path);
        if ($parts === null) {
            return 'path';
        }
        if ($parts === []) {
            return 'exists';
        }
        $name = $parts[count($parts) - 1];
        $parent = lookup($this->root, array_slice($parts, 0, -1));
        if ($parent === null) {
            return 'no_parent';
        }
        if (!$parent->isDir) {
            return 'not_dir';
        }
        if (isset($parent->children[$name])) {
            return 'exists';
        }
        $parent->children[$name] = new Node($name, $isDir, $size, $parent);
        return null;
    }

    /** T3 - remove a file or a whole subtree. */
    public function remove(string $path): ?string
    {
        $parts = splitPath($path);
        if ($parts === null) {
            return 'path';
        }
        if ($parts === []) {
            return 'root';
        }
        $name = $parts[count($parts) - 1];
        $parent = lookup($this->root, array_slice($parts, 0, -1));
        if ($parent === null || !isset($parent->children[$name])) {
            return 'no_such';
        }
        unset($parent->children[$name]); // the subtree loses its owner: it is freed right here
        return null;
    }

    /** T6 - a bookmark is a WEAK handle: it finds the node again without keeping it alive. */
    public function bookmark(string $name, string $path): ?string
    {
        $parts = splitPath($path);
        if ($parts === null) {
            return 'path';
        }
        $node = lookup($this->root, $parts);
        if ($node === null) {
            return 'no_such';
        }
        $this->bookmarks[$name] = WeakReference::create($node); // a weak handle: it does not keep the node alive
        return null;
    }

    public function show(string $name): ?string
    {
        $ref = $this->bookmarks[$name] ?? null;
        if ($ref === null) {
            return 'no_bookmark';
        }
        $node = $ref->get();
        echo $node !== null ? "show $name alive path=" . fullPath($node) . "\n" : "show $name gone\n";
        return null;
    }
}

function parseSize(string $raw): ?int
{
    return ctype_digit($raw) && strlen($raw) <= 12 ? (int) $raw : null;
}

/** T1 - one command; returns the reason of a refusal, or null. */
function execute(Drive $d, array $t): ?string
{
    $op = $t[0];
    $argc = count($t) - 1;
    if ($op === 'mkdir' && $argc === 1) {
        return $d->create($t[1], true, 0);
    }
    if ($op === 'add' && $argc === 2) {
        if (splitPath($t[1]) === null) {
            return 'path';
        }
        $size = parseSize($t[2]);
        return $size === null ? 'size' : $d->create($t[1], false, $size);
    }
    if ($op === 'rm' && $argc === 1) {
        return $d->remove($t[1]);
    }
    if ($op === 'du' && $argc === 1) {
        $parts = splitPath($t[1]);
        if ($parts === null) {
            return 'path';
        }
        $node = lookup($d->root, $parts);
        if ($node === null) {
            return 'no_such';
        }
        [$size, $count] = du($node);
        echo "du {$t[1]} size=$size nodes=$count\n";
        return null;
    }
    if ($op === 'snapshot' && $argc === 1) {
        if (isset($d->snapshots[$t[1]])) {
            return 'dup_snapshot';
        }
        $d->snapshots[$t[1]] = deepCopy($d->root);
        return null;
    }
    if ($op === 'rollback' && $argc === 1) {
        $snap = $d->snapshots[$t[1]] ?? null;
        if ($snap === null) {
            return 'no_snapshot';
        }
        $d->root = deepCopy($snap); // the old tree loses its owner: it is freed right here
        return null;
    }
    if ($op === 'bookmark' && $argc === 2) {
        return $d->bookmark($t[1], $t[2]);
    }
    if ($op === 'show' && $argc === 1) {
        return $d->show($t[1]);
    }
    return 'syntax';
}

function run(string $path): int
{
    $f = @fopen($path, 'rb');
    if ($f === false) {
        throw new RuntimeException("cannot open $path");
    }
    $d = new Drive();
    $ops = $applied = $rejected = 0;
    $n = 0;
    while (($line = fgets($f)) !== false) {
        $n++;
        $line = trim($line);
        if ($line === '' || $line[0] === '#') {
            continue;
        }
        $ops++;
        $tokens = preg_split('/\s+/', $line);
        $reason = execute($d, $tokens);
        if ($reason === null) {
            $applied++;
        } else {
            $rejected++;
            fwrite(STDERR, "reject line=$n op={$tokens[0]} reason=$reason\n");
        }
    }
    fclose($f);
    // T7 - the summary and the exit code
    $alive = 0;
    foreach ($d->bookmarks as $ref) {
        if ($ref->get() !== null) {
            $alive++;
        }
    }
    $nodes = du($d->root)[1];
    $gone = count($d->bookmarks) - $alive;
    echo "ops=$ops applied=$applied rejected=$rejected nodes=$nodes snapshots=" . count($d->snapshots)
        . " bookmarks_alive=$alive bookmarks_gone=$gone\n";
    return $rejected > 0 ? 1 : 0;
}

if ($argc !== 2) {
    fwrite(STDERR, "usage: main.php OPS\n");
    exit(2);
}
try {
    exit(run($argv[1]));
} catch (RuntimeException $e) {
    fwrite(STDERR, $e->getMessage() . "\n");
    exit(2);
}
```

### Rust (`main.rs`) — `rustc --edition 2024 -O main.rs -o main_rs && ./main_rs ops.txt`

```rust
use std::cell::RefCell;
use std::collections::{BTreeMap, HashMap};
use std::fs;
use std::process::ExitCode;
use std::rc::{Rc, Weak};

type Link = Rc<RefCell<Node>>;

/// T2 - a file or a directory. Children are OWNED (Rc, strong); the parent link is WEAK.
struct Node {
    name: String,
    size: u64,
    is_dir: bool,
    children: BTreeMap<String, Link>,
    parent: Weak<RefCell<Node>>,
}

fn new_node(name: &str, is_dir: bool, size: u64, parent: Weak<RefCell<Node>>) -> Link {
    Rc::new(RefCell::new(Node { name: name.to_string(), size, is_dir, children: BTreeMap::new(), parent }))
}

/// T1 - "/a/b" -> ["a", "b"], "/" -> []; None if the path is not valid.
fn split_path(path: &str) -> Option<Vec<&str>> {
    let rest = path.strip_prefix('/')?;
    if rest.is_empty() {
        return Some(vec![]);
    }
    let parts: Vec<&str> = rest.split('/').collect();
    if parts.iter().any(|p| matches!(*p, "" | "." | "..")) {
        return None;
    }
    Some(parts)
}

fn lookup(root: &Link, parts: &[&str]) -> Option<Link> {
    let mut node = Rc::clone(root);
    for p in parts {
        let next = node.borrow().children.get(*p).cloned()?;
        node = next;
    }
    Some(node)
}

/// T6 - walk the parent links up to the root.
fn full_path(node: &Link) -> String {
    let mut names: Vec<String> = Vec::new();
    let mut cur = Rc::clone(node);
    loop {
        let parent = cur.borrow().parent.upgrade();
        match parent {
            Some(p) => {
                names.push(cur.borrow().name.clone());
                cur = p;
            }
            None => break,
        }
    }
    names.reverse();
    format!("/{}", names.join("/"))
}

/// T4 - (total size, number of nodes) of a subtree, by recursion.
fn du(node: &Link) -> (u64, u64) {
    let n = node.borrow();
    let (mut size, mut count) = (n.size, 1);
    for child in n.children.values() {
        let (s, c) = du(child);
        size += s;
        count += c;
    }
    (size, count)
}

/// T5 - an independent copy of a subtree: every node is new, every parent link is re-created.
fn deep_copy(src: &Link, parent: Weak<RefCell<Node>>) -> Link {
    let s = src.borrow();
    let copy = new_node(&s.name, s.is_dir, s.size, parent);
    for (name, child) in &s.children {
        let c = deep_copy(child, Rc::downgrade(&copy));
        copy.borrow_mut().children.insert(name.clone(), c);
    }
    copy
}

struct Drive {
    root: Link,
    snapshots: HashMap<String, Link>,
    bookmarks: HashMap<String, Weak<RefCell<Node>>>,
}

impl Drive {
    fn new() -> Self {
        Drive { root: new_node("", true, 0, Weak::new()), snapshots: HashMap::new(), bookmarks: HashMap::new() }
    }

    /// T3 - mkdir and add share the same checks; returns the reason of a refusal, or None.
    fn create(&mut self, path: &str, is_dir: bool, size: u64) -> Option<&'static str> {
        let Some(parts) = split_path(path) else { return Some("path") };
        let Some((name, dir)) = parts.split_last() else { return Some("exists") };
        let Some(parent) = lookup(&self.root, dir) else { return Some("no_parent") };
        if !parent.borrow().is_dir {
            return Some("not_dir");
        }
        if parent.borrow().children.contains_key(*name) {
            return Some("exists");
        }
        let node = new_node(name, is_dir, size, Rc::downgrade(&parent));
        parent.borrow_mut().children.insert(name.to_string(), node);
        None
    }

    /// T3 - remove a file or a whole subtree.
    fn remove(&mut self, path: &str) -> Option<&'static str> {
        let Some(parts) = split_path(path) else { return Some("path") };
        let Some((name, dir)) = parts.split_last() else { return Some("root") };
        let Some(parent) = lookup(&self.root, dir) else { return Some("no_such") };
        // `remove` hands the owner back; dropping it frees the whole subtree right here
        let removed = parent.borrow_mut().children.remove(*name);
        if removed.is_none() { Some("no_such") } else { None }
    }

    /// T6 - a bookmark is a WEAK handle: it finds the node again without keeping it alive.
    fn bookmark(&mut self, name: &str, path: &str) -> Option<&'static str> {
        let Some(parts) = split_path(path) else { return Some("path") };
        let Some(node) = lookup(&self.root, &parts) else { return Some("no_such") };
        self.bookmarks.insert(name.to_string(), Rc::downgrade(&node)); // a weak handle: it does not keep the node alive
        None
    }

    fn show(&self, name: &str) -> Option<&'static str> {
        let Some(weak) = self.bookmarks.get(name) else { return Some("no_bookmark") };
        match weak.upgrade() {
            Some(node) => println!("show {name} alive path={}", full_path(&node)),
            None => println!("show {name} gone"),
        }
        None
    }
}

fn parse_size(raw: &str) -> Option<u64> {
    if !raw.is_empty() && raw.len() <= 12 && raw.bytes().all(|b| b.is_ascii_digit()) { raw.parse().ok() } else { None }
}

/// T1 - one command; returns the reason of a refusal, or None.
fn execute(d: &mut Drive, t: &[&str]) -> Option<&'static str> {
    match t {
        ["mkdir", path] => d.create(path, true, 0),
        ["add", path, raw] => {
            if split_path(path).is_none() {
                return Some("path");
            }
            match parse_size(raw) {
                None => Some("size"),
                Some(size) => d.create(path, false, size),
            }
        }
        ["rm", path] => d.remove(path),
        ["du", path] => {
            let Some(parts) = split_path(path) else { return Some("path") };
            let Some(node) = lookup(&d.root, &parts) else { return Some("no_such") };
            let (size, count) = du(&node);
            println!("du {path} size={size} nodes={count}");
            None
        }
        ["snapshot", name] => {
            if d.snapshots.contains_key(*name) {
                return Some("dup_snapshot");
            }
            d.snapshots.insert(name.to_string(), deep_copy(&d.root, Weak::new()));
            None
        }
        ["rollback", name] => {
            let Some(snap) = d.snapshots.get(*name) else { return Some("no_snapshot") };
            d.root = deep_copy(snap, Weak::new()); // the old tree loses its owner: it is freed right here
            None
        }
        ["bookmark", name, path] => d.bookmark(name, path),
        ["show", name] => d.show(name),
        _ => Some("syntax"),
    }
}

fn run(path: &str) -> Result<u8, String> {
    let text = fs::read_to_string(path).map_err(|_| path.to_string())?;
    let mut d = Drive::new();
    let (mut ops, mut applied, mut rejected) = (0, 0, 0);
    for (i, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        ops += 1;
        let tokens: Vec<&str> = line.split_whitespace().collect();
        match execute(&mut d, &tokens) {
            None => applied += 1,
            Some(reason) => {
                rejected += 1;
                eprintln!("reject line={} op={} reason={reason}", i + 1, tokens[0]);
            }
        }
    }
    // T7 - the summary and the exit code
    let alive = d.bookmarks.values().filter(|w| w.upgrade().is_some()).count();
    println!(
        "ops={ops} applied={applied} rejected={rejected} nodes={} snapshots={} bookmarks_alive={alive} bookmarks_gone={}",
        du(&d.root).1,
        d.snapshots.len(),
        d.bookmarks.len() - alive
    );
    Ok(if rejected > 0 { 1 } else { 0 })
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("usage: main OPS");
        return ExitCode::from(2);
    }
    match run(&args[1]) {
        Ok(code) => ExitCode::from(code),
        Err(path) => {
            eprintln!("cannot open {path}");
            ExitCode::from(2)
        }
    }
}
```

### Go (`main.go`, `go.mod` con `go 1.24`) — `go build -o main_go && ./main_go ops.txt`

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"runtime"
	"strconv"
	"strings"
	"weak"
)

// Node is T2 - a file or a directory. There is no "owner" in the type: the garbage collector keeps
// alive whatever is reachable. The parent link is an ordinary pointer: a cycle is not a problem for a tracing GC.
type Node struct {
	name     string
	size     int
	isDir    bool
	children map[string]*Node
	parent   *Node
}

func newNode(name string, isDir bool, size int, parent *Node) *Node {
	return &Node{name: name, size: size, isDir: isDir, children: map[string]*Node{}, parent: parent}
}

// splitPath is T1 - "/a/b" -> [a b], "/" -> []; ok is false if the path is not valid.
func splitPath(path string) (parts []string, ok bool) {
	if !strings.HasPrefix(path, "/") {
		return nil, false
	}
	if path == "/" {
		return nil, true
	}
	parts = strings.Split(path[1:], "/")
	for _, p := range parts {
		if p == "" || p == "." || p == ".." {
			return nil, false
		}
	}
	return parts, true
}

func lookup(root *Node, parts []string) *Node {
	node := root
	for _, p := range parts {
		next, ok := node.children[p]
		if !ok {
			return nil
		}
		node = next
	}
	return node
}

// fullPath is T6 - walk the parent links up to the root.
func fullPath(node *Node) string {
	var names []string
	for cur := node; cur.parent != nil; cur = cur.parent {
		names = append(names, cur.name)
	}
	for i, j := 0, len(names)-1; i < j; i, j = i+1, j-1 {
		names[i], names[j] = names[j], names[i]
	}
	return "/" + strings.Join(names, "/")
}

// du is T4 - (total size, number of nodes) of a subtree, by recursion.
func du(node *Node) (size, count int) {
	size, count = node.size, 1
	for _, child := range node.children {
		s, c := du(child)
		size += s
		count += c
	}
	return
}

// deepCopy is T5 - an independent copy of a subtree: every node is new, every parent link is re-created.
func deepCopy(src, parent *Node) *Node {
	cp := newNode(src.name, src.isDir, src.size, parent)
	for name, child := range src.children {
		cp.children[name] = deepCopy(child, cp)
	}
	return cp
}

type Drive struct {
	root      *Node
	snapshots map[string]*Node
	bookmarks map[string]weak.Pointer[Node]
}

func newDrive() *Drive {
	return &Drive{root: newNode("", true, 0, nil), snapshots: map[string]*Node{}, bookmarks: map[string]weak.Pointer[Node]{}}
}

// create is T3 - mkdir and add share the same checks; it returns the reason of a refusal, or "".
func (d *Drive) create(path string, isDir bool, size int) string {
	parts, ok := splitPath(path)
	if !ok {
		return "path"
	}
	if len(parts) == 0 {
		return "exists"
	}
	name := parts[len(parts)-1]
	parent := lookup(d.root, parts[:len(parts)-1])
	if parent == nil {
		return "no_parent"
	}
	if !parent.isDir {
		return "not_dir"
	}
	if _, dup := parent.children[name]; dup {
		return "exists"
	}
	parent.children[name] = newNode(name, isDir, size, parent)
	return ""
}

// remove is T3 - remove a file or a whole subtree.
func (d *Drive) remove(path string) string {
	parts, ok := splitPath(path)
	if !ok {
		return "path"
	}
	if len(parts) == 0 {
		return "root"
	}
	name := parts[len(parts)-1]
	parent := lookup(d.root, parts[:len(parts)-1])
	if parent == nil {
		return "no_such"
	}
	if _, found := parent.children[name]; !found {
		return "no_such"
	}
	delete(parent.children, name) // the subtree becomes unreachable: the GC will free it, but WHEN is not ours to say
	return ""
}

// bookmark is T6 - a WEAK handle: it finds the node again without keeping it alive.
func (d *Drive) bookmark(name, path string) string {
	parts, ok := splitPath(path)
	if !ok {
		return "path"
	}
	node := lookup(d.root, parts)
	if node == nil {
		return "no_such"
	}
	d.bookmarks[name] = weak.Make(node) // a weak handle: it does not keep the node alive
	return ""
}

func (d *Drive) show(name string) string {
	ptr, ok := d.bookmarks[name]
	if !ok {
		return "no_bookmark"
	}
	runtime.GC() // a weak pointer is cleared only when a collection runs: force one so that the answer is deterministic
	if node := ptr.Value(); node != nil {
		fmt.Printf("show %s alive path=%s\n", name, fullPath(node))
	} else {
		fmt.Printf("show %s gone\n", name)
	}
	return ""
}

func parseSize(raw string) (int, bool) {
	if raw == "" || len(raw) > 12 {
		return 0, false
	}
	for i := 0; i < len(raw); i++ {
		if raw[i] < '0' || raw[i] > '9' {
			return 0, false
		}
	}
	n, err := strconv.Atoi(raw)
	return n, err == nil
}

// execute is T1 - one command; it returns the reason of a refusal, or "".
func execute(d *Drive, t []string) string {
	op, argc := t[0], len(t)-1
	switch {
	case op == "mkdir" && argc == 1:
		return d.create(t[1], true, 0)
	case op == "add" && argc == 2:
		if _, ok := splitPath(t[1]); !ok {
			return "path"
		}
		size, ok := parseSize(t[2])
		if !ok {
			return "size"
		}
		return d.create(t[1], false, size)
	case op == "rm" && argc == 1:
		return d.remove(t[1])
	case op == "du" && argc == 1:
		parts, ok := splitPath(t[1])
		if !ok {
			return "path"
		}
		node := lookup(d.root, parts)
		if node == nil {
			return "no_such"
		}
		size, count := du(node)
		fmt.Printf("du %s size=%d nodes=%d\n", t[1], size, count)
		return ""
	case op == "snapshot" && argc == 1:
		if _, dup := d.snapshots[t[1]]; dup {
			return "dup_snapshot"
		}
		d.snapshots[t[1]] = deepCopy(d.root, nil)
		return ""
	case op == "rollback" && argc == 1:
		snap, ok := d.snapshots[t[1]]
		if !ok {
			return "no_snapshot"
		}
		d.root = deepCopy(snap, nil) // the old tree becomes unreachable
		return ""
	case op == "bookmark" && argc == 2:
		return d.bookmark(t[1], t[2])
	case op == "show" && argc == 1:
		return d.show(t[1])
	}
	return "syntax"
}

func run(path string) int {
	f, err := os.Open(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "cannot open %s\n", path)
		return 2
	}
	defer f.Close()
	d := newDrive()
	ops, applied, rejected := 0, 0, 0
	sc := bufio.NewScanner(f)
	for n := 1; sc.Scan(); n++ {
		line := strings.TrimSpace(sc.Text())
		if line == "" || strings.HasPrefix(line, "#") {
			continue
		}
		ops++
		tokens := strings.Fields(line)
		if reason := execute(d, tokens); reason == "" {
			applied++
		} else {
			rejected++
			fmt.Fprintf(os.Stderr, "reject line=%d op=%s reason=%s\n", n, tokens[0], reason)
		}
	}
	// T7 - the summary and the exit code
	runtime.GC()
	alive := 0
	for _, p := range d.bookmarks {
		if p.Value() != nil {
			alive++
		}
	}
	_, nodes := du(d.root)
	fmt.Printf("ops=%d applied=%d rejected=%d nodes=%d snapshots=%d bookmarks_alive=%d bookmarks_gone=%d\n",
		ops, applied, rejected, nodes, len(d.snapshots), alive, len(d.bookmarks)-alive)
	if rejected > 0 {
		return 1
	}
	return 0
}

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: main OPS")
		os.Exit(2)
	}
	os.Exit(run(os.Args[1]))
}
```
