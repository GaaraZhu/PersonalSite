---
title: "Gate: a deterministic PII boundary between your data and AI agents"
date: 2026-05-20T20:17:00+12:00
draft: false
tags: ["AI", "Security", "PII", "Rust", "MCP", "Claude Code"]
readingTime: 10
customSummary: A small Rust CLI that intercepts query results — including those returned by your MCP servers — before they reach the model's context. Rule-driven, reproducible, and audit-ready.
---

## The thing that should have been a 2 a.m. incident

You wire a database tool into your coding agent. PostgreSQL, Databricks, an internal HTTP API — whatever. The agent is *useful*. It joins tables you'd forgotten existed, drafts the migration, and writes the report. Productivity goes up.

Two weeks later, you scroll back through a transcript and see this:

```text
> select * from users where signup_at > '2026-04-01' limit 5;

[{
  "id": 41021,
  "full_name": "Alice Johnson",
  "email": "alice.johnson@example.com",
  "phone": "+1 415-555-0142",
  "card_last_four": "4242",
  "ssn": "123-45-6789",
  "status": "active"
}, ...]
```

That's now in the model's context. From there it's in the conversation log, in any summary the agent generates, in the file it just wrote to `/tmp`, and — if the harness has any kind of memory or "share session" feature — potentially somewhere on someone else's machine. The agent didn't do anything wrong. Neither did you. The tool returned what you asked for, the model ingested it, life went on.

This is the default. Every CLI client, MCP server, and `curl | jq` pipeline returns the same bytes a human would see — and with agents, there's no human in the loop to triage what enters the model's window.

This post is about a tool that fixes the leak at the layer where it can be fixed deterministically.

## What `gate` is

[`gate`](https://github.com/GaaraZhu/gate) is a single Rust binary that sits between an AI agent and the data tools it calls. It intercepts the *output* of configured commands, scans it for PII, and rewrites the values to typed placeholders before the bytes reach the model:

![gate intercepting PII before it reaches the model](/images/introducing-gate/demo.gif)

```diff
- {"id": 1, "email": "alice@example.com", "ssn": "123-45-6789", "status": "active"}
+ {"id": 1, "email": "[PII:email]", "ssn": "[PII:ssn]", "status": "active", "_gate_summary": {"redacted": 2, "types": ["email", "ssn"]}}
```

The original JSON shape is preserved. The agent can still iterate, count, and reason about rows; it just never gets to see the values it doesn't need.

The design constraints, in order of priority:

1. **Deterministic.** No LLM-in-the-loop redaction. Same input → same output, every run, on a plane with no network.
2. **Bypass-resistant within the harness's threat model.** The agent should not be able to disable the filter by asking nicely, calling the tool in a clever way, or shelling out through a different verb.
3. **Fast on the hot path.** It runs on every Bash command the agent invokes. If it's slow, people turn it off.
4. **Honest about its limits.** A false-negative is worse than a false-positive — the failure mode is silent data exposure, not a noisy block.

The whole thing is ~7k lines of Rust with ~6k of tests (491 passing, golden-file coverage on the redactor), MIT-licensed, builds with `cargo build`, and ships on Homebrew.

## The two access paths

Modern agent harnesses give models data through two doors. The Model Context Protocol has become the de-facto integration layer for Postgres, Snowflake, GitHub, Linear, and internal APIs — and most of the published material on it is about *building* servers, not about auditing what they return. The model trusts what the server hands back. Nobody is reading the bytes.

`gate` covers both doors: MCP servers via a stdio proxy, and Bash/CLI tools via a harness hook.

### 1. MCP servers (via a stdio proxy)

`gate mcp` is a tiny stdio JSON-RPC proxy. You register it as the MCP server in your harness; it spawns the real server underneath and forwards every message verbatim — *except* `tools/call` responses, which are passed through the value-scanner before the bytes return to the model.

```text
AI ──tools/call──> gate mcp ──forward──> upstream MCP server
                       │
                       │ <── tools/call response with PII
                       │
                       │ Gate 2 scan + redact
                       │
AI <───redacted result─┘
```

The proxy is transparent. Upstream servers run unchanged, and you migrate the whole fleet in one shot:

```bash
gate init --wrap-mcp                                    # dry-run: lists every server that would be wrapped
gate init --wrap-mcp --yes                              # apply
gate init --wrap-mcp --servers postgres,github --yes   # opt-in subset
```

This converts every server in your `~/.claude.json` (or `./.mcp.json` for project scope, or the OpenCode / Copilot CLI equivalents) into a `gate mcp <original-command>` proxy in one shot. Already-proxied servers are skipped, so re-running is idempotent. When you add a new MCP server later, run it again.

What this means concretely: you can adopt a third-party MCP server you don't control — a vendor's Postgres connector, an internal team's CRM bridge — and still get a deterministic PII boundary between what it returns and what your model ingests. Without changing the server. Without trusting its author to have thought about redaction.

### 2. Bash tools (via a harness hook)

Every command the agent wants to run — `tkpsql query ...`, `psql -c ...`, `databricks api post ...`, `curl https://internal/...` — passes through `gate hook` first. The hook checks whether the command matches a tool listed in config. If it does, the command is silently rewritten to:

```text
gate run -- <original command>
```

`gate run` spawns the original subprocess, captures its stdout, runs the two-stage redaction pipeline on the bytes, and emits the sanitized result back. The agent sees the same JSON structure it always did, with values replaced by `[PII:<type>]`.

The rewrite happens in the harness's pre-tool-execution hook, which means it is **enforcing**, not advisory:

- **Claude Code** — `PreToolUse` hook in `~/.claude/settings.json`; Claude Code substitutes the rewritten command via `updatedInput` before spawning.
- **OpenCode** — a TypeScript plugin's `tool.execute.before` handler mutates `output.args.command` in-flight.
- **GitHub Copilot CLI** — `PreToolUse` hook in `.github/hooks/PreToolUse.json` returns `modifiedArgs`.

The agent doesn't know `gate` is there, and humans running the same commands in a normal terminal are untouched — there's no wrapper script on PATH.

## The two-gate detection pipeline

Despite the name, `gate` is two filters, applied in sequence, with very different jobs.

### Gate 1: SQL intent analysis (best-effort)

When the intercepted command has a `sql_arg` configured (e.g. `tkpsql --sql`, `psql -c`, `databricks --json statement`), `gate` extracts the SQL string and runs it through a hand-written tokenizer. The goal is modest: figure out which *columns* the query selects, so they're marked for guaranteed redaction regardless of what comes back in the value.

```sql
SELECT u.first_name, u.email AS contact, p.phone
FROM users u JOIN profiles p ON u.id = p.user_id
WHERE u.signup_at > NOW() - INTERVAL '30 days'
```

Gate 1 extracts `first_name`, `email` (aliased as `contact`), and `phone`. Any of those that match a PII heuristic gets added to a `forced_columns` map. Gate 2 then redacts those fields unconditionally — even if the value happens to be `NULL` or `"unknown"` or fails a regex check.

> **Why a hand-written tokenizer instead of `sqlparser-rs`?**
> Because Gate 1 only needs to find column references. Pulling in a full SQL parser turned out to be a bad trade: more dependencies, more dialect bugs, more code paths where a parse failure could silently drop columns from the plan. The tokenizer is ~300 lines, dialect-agnostic, and on a parse failure it errs toward "I don't know which columns" — which is fine, because Gate 2 then runs on every field.

Gate 1 is **explicitly best-effort**. It is documented as such. Wildcards, CTEs, function calls around columns, and weird dialects all degrade gracefully:

| Pattern | Gate 1 behaviour | Safety net |
|---|---|---|
| `SELECT email, name FROM u` | columns extracted ✓ | — |
| `SELECT LOWER(email) FROM u` | function call — column skipped | Gate 2 catches the value via email regex |
| `SELECT email AS contact` | alias tracked: `contact → email` ✓ | — |
| `SELECT * FROM u` | wildcard — no column hints | Gate 2 runs on every field; `wildcard_policy: reject` can block |
| `WITH x AS (SELECT email...)` | only outermost SELECT analysed | Gate 2 catches via value regex |
| Non-standard dialect | may produce empty plan | Gate 2 catches via value regex |

This is the load-bearing design choice in `gate`: **Gate 1 is allowed to be wrong, because Gate 2 is the safety net.**

### Gate 2: value scanning + column-name heuristics

Gate 2 runs on the JSON response after the subprocess returns. For each field, it applies three checks:

1. **Forced columns from Gate 1** → always redact, regardless of value.
2. **Column-name heuristics** → tokenise the JSON key (handling `snake_case`, `camelCase`, `PascalCase`, `UPPER_CASE`) and match against ~50 PII categories. `userEmail`, `user_email`, and `USER_EMAIL` all resolve to the same rule.
3. **Value patterns** → regex matches for email, US SSN, US phone, plus a Luhn check for payment-card numbers.

The column-name match adds a confidence boost to any value match in the same field, so a borderline value (e.g. a 9-digit string in a column called `tax_id`) tips over the redaction threshold.

The output goes back as the same JSON the tool produced, with values rewritten in-place and a `_gate_summary` block appended so the agent can reason about what was scrubbed:

```json
{
  "rows": [{"id": 1, "email": "[PII:email]", "ssn": "[PII:ssn]"}],
  "count": 1,
  "_gate_summary": {"redacted": 2, "types": ["email", "ssn"], "warnings": []}
}
```

#### Correlating without seeing

A frequent objection: "but the agent needs to *join* on email to dedupe rows." Set `hash_values: true` and each placeholder gets a deterministic 8-char hex suffix derived from the original value plus an optional salt:

```json
{"email": "[PII:email:7f83b165]"}
{"email": "[PII:email:7f83b165]"}   // same person, same suffix
{"email": "[PII:email:a3e21f9c]"}   // different person
```

The agent can group, count, and dedupe across rows without ever touching the raw value. Same property holds across separate queries within the same config.

## Knowing what you're walking into: `gate scan`

Before you wire `gate` into a harness, you usually want to know how much PII your schema actually exposes. `gate scan` reads schema metadata from stdin and prints a risk report weighted by category sensitivity (one SSN column matters more than twenty address columns):

```bash
psql -d mydb -c "SELECT TABLE_NAME, COLUMN_NAME
                 FROM information_schema.columns
                 WHERE table_schema = 'public'
                 ORDER BY table_name, ordinal_position" | gate scan
```

The output groups columns by tier (Critical / Elevated / Standard) and emits a risk floor based on prevalence. It exits 1 if any PII is found, so it slots into CI as a schema audit. False positives — `city` in a `products` table, `bank_account_id` used as a foreign key — get handled by an interactive `--review` mode that adds them to an allowlist with one keystroke each. Allowlisted columns skip *name-based* redaction only; Gate 2 still inspects their values for regex and Luhn hits.

Before, schema PII auditing was "grep and squint"; now it's a single piped command, scriptable in CI.

## Knowing what got caught: `gate retro`

`gate scan` is the *before*. `gate retro` is the *after*. A deterministic filter has the rare property that you can count exactly what it did — there is no sampling, no model-of-the-month estimating, no confidence interval. Once `gate` has been running for a sprint or a quarter, `retro` reads the local JSONL stats log and prints the tally:

![gate retro output showing all-time statistics](/images/introducing-gate/retro.jpg)

It answers the question every security team asks of an opt-in filter on the hot path: *is this thing actually doing something, or a placebo we forget about?* The numbers come from the same pipeline that did the redaction — no sampling, no estimate. It also surfaces config gaps: a `Hit rate` of 0% on a tool that should be returning PII usually means the data is coming back in a format `gate` couldn't parse — CSV without a `pipe:`, an unusual JSON envelope, a column-name convention not yet in config. A surprising sub-category in the breakdown — `tax_id` showing up in a table you forgot was joined — is a finding worth tracing.

Stats are local-only: they're written to a JSONL log on disk, never sent anywhere, and `stats.enabled: false` turns the recording off entirely.

## Honesty about the gaps

The full threat model is in the [repo](https://github.com/GaaraZhu/gate/blob/main/THREAT-MODEL.md) but the headlines are:

- **`gate` is not a sandbox.** It only filters commands explicitly listed in `tools:`. Anything else passes through.
- **The adversary model is an inadvertent agent, not a malicious one.** `sudo gate protect` (Unix) chowns the config to root so a hijacked agent can't disable gate via config edits, but a jailbroken agent that deliberately base64-encodes data, requests CSV output, or exfiltrates through a non-intercepted tool is still out of scope. Combine `gate` with harness-level tool restrictions if you need that boundary.
- **Value regex is narrow.** Email, US SSN (dashes required — `123456789` slips), US phone, payment cards (via Luhn). Everything else — IBAN, passport, NHS number, NZ IRD, AU TFN — is column-name-based. If the column has an unusual name and isn't in your config's `column_names` list, the value will pass through. Configure for your region.
- **MCP `resources/read` and `prompts/get` are not redacted.** Only `tools/call` responses go through the scanner.
- **Non-JSON output is not redacted.** If a tool emits CSV or plain text, configure a `pipe:` to convert it (the example config uses `jq -c .` for curl and a 3-line Python `csv.DictReader` for `psql --csv`).
- **Disable mechanisms exist.** `enabled: false` in config, deleting the config file, or removing the hook entry from the harness settings. `sudo gate protect` (Unix) chowns the config to root to block the first two from inside the agent, but the harness settings file is still user-writable.

If any of these are deal-breakers, the tool is honest about it up front. Better than discovering it in a post-mortem.

## Why a deterministic CLI and not "just ask the model"

It is technically possible to ask the model to redact its own input before it ingests it. People are building this. I chose not to, for three reasons:

1. **Cost.** Every query result would round-trip through a model call. A single agent session might run hundreds of queries.
2. **Latency.** A hook on every Bash command needs to return in single-digit milliseconds. `gate hook`'s passthrough path is in that ballpark; an LLM call is not.
3. **Auditability.** "Why was this field redacted?" needs an answer that survives review. A regex and a tokenizer can be inspected, golden-file tested, and re-run on the same input forever. A model in 2026 will not give the same output on the same input in 2027, and you will not get a stack trace.

Existing PII tools (Presidio, Nightfall, Skyflow) take the opposite trade — they're mostly built for data-pipeline or SaaS-gateway use, sitting at API boundaries or in batch jobs, not in the agent's tool-execution path with single-digit-ms latency and harness-level hook enforcement. `gate` is shaped specifically for that boundary.

## What it costs to try

```bash
# macOS / Linux via Homebrew
brew tap GaaraZhu/gate && brew install gate

# or cargo binstall / direct download — see the README
```

```bash
gate config           # creates ~/.config/gate/config.yaml in your editor
gate init             # registers the PreToolUse hook in ~/.claude/settings.json
gate init --wrap-mcp  # dry-run: shows which MCP servers would be wrapped
gate validate         # compiles all regex patterns, lints the config
```

Pipe your schema into `gate scan` first to see what you're working with. Run `gate disable` to turn it off if you need to debug something, and `gate enable` to switch it back on. `gate uninstall` removes everything `gate` added to your system and asks for confirmation before each step.

## Where this is going

What's next, roughly in priority order:

- **More built-in patterns by region.** The value-regex coverage is US-centric; community PRs adding IBAN, passport, NHS, IRD, TFN, Aadhaar, etc., are explicitly welcome.
- **MCP `resources/read` redaction.** Closing the one documented gap in the MCP path.
- **More harness integrations.** Claude Code, OpenCode, and Copilot CLI are in. Cursor, Aider, and others are open questions — file an issue if you want one.
- **Write-path inspection.** Today `gate` only sees query results; `INSERT`/`UPDATE`/`DELETE` are not inspected. There's a plausible v0.9 line for blocking writes that target redacted-fielded tables.

If you've been holding off on connecting your AI agent to a real database because "what the model sees" was a vibes-based decision, this is the layer that turns it into a config file. Try it, scan your schema, and share what you find. The repo is [github.com/GaaraZhu/gate](https://github.com/GaaraZhu/gate). The issue tracker is open. The license is MIT.

I'd rather hear "gate redacted something it shouldn't have" than "gate let something through that it shouldn't have." If you find the second one, that's a security bug and there's a [process for it](https://github.com/GaaraZhu/gate/blob/main/SECURITY.md).
