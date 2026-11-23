# ASPA BGP Checker

**Offline ASPA-Based BGP Route Leak Detection, Export Policy Verification and Safe ASPA Change Planning**

CS6198 Internet Engineering and Operations course project
by **Devender** (Roll No. 264101018) and **Gunjan Naruka** (Roll No. 264101021).

---

## 1. What this project does

The tool answers three questions about BGP route leaks, using synthetic data only:

1. **Is this export safe?**
   Given a small network, its ASPA records, a route and a proposed export
   ("AS X wants to send this route to neighbour Y"), the tool checks the export with
   **two independent layers** and explains the result:
   * **Layer 1, ASPA verification** (receiver's view): would Y's ASPA check accept the AS_PATH?
     Implements draft-ietf-sidrops-aspa-verification-31.
   * **Layer 2, export policy** (exporter's view): is X allowed to send this route to Y?
     Valley-free rules, BGP Roles and Only-To-Customer (OTC) from RFC 9234.

   The two verdicts are combined into one overall verdict: **VALID**, **INVALID** or **UNKNOWN**.

2. **What is hidden behind an UNKNOWN?**
   When records or relationships are missing, the **Unknown-resolution engine** fills in every
   possible value, re-runs the real engines, and reports whether the route is
   **LATENT-VALID**, **LATENT-INVALID** (a leak that is accepted today) or **UNDETERMINED**,
   which missing facts matter most (influence) and the smallest set that would settle the verdict.

3. **How can I change my ASPA record safely?**
   The **change planner** finds an order of ASPA publications, BGP changes and waits that
   never rejects a legitimate route while validators hold different record versions,
   and compares it with the naive and the make-before-break approaches.

**Safety:** the tool is completely offline. It never connects to routers, never changes
routing tables and never blocks real announcements.

---

## 2. Requirements and installation

* Python 3.9 or newer
* No third-party packages for the tool itself (standard library only)
* `pytest` for the tests

```bash
cd aspa-bgp-checker
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt    # installs pytest
```

---

## 3. Quick start

```bash
python3 main.py --input examples/valid.json            # a valid export
python3 main.py --input examples/invalid_aspa.json     # a leak caught by ASPA
python3 main.py --input examples/invalid_policy.json   # an export forbidden by policy
python3 main.py --input examples/unknown.json          # missing record -> UNKNOWN + analysis
python3 main.py --input examples/malformed.json        # rejected input (exit code 2)

python3 main.py --input examples/changes/first_publication.json --plan   # change planner
python3 -m pytest -v                                                     # all 194 tests
```

---

## 4. Usage

### 4.1 Check one export

```bash
python3 main.py --input <scenario.json>            # readable report
python3 main.py --input <scenario.json> --json     # machine-readable JSON
python3 main.py --input <scenario.json> --no-color # plain text (no colours)
```

### 4.2 Check a whole folder

```bash
python3 main.py --input tests/fixtures --summary
```

Prints one line per scenario: ASPA, policy and overall verdicts, the Unknown class, and the
expected result for fixtures.

### 4.3 Plan an ASPA change

```bash
python3 main.py --input examples/changes/transit_to_peering.json --plan
python3 main.py --input examples/changes/transit_to_peering.json --plan --json
```

### 4.4 Use your own data

Copy and edit the templates in `examples/templates/`:

```bash
python3 main.py --input examples/templates/my_route.json          # your own export
python3 main.py --input examples/templates/my_change.json --plan  # your own change
```

Every field is explained in `examples/templates/README.md` and in
[`docs/user_guide.md`](docs/user_guide.md).

### 4.5 Exit codes

| Code | Meaning |
|---|---|
| 0 | the check ran (whatever the verdict) |
| 2 | the input file is missing, is not valid JSON, or fails validation (`INPUT ERROR: ...` names the field) |

---

## 5. Example output (shortened)

```
 7. ASPA VERIFICATION: VALID   (upstream verification: VALID)
      - AS100 receives from its customer AS200 -> upstream algorithm (Sec. 5.5)
      - every hop is customer->provider and attested (Provider+), so the path is a pure up-ramp
      Rule: Sec. 5.5 step 6: ramps cover the path
 8. EXPORT POLICY VERIFICATION: VALID   (export permitted)
      - valley-free check: VALID -- RFC 9234 Sec. 3.1: a route learned from a customer
        may be sent to any neighbour
      - OTC check: PASS (received without OTC)
10. OVERALL RESULT: VALID
11. Explanation: Both layers ran with sufficient information and both passed.
```

Every verdict names the deciding hop, the ASPA record behind it, the rule and the
specification section.

---

## 6. How it works

```
 scenario.json
      |
   parser ------------> InputError (exit 2) for malformed input
      |
   topology + receiver-visible path
      |
      +--> Layer 1: ASPA verification (receiver's view)   --+
      |                                                     +--> aggregation --> explanation
      +--> Layer 2: export policy + OTC (exporter's view) --+          |
                                                                       v
                                                     Unknown-resolution engine (if UNKNOWN)

 change.json --> change planner (route simulation, validator views, Dijkstra search)
```

* **Layer 1** builds the AS_PATH the receiver will see (the exporter prepends itself),
  computes `authorized()` for every hop and the up/down ramp bounds, and runs the upstream or
  downstream procedure (both if the relationship is undeclared).
* **Layer 2** applies the valley-free export rules and the RFC 9234 OTC ingress and egress rules.
* **Aggregation** uses the precedence INVALID > NOT_IMPLEMENTED > UNKNOWN > VALID
  (so ASPA UNKNOWN + policy INVALID = INVALID).
* **Unknown resolution** enumerates up to 4096 completions exactly (otherwise 2000 random
  ones with a fixed seed) under two models: *any-record* and *topology-consistent*.
* **Change planner** simulates routes (Gao-Rexford), checks every state under every mix of
  old and new record versions, and searches for the fastest safe plan, then the one with least
  leak exposure.

Full procedures, definitions and proofs are in [`docs/`](#10-documentation).

---

## 7. Project structure

```
main.py                         command-line interface and the check() pipeline
core/
  models.py                     shared data types (scenario, records, verdicts, results)
  parser.py                     JSON loading and validation (InputError)
  topology.py                   declared relationships, route servers, complex roles, warnings
  path_builder.py               receiver-visible path, prepend compression, loop and AS_SET checks
  aspa/ramp_logic.py            authorized() and ramp bounds           (draft-31 Sec. 5.3-5.4)
  aspa/validator.py             LAYER 1: upstream / downstream ASPA    (draft-31 Sec. 5.5-5.6, 6.4)
  policy/valley_free.py         conventional export rules              (RFC 9234 Sec. 3.1)
  policy/otc.py                 OTC ingress / egress rules             (RFC 9234 Sec. 5)
  policy/export_policy.py       LAYER 2
  aggregation.py                combines the two layers
  explanation.py                text and JSON reports
  unknown_resolver.py           Unknown-resolution engine
  planner/routing.py            route simulation, leak probes, reachability
  planner/plan.py               change model, safety check, naive / make-before-break / search
  planner/loader.py             change-scenario loader
  planner/report.py             plan report
examples/                       teaching scenarios, change scenarios, templates for your own data
tests/
  fixtures/                     34 hand-derived scenarios with expected results
  test_core.py                  parser, paths, topology, aggregation, CLI
  test_fixtures.py              runs all fixtures + resolver properties
  test_planner.py               lemmas (random property tests), theorem, example plans
tools/
  make_fixtures.py              generates tests/fixtures (expected results derived by hand)
  make_change_examples.py       generates examples/changes
  evaluate_planner.py           96-scenario synthetic experiment
results/                        experiment CSVs and summaries
docs/                           design documentation (see section 10)
```

Every module, class and function has a docstring describing its purpose, inputs and outputs.

---

## 8. Testing

```bash
python3 -m pytest -v
```

**194 test cases, all passing:**

* **34 fixtures** (14 VALID, 13 INVALID, 7 UNKNOWN), each checked four ways: layer and overall
  verdicts, that every result is explained, the Unknown analysis (bounds, class, influence,
  settling sets), and that the resolver never contradicts a decided layer.
* Malformed input (9 kinds), the aggregation table (13 rows), the CLI, path building,
  relationships.
* The planner's lemmas on 4000 random cases each, the make-before-break theorem and the
  example plans.

Fixture expectations were derived by hand from the specifications before the code was run
(see `tools/make_fixtures.py`). All fixtures are marked **proposed** until reviewed.

---

## 9. Reproducing the experiment

```bash
python3 tools/evaluate_planner.py --runs 48 --seed 11 --adopt 0.3 --out results/planner_eval_adopt30.csv
python3 tools/evaluate_planner.py --runs 48 --seed 12 --adopt 0.7 --out results/planner_eval_adopt70.csv
```

Main results (96 synthetic changes on 19-AS topologies, Delta = 60 min, Beta = 5 min):

| Plan | Unsafe | Mean duration (30% / 70% adoption) | Mean exposure (route-min) |
|---|---|---|---|
| naive | 96 / 96 | 70 / 70 min | (unsafe) |
| make-before-break | 0 / 96 | 110 / 110 min | 320.8 / 144.9 |
| best (search) | 0 / 96 | 84 / 90 min | 162.1 / 87.0 |

The searched plan was never slower than make-before-break and faster in 37 of 96 scenarios.

---

## 10. Documentation

| File | Contents |
|---|---|
| [`docs/user_guide.md`](docs/user_guide.md) | input formats (all fields), output sections, worked usage |
| [`docs/standards.md`](docs/standards.md) | pinned documents and sections, design decisions Q1-Q8, modelling assumptions |
| [`docs/algorithm.md`](docs/algorithm.md) | every algorithm in procedure form, the Unknown-resolution theorem |
| [`docs/change_planner.md`](docs/change_planner.md) | planner problem, literature check, model, lemmas and proofs, evaluation, limits |
| [`examples/templates/README.md`](examples/templates/README.md) | short field guide for your own data |

---

## 11. Standards used

| Purpose | Document |
|---|---|
| ASPA path verification | draft-ietf-sidrops-aspa-verification-31 (Internet-Draft, pinned) |
| BGP Roles and OTC | RFC 9234 |
| Route-leak taxonomy | RFC 7908 |
| BGP loop detection | RFC 4271 Sec. 9.1.2 |
| AS_SET deprecation | RFC 9774 |
| Export conventions | Gao-Rexford valley-free model |

---

## 12. Limitations and work not completed

* Synthetic data only: no real RPKI or BGP data is read, and signature validity is a status field.
* Relationships must be declared; results are only as correct as the declared topology.
* ASPA is still an Internet-Draft; the code is pinned to revision 31 with section references.
* The planner supports provider-customer and peer links, uses one lag bound for all
  validators, and is limited to 5000 search states.
* **Not implemented:** the web playground (FastAPI + React) and ROA-based prefix-hijack detection.
  `check()` and the versioned JSON output are designed so that a web front end can reuse the
  engine without duplicating any verification logic.
