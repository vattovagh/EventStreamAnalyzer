# Event Stream Analyzer

Event Stream Analyzer is a Java 17 command-line tool that reads a log file of `key=value` events and sorts each line into one of three record types: Sensor Readings, Device Events and Access Records. It then computes summary statistics for each type and writes them to three JSON files: `sensors.json`, `devices.json` and `access.json`. Lines that are malformed or don't match any record type are skipped and counted in the summary line, never treated as errors.

## Prerequisites

- JDK 17 or newer
- Maven 3.8 or newer

## Build

```bash
mvn package
```

This compiles the code, runs the tests and produces a runnable jar at `target/analyzer.jar`.

## Run

```bash
java -jar target/analyzer.jar --file samples/events.txt --outdir results/
```

Usage:

```
java -jar analyzer.jar --file <input.txt> [--outdir <directory>]
```

- `--file` is required.
- `--outdir` is optional and defaults to the current directory. It is created if it doesn't exist.
- Flags may appear in any order.

On success the tool prints a summary such as:

```
Processed 14 lines (2 skipped). Wrote sensors.json, devices.json, access.json to results/
```

All three output files are always written. If a record type has no data, its file contains exactly `{}`.

## Test

```bash
mvn test
```

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Input file not found, input file unreadable (or a directory), or the output directory cannot be created or written |
| 2 | Bad command-line arguments: missing `--file`, a flag with no value, or an unknown flag |

Error messages go to stderr. Arguments errors also print the usage line.

## Input format

Each line is a list of space-separated `key=value` pairs. Values containing spaces are wrapped in double quotes (no escape sequences are supported):

```
ts=2026-11-09T10:00:00Z severity=INFO device=thermostat-01 msg="Heartbeat OK"
```

A line is rejected if any token is malformed: a token with no `=`, an empty key, an unterminated quote, or a closing quote not followed by whitespace. If a key repeats, the last value wins. The `ts` key is parsed like any other key and then ignored.

## Record types

A line is checked against the types in this fixed order, and the first match wins: **Sensor → Device → Access**. Extra keys are ignored. A line that matches none is skipped.

| Type | Required keys | Validation |
| --- | --- | --- |
| Sensor Reading | `sensor`, `reading` | `sensor` is non-empty; `reading` is a plain decimal number (optional sign, optional exponent). `NaN`, `Infinity`, `1d` and hex floats are rejected. |
| Device Event | `severity`, `device` | `severity` is exactly `CRITICAL`, `ERROR`, `WARN` or `INFO` (case-sensitive); `device` is non-empty. |
| Access Record | `user`, `door`, `result`, `duration_ms` | `result` is exactly `GRANTED` or `DENIED`; `duration_ms` is an integer ≥ 0; `user` and `door` are non-empty. |

## Output

**`sensors.json`**: one entry per sensor with `count`, `minimum`, `median`, `average`, `maximum` and `std_dev`.

**`devices.json`**:
- `by_severity`: the count for each of `CRITICAL`, `ERROR`, `WARN`, `INFO`. All four keys are always present, with 0 for unseen levels.
- `by_device`: per device, `total` and `most_frequent_severity`. A tie is broken in favor of the more severe level.

**`access.json`**: one entry per door with:
- `decision_times`: `min`, `50_percentile`, `90_percentile`, `99_percentile` and `max` of `duration_ms`.
- `results`: `GRANTED` and `DENIED` counts. Both keys are always present.
- `flagged_users`: users with 3 or more `DENIED` results on that door, sorted alphabetically, or `[]`.

Output is UTF-8, indented with two spaces, and ends with a single newline.

## Calculation rules

- **Percentiles** use the nearest-rank method: the value at index `ceil(p/100 × N) − 1` of the sorted list, with the rank clamped to at least 1. The rank is computed in integer math to avoid floating-point errors.
- **Median** of an odd number of values is the middle value. For an even number of values it is the mean of the two middle values.
- **p50 and median can differ on purpose.** The 50th percentile uses nearest-rank and always returns an actual value from the data, while the median averages the two middle values for even N.
- **Standard deviation** is the population standard deviation (divide by N, not N − 1), computed from the unrounded mean.
- **Rounding** is HALF_UP to 1 decimal place, applied only at the very end.
- **Integer output rule:** after rounding, a value with a zero fractional part is written as an integer (`40`, not `40.0`); otherwise it is written with one decimal (`21.3`). Counts, totals and durations are always integers.
- **Classification order** is Sensor → Device → Access. A line that qualifies as more than one type is counted as the first.
- **Malformed lines:** a line with any malformed token is skipped entirely.

## Project layout

```
src/main/java/edu/eventanalyzer/
  Main.java        entry point (the only place that calls System.exit)
  cli/             argument parsing and the application runner
  pipeline/        read lines -> parse -> dispatch -> collect results
  parse/           line parsers (key=value)
  classify/        per-type classifiers and shared field validators
  model/           records and enums for each event type
  aggregate/       per-type aggregators, statistics and number formatting
  recordtype/      record type definitions, handler chain and registry
  output/          output writers (JSON)
samples/           sample input
src/test/          unit and end-to-end tests
docs/              design document and class diagram
```

The processing pipeline has four stages (parse, dispatch, aggregate, write), each behind an interface. Adding a new record type means adding its classes and registering it in `RecordTypeRegistry`; adding a new input or output format means adding a parser or writer and one case in the matching factory. Existing logic does not need to change.

## Dependencies

- [Jackson](https://github.com/FasterXML/jackson) (`jackson-databind`) for JSON output
- [JUnit 5](https://junit.org/junit5/) for tests (test scope only)
