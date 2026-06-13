---
name: quality-gate
description: The independent proof-of-correctness gate between "built" and "shipped" — the station that makes build-loop ship solid, secure software instead of just moving cards. Use this skill whenever a feature/fix needs to be PROVEN before it advances: when build-loop finishes building an issue and is about to slide it to `in-review`, or when Tom says "is this actually tested", "prove it works", "run the quality gate", "verify this branch/PR", "is this secure", "gate this before I merge", or wants confidence in a change before it ships. It spawns a FRESH agent (never the builder) to write tests against the issue spec, runs the app to observe real behavior, and runs adversarial `code-review` + a two-lens security pass (`security-review` plus `vibe-security` for the AI-introduced vulnerability classes) — then returns a PASS/FAIL verdict with a short, readable report. On FAIL it files the findings back as fixes; only a real PASS lets a card reach `in-review`. Its whole reason to exist: the same agent that writes a bug will write a passing test and call the code secure, so the checker must be independent and adversarial. Companion to build-loop (which calls it) and design-queue (whose issue body is the spec it tests against). It never designs and never decides scope — it only proves.
---

# Quality Gate — prove it before it ships

build-loop moves work; this skill proves it. It is the missing station between **built** and **`in-review`** — the one that asks not "did the agent finish?" but "is this actually correct, secure, and doing what the spec said?" Without it the pipeline optimizes *flow*: a card slides, a PR opens, a label changes — all green, none of it proven. This skill optimizes *truth*. A feature does not reach `in-review` until it has earned it here.

## The one rule everything rests on: the checker is never the builder

The agent that wrote the code will write a test that confirms its own bugs and pronounce the result secure — confidently, every time. A green check it produced for itself is worth nothing and looks identical to a real one. So **every station in this gate runs in a FRESH agent / fresh context**, blind to the builder's shortcuts and told to be adversarial — to *break it*, not to bless it. Independence is not a nicety here; it is the entire point. If you find yourself letting the building agent grade its own work, stop — you've defeated the gate.

## The four stations

Run against the feature branch, with the **issue body as the spec** (design-queue wrote it; it says what "correct" means). For each, spawn a fresh sub-agent.

1. **Tests from the spec — not from the code.** A fresh agent reads the *issue spec* (what it should do, its edge cases) and writes tests that assert **that behavior**, deliberately not reading the implementation first so it can't be led into testing the bug as if it were the feature. Then it runs them. Tests that pass by asserting nothing, or that just mirror the implementation, are theater — reject them. If the repo has no test runner yet, this station establishes the minimum one (its own small, real win toward Tom's goal of building the *right* tests).
2. **`verify` — run it and watch.** Invoke the `verify` skill: actually launch the app and observe the real behavior the issue promised, in a browser/CLI — not an assumption that it works. Behavior observed > behavior assumed.
3. **`code-review` — adversarial correctness.** Run `code-review` on the diff at a real effort level, framed to hunt for breakage, not to nod. Edge cases, error paths, the regression an innocent change quietly caused.
4. **Security — two complementary lenses, not vibes.** Run **both** passes in the fresh agent, because they catch different things:
   - **`security-review`** — general adversarial reasoning over the diff's actual attack surface: auth/ownership, input handling, secrets, any boundary the feature touches.
   - **`vibe-security`** — a systematic sweep of the specific vulnerability classes AI assistants reliably introduce (exposed `service_role`/keys, broken Supabase RLS, client-trusted prices, missing auth validation, unrate-limited AI/expensive calls). It operationalizes the project's standing invariants — **RLS on every table, service-role key server-side only, mobile-first** — as a concrete checklist rather than leaving them to memory.

   Together they are the security verdict; "looks fine" is not. **Gating rule:** any `vibe-security` finding at **Critical or High** is an automatic FAIL — those are the breach-grade classes (a leaked service key, a $0.01 checkout, a missing RLS policy), not nits. Medium/Low get reported in the verdict and fixed if cheap, but don't have to block on their own.

## The verdict — PASS advances, FAIL self-heals then escalates

The gate returns one of two things, plus a short report.

- **PASS** → the feature is proven. build-loop may now open/keep the PR and slide the card to `in-review`. The gate's report goes on the issue as a comment so the proof is part of the permanent record.
- **FAIL** → it does **not** advance. First, **post the findings as a comment on the GitHub issue** so there's a durable record of what failed — one entry per finding, each with the station that caught it (test / verify / code-review / security), a one-sentence root cause, and the **`file:line` location(s)** so the builder or whoever picks it up later knows exactly where to go. That comment is the permanent FAIL record; the same findings are then handed to the builder as fixes on the same branch, who addresses them and re-runs the gate. Up to **two self-heal rounds** unattended; if it still can't pass, **stop and surface it to Tom** with exactly what's failing and why — that's a real judgment call, not a loop to grind. Don't paper over a genuine failure to make the card move; a stuck card is information.

Never advance on a partial pass. "Tests green but security flagged an open RLS hole" is a FAIL.

## Scale the gate to the change — don't full-audit a copy tweak

The gate is tunable, not ceremony. Match the rigor to the surface area:
- **Copy / styling / a self-contained UI tweak** → `verify` + a light `code-review`; skip the test-authoring and security stations.
- **New data, an endpoint, anything touching auth/ownership/money** → the full four stations, security non-optional.
- **A bug fix** → a test that *reproduces the bug first* (proves it was real), then the fix turns it green — plus whatever station the bug lived in.

When in doubt on something that touches user data or auth, run the full gate. When it's plainly cosmetic, don't make a federal case of it.

## How build-loop calls it (the seam)

build-loop owns *building*; this skill owns *proving*; they stay separate skills that hand off — same split as design-queue/build-loop. The seam is a single insertion in build-loop's finish step: **build → call quality-gate → on PASS, open the PR + relabel `in-review` + slide the card; on FAIL, fix and re-gate.** build-loop never grades its own build; it delegates that here and trusts the verdict. (One-line hook to add when Tom's ready: between "Build to the spec" and "Finish," run `quality-gate` on the branch and only proceed on PASS.)

## Standalone use

It's also a thing you run directly: `/quality-gate` on the current branch, a named branch, or a PR — "prove this before I merge," "is this secure?", a confidence check on a change that didn't come through build-loop. Same four stations, same verdict; it just isn't being called by the builder.

## The report is for Tom — it's how the judgment grows

End every run with a **short, readable verdict** (not a wall of logs): what was checked, what's substantive vs. noise, and the one or two things that actually mattered. This is the point where Tom gets to *read a verdict and judge whether it's real* — the literate-not-fluent muscle — without reading the diff himself. When a station finds something non-trivial, state the **root cause in a sentence**, not just "fixed it." Over time that report is how an orchestrator who doesn't write the code still develops the instinct to smell when a check is lying.

## Don't over-build it

Like build-loop, the gate is worth only what flows through it. Four stations, two self-heal rounds, scale-to-change, one verdict — that's the whole thing. Don't grow it into a CI cathedral, don't add stations for failure modes you haven't actually hit, don't let it become slower than the build it guards. A lean gate that genuinely proves correctness beats an elaborate one that's always being tuned. Its value is that Tom can trust a green card — not that it's clever.
