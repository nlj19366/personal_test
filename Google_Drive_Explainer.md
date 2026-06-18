# How the Degree-Audit Data Is Organized — and Why

*A plain-language explainer for advisors, faculty, project managers, and engineering leads. This document explains the reasoning behind the design. It is not the technical spec; the README covers implementation.*

---

## Step 1 — The problem we are solving

We are building a system that checks, automatically, whether a student has met every requirement for a degree, major, minor, emphasis, or certificate. To do that, the system needs each program's requirements written down in a form a computer can read and check — not a PDF worksheet, not a paragraph on an advising page, but structured data.

The hard part is that USC has hundreds of programs, and their requirements are written in many different styles. We need one consistent structure that can represent **all** of them, so we don't rebuild the system for every school.

## Step 2 — Why a flat course list fails

The tempting shortcut is to store each program as a simple list: "these are the courses; did the student take them?" That breaks almost immediately, because real requirements are rarely a flat list:

- Some requirements say **"take this exact course."**
- Some say **"choose two from this list."**
- Some say **"complete twelve units of upper-division coursework in this department"** — and never name a single specific course.

A flat checklist can't express "choose two" or "twelve units of anything matching a pattern." Worse, it fails *silently*: it might quietly pass a student who hasn't actually met a requirement, or block a student who has. For something that determines whether a person graduates, silent errors are the worst possible outcome.

## Step 3 — The structure we use instead: eligible courses + weights + thresholds

Every course requirement, no matter how it's worded, is really answering three questions:

1. **Which courses are allowed to count?** (the *eligible set*)
2. **How much is each one worth?** (the *weight* — either "one course" or "its number of units")
3. **How much is enough?** (the *threshold*)

A requirement is met when the courses the student took, multiplied by what each is worth, add up to at least the threshold. That's it. Three questions, one rule.

## Step 4 — Why fixed courses, electives, and unit pools are the same thing

This is the key insight that makes the whole system manageable. Those three requirement styles from Step 2 are not three different problems — they are the *same* structure with different settings:

| The requirement says… | Eligible courses | Each worth | Threshold |
|---|---|---|---|
| "Take BUAD 311." | just BUAD 311 | 1 course | 1 course |
| "Choose two from this list." | the named list | 1 course each | 2 courses |
| "Twelve units of upper-division DSO." | any DSO course numbered 300–499 | its units | 12 units |

Because all three fit one pattern, engineers write the checking logic **once** and it handles every variation. We don't need custom code for each major. New programs are just new data, not new software.

## Step 5 — Why course details live in a separate database

You'll notice the requirements file mostly refers to courses by an ID, and doesn't repeat things like the course title, its unit value, its prerequisites, or its description. That's deliberate.

Those facts about a course belong in **one** place — a separate course database — for the same reason you keep one address book instead of rewriting a friend's phone number into every document that mentions them. If a course changes from 3 units to 4, we update it once, and every program that uses it is instantly correct. If we copied unit values into hundreds of program files, one course change would mean hundreds of edits and inevitable mistakes.

So the requirements file says *which* courses count and *how much* is needed; the course database says what each course actually *is*. The one exception: if an official worksheet explicitly prints a unit value, we record it directly so we're faithful to the source — otherwise the system looks it up.

## Step 6 — Why we track where and when the data came from

Requirements change, and they differ by the year a student entered. So every program file records **when it was scraped, who/what scraped it, the source web pages, and how confident we are.** Each individual requirement also carries a short piece of source evidence — ideally a direct quote and link.

This matters for three reasons: an advisor can trust (and double-check) any result by following it back to the source; we can tell stale data from current data; and when USC updates a requirement, we know exactly what to re-verify. Anything we're unsure about is explicitly flagged for a human to review, rather than guessed at.

## Step 7 — Why Version 1 focuses on the common cases

The vast majority of real requirements are the three types in Step 4: a required course, a choose-from-a-list, or a units-from-a-pool. If Version 1 handles those well — plus catalogue-year tracking, source evidence, the no-double-counting rule, and equivalent-course handling — it already covers most of what advisors check by hand today.

Trying to perfect every rare edge case before launching would delay something genuinely useful by a year. We ship the common cases first, correctly, and deliver value early.

## Step 8 — Why we still include empty sections for the harder rules

Degrees also involve trickier rules: minimum GPAs, grade floors, limits on Pass/No-Pass courses, residency and class-standing requirements, exam (AP/IB) credit, graduate units borrowed into an undergraduate degree, advisor-approved exceptions, and more.

Version 1 doesn't fully automate all of these yet — but the data structure **reserves a labeled spot for every one of them now**, even when that spot is empty. Each spot also notes what the system should eventually do there, and whether a human needs to review it in the meantime.

A useful detail: some of these rules are deliberately *not* simple checklists. For example, a "gateway" course isn't just another required course — if it's missing, the system reports that the student isn't even eligible to declare the program yet, which is different from "you still owe a class." And graduate-unit borrowing is a *ceiling* ("no more than 9 units"), not a goal to reach. The structure keeps these distinct so they're never confused.

## Step 9 — Why this approach avoids expensive rework

Reserving those spots now is the difference between adding a feature and rebuilding the foundation. When we're ready to automate, say, GPA checks, the GPA section already exists in every file and the engine simply starts using it. We don't have to re-scrape hundreds of programs or redesign the data format — which is exactly the year of post-launch rework this design is built to avoid. We agree on the full shape of the data once, up front, and grow into it.

## Step 10 — How the system checks a real student

When a student's transcript comes in, the system works through each requirement the same way an advisor would, but instantly and consistently:

1. **Find the right program file** for that student's catalogue year.
2. **For each requirement, list the courses that could count.**
3. **Match those against what the student actually took.**
4. **Add up the value** — counting courses, or adding up units, as the requirement specifies.
5. **Compare to the threshold:** is it met, partly met, or not met?
6. **Apply the program-wide rules** — for example, make sure one course isn't counted toward two requirements at once.
7. **Report the result** for each requirement: satisfied, in progress, not yet satisfied, or *needs a human to review*.

That last category is essential. When the system isn't sure — an unusual substitution, an exchange program, an advisor exception, a residency question — it does **not** guess. It hands the case to a person. The guiding principle throughout is simple: it is far better to flag something for review than to silently tell a student they've graduated when they haven't.

---

*Questions or corrections about specific USC requirements should go to the relevant school's advising office (Marshall first, since its emphasis values are already partly verified). This document explains the structure; the requirements themselves are still being verified school by school.*
