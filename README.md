<!---
{
  "id": "d88d452b-baf5-4960-8685-9d17eddf4537",
  "teaches": "Debugging: Structured Logging as JSON Lines, Evaluated with jq",
  "depends_on": [
    "b63c3f2e-d7cb-4dd0-ad74-26eb91a7173f",
    "b84109dc-c1a3-4acd-90b0-60758873854c"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-05",
  "keywords": [
    "logging",
    "JSON lines",
    "jq",
    "stderr",
    "log level",
    "structured data",
    "timestamps"
  ]
}
--->

# Debugging: Structured Logging as JSON Lines, Evaluated with `jq`

> In this exercise you will replace free-text debug output with structured log records: one JSON object per line, with a timestamp, a level and named fields. You will then answer questions about a program's behaviour with `jq` instead of reading thousands of lines. You will also see where logging reaches its limits. No `sudo` is required.

## Introduction

`DBG("i=%d sum=%d", i, sum)` is fine for one run that you read with your own eyes. It is useless for a bug that appears once in a thousand runs, or only on a customer's machine, or only after the program has been running for an hour. For those you need output that is *kept* and *machine readable*.

The simplest robust format is **JSON Lines**: every line is a complete JSON object, so a file can be appended to forever, cut with `head` and `tail`, and queried with `jq`. Each record carries a timestamp, a severity level, the place in the code and whatever fields matter. Logging is not a temporary hack like `DBG`; it ships with the product and is usually controlled by a level at run time.

### 1.1) Further Readings and Other Sources

* [JSON Lines specification](https://jsonlines.org/)
* [jq Manual](https://jqlang.github.io/jq/manual/)
* [RFC 5424: The Syslog Protocol (severity levels)](https://www.rfc-editor.org/rfc/rfc5424#section-6.2.1)
* [clock_gettime(2) man page](https://man7.org/linux/man-pages/man2/clock_gettime.2.html)

## Tasks

### Task 1: A Minimal JSON Logger

```bash
mkdir -p ~/dbg-log && cd ~/dbg-log
```

```c
// file: log.h
#ifndef LOG_H
#define LOG_H
#include <stdio.h>
#include <time.h>

enum log_level { LOG_ERROR, LOG_WARN, LOG_INFO, LOG_DEBUG };
extern enum log_level log_threshold;

#define LOG(level, event, fmt, ...) do {                                     \
    if ((level) <= log_threshold) {                                          \
        struct timespec ts; clock_gettime(CLOCK_REALTIME, &ts);              \
        fprintf(stderr,                                                      \
            "{\"ts\":%ld.%03ld,\"level\":\"%s\",\"src\":\"%s:%d\","           \
            "\"event\":\"%s\"," fmt "}\n",                                    \
            (long)ts.tv_sec, ts.tv_nsec / 1000000L,                          \
            log_level_name(level), __FILE__, __LINE__, event, __VA_ARGS__);  \
    }                                                                        \
} while (0)

const char *log_level_name(enum log_level l);
#endif
```

```c
// file: log.c
#include "log.h"
enum log_level log_threshold = LOG_INFO;
const char *log_level_name(enum log_level l) {
    static const char *n[] = { "ERROR", "WARN", "INFO", "DEBUG" };
    return n[l];
}
```

The `do { ... } while (0)` wrapper makes the macro behave like a single statement, so it is safe inside an `if` without braces. `fmt` must contain the *remaining* JSON fields, for example `"\"id\":%d,\"value\":%.3f"`.

### Task 2: A Program That Logs

```c
// file: sensor.c
#include <stdlib.h>
#include <string.h>
#include "log.h"

static double read_sensor(int id) {
    double v = 20.0 + (rand() % 1000) / 100.0;        /* 20.00 .. 29.99 */
    if (rand() % 50 == 0) v = -999.0;                 /* rare glitch */
    LOG(LOG_DEBUG, "read", "\"id\":%d,\"value\":%.2f", id, v);
    return v;
}

int main(int argc, char **argv) {
    if (argc > 1 && strcmp(argv[1], "-v") == 0) log_threshold = LOG_DEBUG;
    LOG(LOG_INFO, "start", "\"sensors\":%d", 3);
    double sum = 0; int n = 0;
    for (int round = 0; round < 1000; round++) {
        for (int id = 0; id < 3; id++) {
            double v = read_sensor(id);
            if (v < -100) { LOG(LOG_WARN, "glitch", "\"id\":%d,\"round\":%d", id, round); continue; }
            sum += v; n++;
        }
    }
    LOG(LOG_INFO, "done", "\"n\":%d,\"mean\":%.3f", n, sum / n);
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 sensor.c log.c -o sensor
./sensor 2> run.log
./sensor -v 2> run_verbose.log
wc -l run.log run_verbose.log
head -3 run.log
```

Note that the program's own output (none here) and the log are separated by the file descriptor, just like in the `DBG` exercise.

### Task 3: Check That Every Line Is Valid JSON

```bash
jq -c . run_verbose.log > /dev/null && echo "all lines valid"
jq -c 'select(.level == "WARN")' run.log | head
```

If `jq` complains, a format string produced invalid JSON. That is the price of building JSON by hand; find the broken line with `jq -c . run_verbose.log 2>&1 | grep -n parse`.

### Task 4: Answer Questions With `jq`

```bash
# how many glitches, per sensor?
jq -r 'select(.event == "glitch") | .id' run.log | sort | uniq -c

# mean value per sensor from the verbose log
jq -r 'select(.event == "read" and .value > -100) | "\(.id) \(.value)"' run_verbose.log \
  | awk '{ s[$1] += $2; c[$1]++ } END { for (k in s) printf "sensor %s: %.3f\n", k, s[k]/c[k] }'

# in which round did the first glitch happen, and how long after start?
jq -s 'map(select(.event == "start" or .event == "glitch")) | .[0:2] | (.[1].ts - .[0].ts)' run.log
```

Write one more query yourself: the number of `read` events between two consecutive glitches.

### Task 5: Levels at Run Time

```bash
./sensor      2>&1 | jq -r .level | sort | uniq -c
./sensor -v   2>&1 | jq -r .level | sort | uniq -c
```

The same binary produces both. Compare with `DBG`, which needs a rebuild to switch.

### Task 6: Where Logging Hurts

Add a tight loop that logs at `DEBUG` level on every iteration and time it as in the `DBG` exercise, once with `-v 2>/dev/null` and once without `-v`. Then estimate: a UART at 115200 baud transports about 11\,500 bytes per second. How many of your log lines (measure one with `wc -c`) fit into one second? Compare with a 1\,kHz control loop.

### Task 7: Rotate and Keep

```bash
for i in 1 2 3; do ./sensor 2>> history.log; done
wc -l history.log
jq -s 'map(select(.event == "done")) | map(.mean)' history.log
```

Appending works because every line is self-contained. Nothing in the file depends on the lines before it.

## Questions

- Why one JSON object per line instead of one big JSON array for the whole file? What breaks with the array when the program crashes?
- What does `do { ... } while (0)` protect against? Write an `if`/`else` that misbehaves without it.
- Why is the timestamp taken with `CLOCK_REALTIME` here, although the `DBG` exercise used `CLOCK_MONOTONIC` for timing?
- A log line is ~120 bytes. How many lines per second fit through a 115200-baud UART, and what does that mean for logging inside a 1\,kHz loop?
- Logging shows the fields you chose to log. Describe a bug that this log could *not* reveal.
- Logging and `DBG` both change timing. Which one is easier to leave in production code, and why?

## Advice

Structured logging is the tool for "I need to know what happened, after the fact, possibly on a machine I cannot touch". It is part of the program, controlled at run time, and evaluated with tools instead of eyes. Learn the three or four `jq` expressions from Task 4 by heart; they answer most questions you will ever ask a log.

Its limits are the same as `DBG`'s, only better hidden: every line costs time, every field was chosen in advance. When the question is "what is the state of *everything* right now, at this line", neither `DBG` nor a log can answer it. That is what the debugger in the next exercises is for.
