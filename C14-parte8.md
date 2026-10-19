> File C14-parte8.md (parte 8 di 8) — C14 I/O, serializzazione, tempo; corso di programmazione multi-linguaggio

### Go — `cd go && go build -o coldchain . && ./coldchain ../readings.ndjson`

#### `go.mod`

```go
module example.com/coldchain

go 1.24
```

#### `coldchain/core.go`

```go
// Package coldchain streams NDJSON temperature readings, validates them at the boundary
// and reports per local day.
package coldchain

import (
	"bufio"
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"math"
	"regexp"
	"sort"
	"strings"
	"time"
	"unicode/utf8"
)

const (
	GapSeconds = 30 * 60   // T1 - silence longer than this is reported
	TempMin    = -500      // tenths of a degree Celsius: -50.0 ..
	TempMax    = 600       // .. 60.0
	jsonWS     = " \t\r\n" // the only whitespace JSON knows
)

var bom = []byte{0xEF, 0xBB, 0xBF} // UTF-8 byte order mark

// rome is the warehouse's civil time; it needs the tz database (see the experiment on tzdata).
var rome = func() *time.Location {
	loc, err := time.LoadLocation("Europe/Rome")
	if err != nil {
		panic(err)
	}
	return loc
}()

// Reason says why a line was rejected: a closed set, not free text.
type Reason string

const (
	Encoding  Reason = "encoding"  // not valid UTF-8
	BadJSON   Reason = "json"      // not valid JSON
	Field     Reason = "field"     // valid JSON, wrong shape or type
	BadTime   Reason = "time"      // timestamp malformed or impossible
	BadRange  Reason = "range"     // temperature out of range or too precise
	Duplicate Reason = "duplicate" // same sensor and instant already seen
)

// Reading is a validated temperature reading.
type Reading struct {
	Sensor string
	At     int64 // UTC instant, seconds since 1970-01-01T00:00:00Z
	Tenths int   // temperature in tenths of a degree Celsius
}

// Decode is T2: bytes -> text -> JSON value, or the reason why not.
func Decode(raw []byte) (any, Reason) {
	if !utf8.Valid(raw) {
		return nil, Encoding
	}
	var v any
	if err := json.Unmarshal(raw, &v); err != nil {
		return nil, BadJSON
	}
	return v, ""
}

var (
	sensorRe = regexp.MustCompile(`^[a-z0-9-]{1,32}$`)
	stampRe  = regexp.MustCompile(`^([0-9]{4})-([0-9]{2})-([0-9]{2})T([0-9]{2}):([0-9]{2}):([0-9]{2})(?:Z|[+-]([0-9]{2}):([0-9]{2}))$`)
)

func atoi(s string) int {
	n := 0
	for _, c := range s {
		n = n*10 + int(c-'0')
	}
	return n
}

// ParseInstant is T4: strict RFC 3339 (no fraction) -> UTC epoch seconds.
func ParseInstant(text string) (int64, bool) {
	m := stampRe.FindStringSubmatch(text)
	if m == nil {
		return 0, false
	}
	y, hh, mi, ss, oh, om := atoi(m[1]), atoi(m[4]), atoi(m[5]), atoi(m[6]), atoi(m[7]), atoi(m[8])
	if y < 1970 || y > 2100 || hh > 23 || mi > 59 || ss > 59 || oh > 14 || om > 59 {
		return 0, false
	}
	t, err := time.Parse(time.RFC3339, text) // also checks the calendar: time.Date would roll Feb 30 over
	if err != nil {
		return 0, false
	}
	return t.Unix(), true
}

// ToTenths converts degrees to tenths, only if in range with at most one decimal.
func ToTenths(x float64) (int, bool) {
	scaled := x * 10
	if !(TempMin <= scaled && scaled <= TempMax) { // false for NaN too
		return 0, false
	}
	n := math.Round(scaled)
	if math.Abs(scaled-n) >= 1e-6 {
		return 0, false
	}
	return int(n), true
}

// Validate is T3: a decoded JSON value -> Reading, or the first rule it breaks.
func Validate(v any) (Reading, Reason) {
	o, ok := v.(map[string]any)
	if !ok {
		return Reading{}, Field
	}
	sensor, ok1 := o["sensor"].(string)
	ts, ok2 := o["ts"].(string)
	temp, ok3 := o["temp"].(float64) // a JSON number; true is a bool, not a float64
	if !ok1 || !ok2 || !ok3 || !sensorRe.MatchString(sensor) {
		return Reading{}, Field
	}
	at, ok := ParseInstant(ts)
	if !ok {
		return Reading{}, BadTime
	}
	tenths, ok := ToTenths(temp)
	if !ok {
		return Reading{}, BadRange
	}
	return Reading{sensor, at, tenths}, ""
}

func localDate(at int64) string { return time.Unix(at, 0).In(rome).Format("2006-01-02") }

// DayHours is T5: the length of a local day in hours: 23, 24 or 25.
func DayHours(date string) int {
	start, _ := time.ParseInLocation("2006-01-02", date, rome)
	end := time.Date(start.Year(), start.Month(), start.Day()+1, 0, 0, 0, 0, rome)
	return int(end.Sub(start).Hours())
}

func localISO(at int64) string { return time.Unix(at, 0).In(rome).Format(time.RFC3339) }

// Rejected is a rejected input line.
type Rejected struct {
	Line   int    `json:"line"`
	Reason Reason `json:"reason"`
}

// Report is the outcome of Run.
type Report struct {
	Readings int
	Valid    []Reading
	Rejects  []Rejected
}

// Run is T6: consume the stream one line at a time; blank lines are skipped.
func Run(in *bufio.Reader) (Report, error) {
	var rep Report
	type key struct {
		sensor string
		at     int64
	}
	seen := map[key]bool{}
	for n := 1; ; n++ {
		raw, err := in.ReadBytes('\n') // no Scanner: its 64 KiB line limit would silently stop the loop
		if len(raw) > 0 {
			if n == 1 {
				raw = bytes.TrimPrefix(raw, bom)
			}
			if strings.Trim(string(raw), jsonWS) != "" {
				rep.Readings++
				v, reason := Decode(raw)
				var r Reading
				if reason == "" {
					r, reason = Validate(v)
				}
				switch {
				case reason != "":
					rep.Rejects = append(rep.Rejects, Rejected{n, reason})
				case seen[key{r.Sensor, r.At}]:
					rep.Rejects = append(rep.Rejects, Rejected{n, Duplicate})
				default:
					seen[key{r.Sensor, r.At}] = true
					rep.Valid = append(rep.Valid, r)
				}
			}
		}
		if err == io.EOF {
			return rep, nil
		}
		if err != nil {
			return rep, err
		}
	}
}

// Day is one sensor on one local day.
type Day struct {
	Sensor string `json:"sensor"`
	Date   string `json:"date"`
	Hours  int    `json:"hours"`
	Count  int    `json:"count"`
	Lo     int    `json:"min_tenths"`
	Hi     int    `json:"max_tenths"`
	AvgH   int    `json:"avg_hundredths"` // mean in hundredths of a degree
}

// Gap is a silence longer than GapSeconds.
type Gap struct {
	Sensor  string `json:"sensor"`
	Start   string `json:"from"`
	End     string `json:"to"`
	Minutes int    `json:"minutes"`
}

// DivRound is n / d rounded half away from zero (d > 0); integer division differs by language.
func DivRound(n, d int) int {
	abs := n
	if n < 0 {
		abs = -n
	}
	q := (2*abs + d) / (2 * d)
	if n < 0 {
		return -q
	}
	return q
}

// Daily groups valid readings by sensor and local date.
func Daily(valid []Reading) []Day {
	type key struct{ sensor, date string }
	groups := map[key][]int{}
	for _, r := range valid {
		k := key{r.Sensor, localDate(r.At)}
		groups[k] = append(groups[k], r.Tenths)
	}
	keys := make([]key, 0, len(groups))
	for k := range groups {
		keys = append(keys, k)
	}
	sort.Slice(keys, func(i, j int) bool {
		if keys[i].sensor != keys[j].sensor {
			return keys[i].sensor < keys[j].sensor
		}
		return keys[i].date < keys[j].date
	})
	out := []Day{} // not nil: a nil slice would be encoded as null
	for _, k := range keys {
		v := groups[k]
		lo, hi, sum := v[0], v[0], 0
		for _, x := range v {
			lo, hi, sum = min(lo, x), max(hi, x), sum+x
		}
		out = append(out, Day{k.sensor, k.date, DayHours(k.date), len(v), lo, hi, DivRound(sum*10, len(v))})
	}
	return out
}

// Gaps finds silences between consecutive readings of the same sensor.
func Gaps(valid []Reading) []Gap {
	by := map[string][]int64{}
	for _, r := range valid {
		by[r.Sensor] = append(by[r.Sensor], r.At)
	}
	sensors := make([]string, 0, len(by))
	for s := range by {
		sensors = append(sensors, s)
	}
	sort.Strings(sensors)
	out := []Gap{}
	for _, s := range sensors {
		times := by[s]
		sort.Slice(times, func(i, j int) bool { return times[i] < times[j] })
		for i := 1; i < len(times); i++ {
			if d := times[i] - times[i-1]; d > GapSeconds {
				out = append(out, Gap{s, localISO(times[i-1]), localISO(times[i]), int(d / 60)})
			}
		}
	}
	return out
}

// Fixed writes an integer in 10**-decimals units as decimal text, no floats involved.
func Fixed(n, decimals int) string {
	p := 1
	for i := 0; i < decimals; i++ {
		p *= 10
	}
	sign, abs := "", n
	if n < 0 {
		sign, abs = "-", -n
	}
	return fmt.Sprintf("%s%d.%0*d", sign, abs/p, decimals, abs%p)
}

// RenderText is the human-readable report.
func RenderText(rep Report) string {
	var b strings.Builder
	fmt.Fprintf(&b, "readings=%d valid=%d rejected=%d\n", rep.Readings, len(rep.Valid), len(rep.Rejects))
	for _, d := range Daily(rep.Valid) {
		fmt.Fprintf(&b, "day sensor=%s date=%s hours=%d count=%d min=%s max=%s avg=%s\n",
			d.Sensor, d.Date, d.Hours, d.Count, Fixed(d.Lo, 1), Fixed(d.Hi, 1), Fixed(d.AvgH, 2))
	}
	for _, g := range Gaps(rep.Valid) {
		fmt.Fprintf(&b, "gap sensor=%s from=%s to=%s minutes=%d\n", g.Sensor, g.Start, g.End, g.Minutes)
	}
	return b.String()
}

// RenderJSON is the machine-readable report: one compact line, fixed key order.
func RenderJSON(rep Report) (string, error) {
	rejects := rep.Rejects
	if rejects == nil {
		rejects = []Rejected{}
	}
	doc := struct {
		Readings int        `json:"readings"`
		Valid    int        `json:"valid"`
		Rejected int        `json:"rejected"`
		Days     []Day      `json:"days"`
		Gaps     []Gap      `json:"gaps"`
		Rejects  []Rejected `json:"rejects"`
	}{rep.Readings, len(rep.Valid), len(rep.Rejects), Daily(rep.Valid), Gaps(rep.Valid), rejects}
	var buf bytes.Buffer
	enc := json.NewEncoder(&buf)
	enc.SetEscapeHTML(false) // by default Go writes < > & as \u003c \u003e \u0026
	if err := enc.Encode(doc); err != nil {
		return "", err
	}
	return buf.String(), nil // Encode already added the newline
}
```

#### `main.go`

```go
package main

import (
	"bufio"
	"compress/gzip"
	"fmt"
	"io"
	"os"
	"strings"

	_ "time/tzdata" // embed the tz database: scratch and distroless images have none

	"example.com/coldchain/coldchain"
)

// openInput returns a stream of bytes: stdin, a gzip file decompressed on the fly, or a plain file.
func openInput(path string) (io.Reader, func(), error) {
	if path == "-" {
		return os.Stdin, func() {}, nil
	}
	f, err := os.Open(path)
	if err != nil {
		return nil, nil, err
	}
	if !strings.HasSuffix(path, ".gz") {
		return f, func() { f.Close() }, nil
	}
	gz, err := gzip.NewReader(f)
	if err != nil {
		f.Close()
		return nil, nil, err
	}
	return gz, func() { gz.Close(); f.Close() }, nil
}

// T8 - CLI: stream the file (or stdin), report on stdout, rejects on stderr.
func main() {
	var args []string
	asJSON := false
	for _, a := range os.Args[1:] {
		if a == "--json" {
			asJSON = true
		} else {
			args = append(args, a)
		}
	}
	path := "-"
	if len(args) > 0 {
		path = args[0]
	}
	in, closeIn, err := openInput(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: cannot read %s\n", path)
		os.Exit(2)
	}
	rep, err := coldchain.Run(bufio.NewReader(in))
	closeIn()
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: cannot read %s\n", path)
		os.Exit(2)
	}
	for _, r := range rep.Rejects {
		fmt.Fprintf(os.Stderr, "line %d: rejected (%s)\n", r.Line, r.Reason)
	}
	if asJSON {
		out, err := coldchain.RenderJSON(rep)
		if err != nil {
			fmt.Fprintln(os.Stderr, "error:", err)
			os.Exit(2)
		}
		fmt.Print(out)
	} else {
		fmt.Print(coldchain.RenderText(rep))
	}
	if len(rep.Rejects) > 0 {
		os.Exit(1)
	}
}
```
