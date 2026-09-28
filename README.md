# yorn <!-- omit from toc -->

`yorn` ("yes or no") answers a yes/no question about a piece of text. It's a
thin CLI over the TypeSafe **Noul** primitive: pipe in a document, ask a
question, get back `yes` or `no` — with the answer also encoded in the exit
code, so `yorn` drops straight into shell conditionals like `grep` or `test`.

This is the **bash prototype** (stage 0). A dependency-free Go binary with the
same name and behavior is planned next

## Table of Contents <!-- omit from toc -->

- [Pre-Reqs](#pre-reqs)
- [Quick start](#quick-start)
- [Usage](#usage)
- [Flags](#flags)
  - [Examples](#examples)
- [Exit codes](#exit-codes)
- [Why bash first, Go next](#why-bash-first-go-next)


## Pre-Reqs

- `bash`, `curl`, `jq`
- `TYPESAFE_API_KEY` in the environment

```bash
export TYPESAFE_API_KEY=your_key_here
```

`yorn` reads the key from the environment; exporting it is the standard way.
(If the key lives in the repo-root `.env`, `set -a; source .env; set +a`
picks it up too.)

## Quick start

```bash
$ yorn -d 'URGENT!! You have won $1,000,000 - click NOW!' -q "Is this truly urgent?"
no
$ echo $?
0

$ yorn -d 'Attaching the Q3 notes ahead of the sync tomorrow.' -q "Is this time sensitive?"
yes
$ echo $?
1
```

`-d` passes the document inline — single quotes, so `$` and `!!` pass through
literally. The document usually comes from stdin instead, piped in:

```bash
if cat email.txt | yorn "Is this spam?"; then
  mv email.txt spam/
fi
```

The exit code *is* the answer (0 = yes, 1 = no), so `yorn` drops straight into
shell conditionals like `grep` or `test`.

## Usage

```
yorn [OPTIONS] QUESTION
```

The document comes from stdin by default, or from `-d TEXT` / `-f PATH`.

## Flags

| Flag | Meaning |
|---|---|
| `-q, --question TEXT` | the question (alternative to positional) |
| `-d, --doc TEXT` | inline document |
| `-f, --file PATH` | document from file (`-` = stdin) |
| `-t, --threshold FLOAT` | yes/no cutoff in `[0, 1]`, default 0.5 |
| `-p, --prob` | print the probability instead of `yes`/`no` |
| `-m, --model MODEL` | model id, default `speed_latest` |
| `--yes-means TEXT`, `--no-means TEXT` | optional Noul criteria ends |
| `--json` | print raw API response (mutually exclusive with `-p`) |
| `--quiet` | suppress stdout; exit code only |
| `--timeout DURATION` | end-to-end deadline, default `5s` |
| `--max-doc-bytes N` | document size cap, default 1 MiB |
| `-h, --help`, `--version` | the usual |

### Examples

<details>
<summary>Flag examples — click to expand</summary>

Each flag gets its own subsection; the command is the input, the line below
it is the output.

**Pipe a document from stdin (default)**

The default input is stdin, read to EOF.

```bash
$ cat email.txt | yorn "Is this message spam?"
yes
```

**Inline document — `-d, --doc`**

```bash
$ yorn -d "URGENT!! claim your prize now" "Is this message spam?"
yes
```

**Document from a file — `-f, --file`**

```bash
$ yorn -f ticket.txt "Is the customer angry?"
no
```

`-` means stdin explicitly, e.g. `yorn -f - "Is this spam?"`.

**Question from a flag — `-q, --question`**

```bash
$ Q="Does this mention a security vulnerability?"
$ cat email.txt | yorn -q "$Q"
no
```

**Raise the bar — `-t, --threshold`**

Same document, different cutoff — the threshold moves where "yes" starts:

```bash
$ cat email.txt | yorn -t 0.5 "Is this message spam?"
yes
$ cat email.txt | yorn -t 0.9 "Is this message spam?"
no
```

**Raw probability — `-p, --prob`**

```bash
$ cat review.txt | yorn -p "Is this review positive?"
0.83
```

The exit code still reflects the thresholded answer, so `-p` composes with
`if` the same way.

**Model — `-m, --model`**

```bash
$ cat email.txt | yorn -m speed_latest "Is this spam?"
yes
```

**Pin down the ends — `--yes-means` / `--no-means`**

```bash
$ yorn --yes-means "states a concrete deadline or asks for same-day action" \
       --no-means "purely informational, no time pressure" \
       "Is this message urgent?" < msg.txt
yes
```

**Raw API response — `--json`**

```bash
$ cat email.txt | yorn --json "Is this spam?" | jq .answers.q.noul
0.94
```

`--json` prints the body byte-for-byte as the API returned it, for `jq` or
debugging. Mutually exclusive with `-p`.

**Pure predicate — `--quiet`**

No stdout at all; the exit code is the only output.

```bash
$ for f in inbox/*.txt; do
    yorn --quiet -f "$f" "Does this mention a security vulnerability?" && echo "$f"
  done
inbox/incident.txt
```

**Tighter deadline — `--timeout`**

```bash
$ yorn --timeout 1s "Is this comment toxic?" < comment.txt
no
```

A late answer is as bad as no answer in a latency-sensitive pipeline; tighten
the deadline and let a timeout surface as exit 3.

**Size cap — `--max-doc-bytes`**

```bash
$ yorn --max-doc-bytes 65536 -f app.log "Does this log contain a crash?"
yes
```

A mistaken `cat bigfile.bin |` fails fast instead of shipping megabytes to the
API.

</details>

## Exit codes

| Code | Meaning |
|---|---|
| 0 | yes (`noul >= threshold`) |
| 1 | no (`noul < threshold`) |
| 2 | usage error — bad flags, missing question/document/API key, bad threshold |
| 3 | runtime error — network failure, timeout, API error, bad response |

Usage errors (2) are caught before any network call. `-p` and `--json` don't
change the exit code — it's always the thresholded yes/no answer.

> **Exit code 1 is a valid "no" answer, not a failure.** Like `grep`, `yorn` is
a predicate: 0 = yes, 1 = no, and both are successful outcomes. Don't use
`||` to mean "it broke" — a `no` is not an error. For actual problems (missing
key, bad flags, timeout, API error) the exit code is 2 or 3. So `yorn ... ||
fallback` fires on a `no` too; check `$?` against 2/3 if you want to
distinguish a real failure from an answer.

## Why bash first, Go next

The bash version is real and works today, but it costs ~15–25ms of shell +
curl + jq startup on top of the API's own sub-100ms response — noticeable
overhead for a tool meant to be nearly free. The planned Go rewrite keeps the
exact same flags, output, and exit codes, and cuts that to a couple of
milliseconds with a single static binary.
