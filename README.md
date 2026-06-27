# quality-gate

The independent **proof-of-correctness gate** between "built" and "shipped" — the station that makes **build-loop** ship solid, secure software instead of just moving cards.

It spawns a **fresh agent (never the builder)** to:

1. Write tests against the issue spec
2. Run the app to observe real behavior
3. Run an adversarial `code-review`
4. Run a two-lens security pass — `security-review` plus `vibe-security` for the AI-introduced vulnerability classes

…then returns a **PASS/FAIL** verdict with a short, readable report. On FAIL it files the findings back as fixes; only a real PASS lets a card reach the In-review column.

## Why it exists

The same agent that writes a bug will write a passing test and call the code secure. The checker must be **independent and adversarial** — so this skill never uses the builder.

## Triggers

When build-loop finishes an issue and is about to move it to the In-review column, or when you say: "is this actually tested", "prove it works", "run the quality gate", "verify this branch/PR", "is this secure", "gate this before I merge".

## Companions

**build-loop** (which calls it) and **design-queue** (whose issue body is the spec it tests against). It never designs and never decides scope — it only proves.

## Install

```powershell
git clone https://github.com/iwantedjusttom/quality-gate.git
New-Item -ItemType Junction -Path "$HOME\.claude\skills\quality-gate" -Target "<path>\quality-gate"
```
