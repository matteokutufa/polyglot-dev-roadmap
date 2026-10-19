> File C13-parte5.md (parte 5 di 5) — C13 Testing e qualità, corso di programmazione multi-linguaggio (PHP, Python, Rust, Go)

### Go — `cd go && go run . ../tickets.csv; go test -race -cover ./...`

#### `go.mod`

```go
module example.com/parking

go 1.24
```

#### `parking/core.go`

```go
// Package parking is a parking fee calculator: pure functions plus one injected dependency.
package parking

import "strings"

const (
	FreeMinutes = 15   // no charge up to and including this many minutes
	DayMinutes  = 1440 // one parking "day"
	CapHours    = 8    // daily cap = 8 hours at the zone rate
)

// RateSource says where hourly rates (in cents) come from.
type RateSource interface {
	Rate(zone string) (int, bool)
}

// StaticRates is a RateSource backed by a map.
type StaticRates map[string]int

func (s StaticRates) Rate(zone string) (int, bool) { r, ok := s[zone]; return r, ok }

// ParseTime is T2: "HH:MM" -> minutes since midnight; ok is false if malformed.
func ParseTime(text string) (minutes int, ok bool) {
	if len(text) != 5 || text[2] != ':' {
		return 0, false
	}
	digit := func(b byte) bool { return b >= '0' && b <= '9' }
	for _, i := range []int{0, 1, 3, 4} {
		if !digit(text[i]) {
			return 0, false
		}
	}
	hh := int(text[0]-'0')*10 + int(text[1]-'0')
	mm := int(text[3]-'0')*10 + int(text[4]-'0')
	if hh > 23 || mm > 59 {
		return 0, false
	}
	return hh*60 + mm, true
}

// Duration is T3: minutes parked; an exit earlier than the entry means next day.
func Duration(in, out int) int {
	if d := out - in; d >= 0 {
		return d
	} else {
		return d + DayMinutes
	}
}

// Fee is T4: fee in cents; first 15 min free, started hours, cap per day.
func Fee(minutes, rate int) int {
	cap := CapHours * rate
	days, rem := minutes/DayMinutes, minutes%DayMinutes
	part := 0
	if rem > FreeMinutes {
		part = rate * ((rem + 59) / 60)
	}
	return days*cap + min(cap, part)
}

// Ticket is a validated parking ticket.
type Ticket struct {
	ID, Zone       string
	Minutes, Cents int
}

// ParseLine is T5: one CSV line -> Ticket, or the reject reason ("" on success).
func ParseLine(line string, rates RateSource) (Ticket, string) {
	f := strings.Split(line, ",")
	for i := range f {
		f[i] = strings.TrimSpace(f[i])
	}
	if len(f) != 4 || f[0] == "" || f[1] == "" {
		return Ticket{}, "fields"
	}
	in, ok1 := ParseTime(f[2])
	out, ok2 := ParseTime(f[3])
	if !ok1 || !ok2 {
		return Ticket{}, "time"
	}
	rate, ok := rates.Rate(f[1])
	if !ok {
		return Ticket{}, "zone"
	}
	m := Duration(in, out)
	return Ticket{f[0], f[1], m, Fee(m, rate)}, ""
}

// Reject is a rejected input line.
type Reject struct {
	Line   int
	Reason string
}

// Report is the outcome of Run.
type Report struct {
	Tickets int
	Valid   []Ticket
	Rejects []Reject
}

// Run is T6: process every non-empty line; ids must be unique.
func Run(lines []string, rates RateSource) Report {
	var rep Report
	seen := map[string]bool{}
	for i, raw := range lines {
		line := strings.TrimSpace(raw)
		if line == "" {
			continue
		}
		rep.Tickets++
		t, reason := ParseLine(line, rates)
		switch {
		case reason != "":
			rep.Rejects = append(rep.Rejects, Reject{i + 1, reason})
		case seen[t.ID]:
			rep.Rejects = append(rep.Rejects, Reject{i + 1, "duplicate"})
		default:
			seen[t.ID] = true
			rep.Valid = append(rep.Valid, t)
		}
	}
	return rep
}
```

#### `main.go`

```go
package main

import (
	"fmt"
	"os"
	"sort"
	"strings"

	"example.com/parking/parking"
)

// T7 - CLI: report on stdout, rejects on stderr, exit code as contract.
func main() {
	data, err := os.ReadFile(os.Args[1])
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: cannot read %s\n", os.Args[1])
		os.Exit(2)
	}
	text := strings.ReplaceAll(string(data), "\r\n", "\n")
	rep := parking.Run(strings.Split(text, "\n"), parking.StaticRates{"A": 120, "B": 80, "C": 200})
	for _, r := range rep.Rejects {
		fmt.Fprintf(os.Stderr, "line %d: rejected (%s)\n", r.Line, r.Reason)
	}
	total, count, sums := 0, map[string]int{}, map[string]int{}
	for _, t := range rep.Valid {
		total += t.Cents
		count[t.Zone]++
		sums[t.Zone] += t.Cents
	}
	fmt.Printf("tickets=%d valid=%d rejected=%d total=%d\n", rep.Tickets, len(rep.Valid), len(rep.Rejects), total)
	zones := make([]string, 0, len(count))
	for z := range count {
		zones = append(zones, z)
	}
	sort.Strings(zones)
	for _, z := range zones {
		fmt.Printf("zone=%s count=%d total=%d\n", z, count[z], sums[z])
	}
	if len(rep.Rejects) > 0 {
		os.Exit(1)
	}
}
```

#### `parking/core_test.go`

```go
package parking_test

import (
	"reflect"
	"testing"

	"example.com/parking/parking"
)

// spyRates is a test double: it records every lookup (spy) and answers from a table (stub).
type spyRates struct {
	table map[string]int
	calls []string
}

func (s *spyRates) Rate(zone string) (int, bool) {
	s.calls = append(s.calls, zone)
	r, ok := s.table[zone]
	return r, ok
}

func TestFee(t *testing.T) {
	// minutes, rate, expected cents: boundaries first
	cases := []struct{ minutes, rate, want int }{
		{0, 120, 0}, {15, 120, 0}, {16, 120, 120}, {60, 120, 120}, {61, 120, 240},
		{480, 120, 960}, {481, 120, 960}, {1439, 120, 960}, {1440, 120, 960},
		{1455, 120, 960}, {1456, 120, 1080}, {2880, 80, 1280},
	}
	for _, c := range cases {
		t.Run("", func(t *testing.T) {
			if got := parking.Fee(c.minutes, c.rate); got != c.want {
				t.Errorf("Fee(%d, %d) = %d, want %d", c.minutes, c.rate, got, c.want)
			}
		})
	}
}

func TestParseTime(t *testing.T) {
	if m, ok := parking.ParseTime("23:59"); !ok || m != 1439 {
		t.Errorf("ParseTime(23:59) = %d, %v", m, ok)
	}
	for _, bad := range []string{"", "9:05", "09:5", "24:00", "12:60", "12-30", "ab:cd", " 9:05", "09:05:00", "+9:05"} {
		if _, ok := parking.ParseTime(bad); ok {
			t.Errorf("ParseTime(%q) accepted", bad)
		}
	}
}

func TestDurationWrapsMidnight(t *testing.T) {
	if got := parking.Duration(22*60, 6*60); got != 8*60 {
		t.Errorf("got %d", got)
	}
}

func TestRunRejectReasons(t *testing.T) {
	spy := &spyRates{table: map[string]int{"A": 120}}
	rep := parking.Run([]string{"t1,A,08:00,09:00", "t2,Z,08:00,09:00", "t3,A,8:00,09:00",
		"t4,A,08:00", "t1,A,10:00,11:00", "", "  "}, spy)
	if rep.Tickets != 5 {
		t.Fatalf("tickets = %d", rep.Tickets)
	}
	if want := []parking.Ticket{{ID: "t1", Zone: "A", Minutes: 60, Cents: 120}}; !reflect.DeepEqual(rep.Valid, want) {
		t.Errorf("valid = %+v", rep.Valid)
	}
	want := []parking.Reject{{2, "zone"}, {3, "time"}, {4, "fields"}, {5, "duplicate"}}
	if !reflect.DeepEqual(rep.Rejects, want) {
		t.Errorf("rejects = %+v", rep.Rejects)
	}
}

func TestRateSourceAskedOnlyForWellFormedLines(t *testing.T) {
	spy := &spyRates{table: map[string]int{"A": 120}}
	parking.Run([]string{"t1,A,08:00,09:00", "t2,B,99:00,09:00"}, spy)
	if !reflect.DeepEqual(spy.calls, []string{"A"}) {
		t.Errorf("calls = %v", spy.calls)
	}
}
```

#### `parking/props_test.go`

```go
package parking_test

import (
	"testing"
	"testing/quick"

	"example.com/parking/parking"
)

func TestFeeProperties(t *testing.T) {
	// testing/quick feeds random arguments; we shrink them into range ourselves
	monotonic := func(a, b uint16, r uint16) bool {
		lo, hi, rate := int(a%14400), int(b%14400), int(r%1000)+1
		if lo > hi {
			lo, hi = hi, lo
		}
		return parking.Fee(lo, rate) <= parking.Fee(hi, rate) // P2
	}
	capPerDay := func(m uint16, r uint16) bool {
		min, rate := int(m%14400), int(r%1000)+1
		return parking.Fee(min, rate) <= (min+1439)/1440*parking.CapHours*rate // P3
	}
	freeWindow := func(m uint8, r uint16) bool {
		return parking.Fee(int(m%16), int(r%1000)+1) == 0 // P1
	}
	for name, p := range map[string]any{"monotonic": monotonic, "cap": capPerDay, "free": freeWindow} {
		if err := quick.Check(p, nil); err != nil {
			t.Errorf("%s: %v", name, err)
		}
	}
}
```

#### `parking/fuzz_test.go`

```go
package parking_test

import (
	"fmt"
	"testing"

	"example.com/parking/parking"
)

// FuzzParseTime is a native fuzz target: the engine mutates the seeds
// and keeps any input that reaches new code or breaks the property.
func FuzzParseTime(f *testing.F) {
	for _, s := range []string{"00:00", "23:59", "9:05", "24:00", "", "+9:05"} {
		f.Add(s)
	}
	f.Fuzz(func(t *testing.T, s string) {
		m, ok := parking.ParseTime(s)
		if ok && fmt.Sprintf("%02d:%02d", m/60, m%60) != s { // P5: round trip
			t.Errorf("ParseTime(%q) = %d does not round-trip", s, m)
		}
	})
}

func BenchmarkFee(b *testing.B) {
	sink := 0
	for i := 0; i < b.N; i++ {
		sink += parking.Fee(i%3000, 120)
	}
	_ = sink
}

var (
	rateVar = 120 // not a constant: the compiler cannot fold it
	sinkVar int   // package-level sink: the result must be kept
)

//go:noinline
func feeNoInline(m, r int) int { return parking.Fee(m, r) }

func BenchmarkFeeGuarded(b *testing.B) {
	s := 0
	for i := 0; i < b.N; i++ {
		s += feeNoInline(i%3000, rateVar)
	}
	sinkVar = s
}
```
