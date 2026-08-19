# TraceGuardAI

**A red-team harness for RAG and agentic AI systems.** Poison the corpus, not the context — then prove what got through.

[![License: MIT](https://img.shields.io/badge/License-MIT-black.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/Python-3.11%20|%203.12%20|%203.14-3776AB.svg)](#tech-stack)
[![Tests](https://img.shields.io/badge/tests-333%20passing-4a7a52.svg)](#testing--evaluation)
[![Blind holdout](https://img.shields.io/badge/blind%20holdout-38.5%25%20–%2088.9%25-5b3ef0.svg)](#the-two-numbers)

> **Synthetic data only.** No real patient data, no external model, no real-world action. Every "clinical" scenario is a fixture.

---

## The problem

Most tools test prompt injection by **pasting the attack into the model's context.** Real attacks don't get that shortcut — a poisoned document has to survive indexing and retrieval before it reaches the model at all.

TraceGuardAI poisons the corpus instead, then tests whether the attack survives the actual retrieval path. When one succeeds, a **canary token** planted in the document appears in the output — evidence the whole path completed, not an inference from how an answer was worded.

---

## The two numbers

This project red-teamed itself and published the result.

A blind holdout — attacks written independently, from the threat model, by someone who never read the detector source — scored **25% recall** against a suite of 179 passing tests. The detectors were matching the phrasings they were written from, not the techniques behind them.

After two rounds of rebuilding, the same holdout scored **88.9%**. A second holdout, fresh wordings for the same techniques, scored **38.5%**.

**The gap is the finding.** The detector generalises unevenly. Quoting only the high number would overstate it; quoting only the low one would understate it. Both are true.

| | Measured against | Citable? |
|---|---|---|
| **88.9% / 38.5%** | Two independently authored holdouts | ✅ the honest signal |
| **79.5%** | This project's own corpus | ❌ inflated by construction |

*Measured 2026-08-17 at commit `7f1078d`. Reproduce with `python holdout_eval.py` and `holdout_eval_2.py`.*

![Dashboard](docs/screenshots/dashboard.png)

*Counts, never an accuracy percentage — and the page says plainly that its own numbers are not the citable ones.*

---

## What it does

| | |
|---|---|
| 🎯 **Retrieval fidelity** | Attacks travel the real retrieval path, not injected context |
| 🐤 **Canary proof** | A planted token in the output is evidence, not inference |
| ⚖️ **Benign corpus** | False positive rate measured alongside recall — most tools report only recall |
| 🔍 **Auditable trace** | Ten-stage execution timeline and provenance verification per run |
| 🧪 **Mutation engine** | 91 base cases expand to 1,200+ through 17 seeded evasion mutators |

![Attack simulations](docs/screenshots/attacks.png)

*Expected against actual, with a derived verdict. The benign control is separated deliberately — a failure there is a real user being blocked, which is as much a failure as an attack getting through.*

---

## How detection works

```mermaid
flowchart LR
    A["Query + retrieved docs"] --> N["Normalizer"]
    N --> D["Seven detectors"]
    D --> P["Scored policy engine"]
    V["Provenance verifier"] --> P
    P --> R["Blocked / allowed<br/>+ rationale"]
    R --> T["Trace log"]
```

**Normalization** folds input to canonical form before any matching — NFKC, homoglyph folding, zero-width and BiDi stripping, spaced-run collapse, leetspeak, and bounded base64/hex/ROT13 decoding with wall-clock and attempt caps so a crafted decode bomb can't stall a request. A digit-preserving view runs alongside so numeric identifiers survive folding.

**Seven detector families** match technique *families* through windowed component co-occurrence, not literal payload strings:

| Family | Catches |
|---|---|
| `instruction_override` | overrides, jailbreak framing, prompt extraction — 11 languages |
| `structural_injection` | fake role blocks, delimiter-transition injections |
| `exfiltration` | DNS/subdomain, external forms, recipient routing |
| `pii` | extraction requests, and identifiers reaching output |
| `unauthorized_action` | self-approval, delegated authority, bypass-review |
| `tool_poisoning` | covert calls, deprecation-bypass, omit-from-summary |
| `source_confusion` | recency claims, self-certified authority |

Findings are **origin-aware**: the same payload in retrieved context scores higher than in a direct query, and PII in model output is a breach rather than data at rest.

**The policy engine** weights findings into a score under a tunable pack — `strict` / `balanced` / `permissive`. A run blocks on a severity trigger **or** a score trigger. An untrusted source adds score and escalates its own findings, but never blocks alone. Every decision carries a plain-English rationale, and the score is derivable on screen — every point accounted for.

![Trace log](docs/screenshots/traces.png)

*Every stage inspectable, and the reasoning stated rather than implied: an untrusted source contributed to the score without blocking on its own, and provenance failure is recorded as a scored signal rather than an automatic block.*

---

## Trust is explicit

Every uploaded document starts **untrusted**. Marking one trusted changes how findings from it are scored — which means it changes what the tool concludes. That toggle is deliberately consequential, because a poisoned document sitting in an otherwise trusted corpus is the attack this project was rebuilt around.

A source can be in the index and still not be trusted to ground an answer. That case — present but not authoritative — is the ordinary one, not the edge case.

![Documents](docs/screenshots/documents.png)

---

## Getting started

You need **two terminals** — both servers run in the foreground.

```bash
git clone https://github.com/sabasiddique1/TraceGuardAI.git
cd TraceGuardAI
```

**Terminal 1 — backend**

```bash
cd backend
python3.11 -m venv .venv          # any Python 3.11+ works
source .venv/bin/activate         # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m uvicorn app.main:app --reload
```

Check: [localhost:8000/health](http://localhost:8000/health) → `{"status":"ok","service":"TraceGuardAI"}`
Anything else means another app holds port 8000 — see [Ports](#ports).

> ⚠️ **The venv is required, not advice.** Without it, `pip install` writes into whatever `python3.11` resolves to. On some setups that fails loudly and you lose nothing. On others — a Homebrew Python without the `EXTERNALLY-MANAGED` marker — **it silently succeeds and rewrites your shared system interpreter.** The dangerous case is the one that succeeds.
>
> It also keeps the pins honest: outside a venv, tests run against whatever happens to be installed, so a green suite proves less than it looks like.

**Terminal 2 — frontend**

```bash
cd frontend                       # from the repository root
npm install
npm run dev
```

App: [localhost:3000](http://localhost:3000) · Node 20+

Once the venv is active, plain `python` is the venv's own — which is why every later command uses `python` rather than a version-specific name.

**These exact steps are verified on every push** that touches this section, `requirements.txt`, or `package.json`. `scripts/verify-clean-clone.sh` runs them from a genuinely fresh clone in CI and is the source of truth if this page and reality disagree.

### Ports

Backend expects **8000**, frontend **3000**.

```bash
lsof -nP -iTCP:8000 -sTCP:LISTEN
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

If 8000 is taken, move the backend and tell the frontend where it went:

```bash
python -m uvicorn app.main:app --reload --port 8100          # terminal 1
NEXT_PUBLIC_API_URL=http://localhost:8100 npm run dev        # terminal 2
```

`NEXT_PUBLIC_API_URL` is read at **build** time too, so for `npm run build && npm run start` it must be set on the *build*.

---

## Using it

| Screen | What it's for |
|---|---|
| **Attacks** | Run a scenario, see expected vs actual, and why the guardrail decided |
| **Lab** | Watch the ten-stage pipeline unfold, step by step |
| **Documents** | Upload a `.txt`/`.md`, review its scan, manage trust |
| **Traces** | Every run's execution timeline, payloads redacted |
| **Provenance** | Which cited sources exist, which are trusted, and why that matters |
| **Dashboard** | Seed-scenario summary — a baseline, not a quality score |

---

## Testing & evaluation

```bash
./scripts/run_all_checks.sh         # pytest + frontend build
```

The detector is measured against sets it was **not** written to fit:

```bash
cd backend && source .venv/bin/activate
python holdout_eval.py              # blind holdout, independently authored
python holdout_eval_2.py            # a second, fresh one
python -m redteam.build_corpus      # write the red-team corpus
python -m redteam.run_corpus        # expand + sliced metrics
```

```mermaid
flowchart LR
    C["56 attacks + 35 benign"] --> M["17 seeded mutators"]
    M --> X["1,200+ cases"]
    H["Blind holdouts"] --> P["Detection pipeline"]
    X --> P
    P --> R["recall · ASR · FPR<br/>precision · F1"]
```

Metrics slice by category, OWASP class, mutation family and delivery vector. The local-target score is a **self-referential baseline** — corpus and detector share an author. The blind holdouts are the honest signal.

---

## Tech stack

**Frontend** — Next.js (App Router), TypeScript, Tailwind, React Aria Components, tailwind-variants, Recharts, Zod
**Backend** — FastAPI, Python 3.11+, Pydantic, pytest · 333 tests verified on 3.11, 3.12 and 3.14 against the pinned `requirements.txt`
**Data** — JSONL seeds; SQLite for uploaded documents only
**CI** — GitHub Actions with `pip-audit` as a hard gate, plus a clean-clone job

---

## Configuration

All environment variables are optional; the defaults are what you get by following [Getting started](#getting-started). Full reference: [`docs/configuration.md`](docs/configuration.md).

**This runs on localhost.** Uvicorn binds `127.0.0.1`, CORS is pinned to localhost origins, and there is **no authentication** — the security boundary is the machine. Two variables are security-relevant if that changes:

| Variable | Default | Why it matters |
|---|---|---|
| `TRACEGUARD_RAW_TRACE_PAYLOADS` | `false` | Returns trace payloads unredacted. A local-debugging affordance, **not** an access control — with no auth, any caller could read them |
| `TRACEGUARD_TRUST_PROXY` | `false` | Reads client IPs from `X-Forwarded-For`. When `true` without a real proxy, callers choose their own rate-limit identity |

If you bind this beyond localhost, all of the following become true at once: anyone reachable can upload documents, change document trust (which changes what the tool concludes), and delete uploads. None of that is a bug — it's what a localhost tool is. It becomes a problem only if moved somewhere it wasn't designed for.

---

## Limitations

Stated plainly, because a security tool that overstates its scope has already told you how much to trust its numbers.

- **Multilingual detection is vocabulary-bound.** Eleven languages, but a fixed verb list — it misses negation-based and idiomatic overrides. Turkish, Indonesian and Bengali are out of scope.
- **Four detector families depend on semantic intent** — source confusion, unauthorized action, instruction override, exfiltration — and are weakest against independently authored wordings. That gap is why pattern work was stopped rather than tuned further.
- **Nested encoding stops at depth three**, deliberately.
- **The external-target harness is command-line only.** This UI drives the local synthetic pipeline; scanning someone else's RAG or agent runs from the terminal.
- **No real LLM, no vector database, no persistence** beyond uploaded documents. Retrieval is deterministic scenario routing — which is what makes a run reproducible from a seed.
- **Provenance uses synthetic source IDs**, not cryptographic verification.
- **No authentication.** Deliberately — auth would introduce the first genuinely sensitive data this project holds, and on localhost it would guard a door in a building with no walls. The reasoning and the trigger to revisit are recorded in the auth directory's README.

---

## Roadmap

- Expose the red-team harness over HTTP — scan history, external targets, guarded-vs-unguarded comparison
- A semantic layer behind the pattern fast-path, for the four intent-dependent families
- Multi-turn / crescendo attack orchestration
- Benchmark runs against BIPIA, AgentDojo and NotInject
- PyPI release and contributor onboarding

---

## License

[MIT](LICENSE) © 2026 Sabaa Siddique