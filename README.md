# Degree Requirements JSON — Engineering README

## 1. What this JSON is for

Each file describes the requirements for **one academic program** (degree, major, minor, emphasis, or certificate) in a machine-checkable form. The degree-audit engine loads one file per program and evaluates a student's completed and in-progress courses against it to determine, requirement by requirement, what is **satisfied**, what is **still owed**, and what needs a **human to review**.

One file = one program + one catalogue year. The JSON answers five questions for the engine: which courses can count, how each one counts, the minimum threshold to satisfy each requirement, which special rules affect validation, and when/where the data was scraped.

## 2. The core abstraction: a weighted threshold over an eligible set

Almost every course requirement is the same shape:

- `S` — the set of **eligible courses**
- `w_i` — the **weight** of course *i* (a course count of `1`, or its unit value)
- `x_i` — `1` if the student completed eligible course *i*, else `0`
- `T` — the **threshold**

A requirement is **satisfied** when:

```
Σ (w_i · x_i) for all i in S   ≥   T
```

That is the whole model. Three common cases, one formula:

| Case | `S` | `w_i` | `T` |
|---|---|---|---|
| Fixed required course | one named course | `1` | `1` |
| Choose N from a set | explicit list | `1` per course | `N` |
| Unit-threshold pool | attribute-matched pool | course units | required units |

So `BUAD 311 required`, `choose 2 of {ECON 351x, ECON 352x, …}`, and `12 units of DSO 300–499` are **not three special cases** — they are one rule with different `S`, `w`, and `T`. Do not special-case them in code.

## 3. How the JSON encodes the three cases

Inside each requirement object:

- **`eligible_courses` defines `S`**, in one of two modes:
  - `explicit_list` — the catalogue names exact courses (fixed course, choose-N).
  - `attribute_pool` — the catalogue describes a pool by `subject_prefixes` + `course_number_min/max` + attributes (the Marshall DSO pattern).
- **`weight` defines `w_i`** — `measure: "course_count"` (value `1`) or `measure: "units"` (looked up per course).
- **`completion_threshold` defines `T`** — `{ measure, value, operator }`.

## 4. How the engine evaluates one requirement

1. **Resolve `S`.** From `explicit_list` `course_id`s, or by querying the course DB for courses matching the `attribute_pool` (prefix ∈ list, number in range, attributes match, minus `excluded_courses`).
2. **Intersect** `S` with the student's completed/in-progress courses → the courses they actually have.
3. **Compute `w_i`** per matched course:
   - `course_count` → `1`.
   - `units` → use `weight_override`/source `units` if present, else **look up units in the course DB**.
4. **Sum** `w_i · x_i`.
5. **Compare** the sum to `completion_threshold.value` using `completion_threshold.operator`.
6. **Apply relevant `global_rules`** (remove double-counted courses, check grade floors/GPA/caps where in scope) — §6–§7.
7. **Emit** a per-requirement result: `satisfied | in_progress | not_satisfied | needs_manual_review`, plus units/courses still owed.

## 5. What lives here vs. in the course database

This file is **not** a course catalog. It references `course_id` and defers course facts to a separate course DB.

| Degree JSON (this file) | Course DB (separate) |
|---|---|
| Which courses count, how they count, the threshold | `course_id`, title, **units**, subject prefix, course number |
| Program-level rules (GPA, residency, caps, double-counting) | description, **prerequisites**, **cross-listings**, repeatability |
| Scrape metadata + per-requirement source evidence | grading restrictions, catalogue metadata |

**Units rule:** if the source *explicitly prints* a unit value, store it as `units` / `weight_override`. Otherwise set the weight's `value_source: "course_catalog_lookup"` and `unit_lookup_required: true` so the engine fetches it. Never hard-code a unit value the source didn't state.

## 6. V1 scope (build the engine logic now)

- Fixed required courses, choose-N, unit-threshold pools
- `explicit_list` and `attribute_pool` eligible sets
- Prefix + course-number-range pools (with `course_level_label` for readability)
- `course_count` and `units` weights; thresholds with `operator`
- Scrape metadata + per-requirement `source_evidence`
- Manual-review flags + top-level `manual_review_items`
- Non-double-counting rules (reference requirement IDs)
- Cross-listed / equivalent-course handling
- Catalogue-year versioning

## 7. V2 hooks (sections exist now; full engine logic later)

GPA gates, grade floors, P/NP caps, standing gates, residency gates, prerequisite chains, gateway/declaration rules, external substitutions, graduate-unit borrowing, exam-credit rules, course-repetition rules, degree-combination restrictions, exceptions/waivers, GE overlap rules.

Their arrays exist in `global_rules` from day one. In V1 they may be empty, or populated with `engine_behavior` describing **what to do later**; an entry with `manual_review_required: true` tells V1 to **flag, not auto-evaluate**. Including the sections now means **no schema redesign** when the logic ships.

Distinctions the engine must respect:

- **Gateway ≠ fixed course.** A missing fixed course = "still owed." A missing gateway = a **declaration/eligibility blocker** the engine must surface separately.
- **Graduate-unit borrowing is a cap, not a minimum.** Model it as a ceiling, never as a threshold requirement.
- **External substitutions** (e.g. an exchange program) can't be auto-checked from transcript lines → always `manual_review_required: true`.
- **"or approved equivalent"** → list known `equivalent_course_ids`; if the equivalent is unknown, flag for manual review.

## 8. Uncertainty, missing evidence, and manual review

- Every requirement carries `source_evidence` (URL + quote + capture date). Use a **clearly labeled placeholder** if not yet captured — never leave it blank.
- Anything ambiguous, unverified, or human-only → set `manual_review_required: true` **and** add an entry to top-level `manual_review_items`.
- `scrape_metadata.confidence` records overall trust in the file; `validation_notes` records the pre-ship self-check and open questions.
- **The engine must fail loud, not silent.** When a requirement can't be evaluated with confidence, return `needs_manual_review` — never a false `satisfied`. A silent pass that graduates an unqualified student is the failure mode this whole design exists to prevent.
