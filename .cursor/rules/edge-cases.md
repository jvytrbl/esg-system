# Identifying Edge Cases

Use this alongside `tdd.md` whenever deciding what the NEXT test case should be, for any function, API route, or component in this project. Where `tdd.md` covers the Red → Green → Refactor process, this file covers *what to actually test* — how to systematically think through likely edge cases instead of guessing at random.

---

## The core principle: categories, not creativity

Don't try to "brainstorm" edge cases from nothing. Instead, walk through the fixed categories below, one at a time, and ask **"does this category apply to what I'm building right now, and if so, what should happen?"**

This is a checklist to run through, not a creativity exercise.

---

## The 8 categories

### 1. Boundary values
If a rule involves a number, length, or limit, test right at the edge, one below, and one above.
*Example: a field with a 2000-character limit — test at exactly 2000 (should pass), 2001 (should fail).*

### 2. Empty / null / missing
What happens when something that's supposed to have a value doesn't?
*Example: empty text input, `null` email, missing required field.*

### 3. Wrong type / malformed data
What if the data isn't even the shape expected?
*Example: an ID arrives as text instead of a number; a field expected to be a string arrives as an array or object.*

### 4. Duplicate / conflicting state
What if something that should be unique isn't, or two pieces of data contradict each other?
*Example: reassigning a ticket to the engineer already assigned to it (a no-op that should be explicitly rejected, not silently allowed).*

### 5. Permission / authorization edge cases
What if the right action is attempted by the wrong person or role?
*Example: a non-staff user attempting an action reserved for engineers.*

### 6. Extreme scale
What if there's far more (or less) data than typically expected?
*Example: a dropdown with thousands of entries; a history list with hundreds of entries.*

### 7. Concurrent / timing issues
What if two things happen at nearly the same moment?
*Example: two engineers reassigning the same ticket simultaneously — a real, previously-flagged risk in this project's IT Hub module.*

### 8. Security-hostile input
What if the input is deliberately malicious, not just accidentally wrong?
*Example: script tags or HTML in free-text fields, SQL-injection-shaped strings, input designed to overflow a field.*

---

## The judgment call — do NOT apply all 8 blindly

Not every category is relevant to every function. Testing all 8 categories for every single piece of logic, regardless of actual risk or likelihood, wastes effort and violates the "pre-agreed seams" principle in `tdd.md` — testing effort should land on the seams and categories that actually matter, not everywhere uniformly.

**Before writing a test for a given category, ask:**
1. **Can this situation actually happen**, given how this function is really called and what upstream validation already exists? If a value is already guaranteed non-null by the time it reaches this function, don't test this function for `null` — that seam belongs elsewhere (e.g. the API route, or wherever the guarantee is enforced).
2. **Does it matter if this goes wrong?** Low-stakes, internal-only, low-traffic features may reasonably skip categories like "extreme scale" or "concurrent timing" if the realistic risk is negligible.
3. **Is this genuinely a new behavior, or does it duplicate an existing test?** If two categories would produce the same underlying test (e.g. two different "invalid input" tests that both get rejected by the same validation line), consolidate rather than pad the test count.

**When a category's relevance is unclear, ask the user rather than assuming.** State which category you're considering and why, and let the user confirm it's worth a test before writing one. A skipped category is a legitimate, deliberate decision — not an oversight — as long as it was actually considered and consciously set aside, not simply never thought of.

---

## Applying this in a TDD cycle

When starting a new domain function or expanding an existing one:

1. State the seam being tested (per `tdd.md`).
2. Walk the 8 categories mentally against that seam. For each one that plausibly applies, note it.
3. Present the candidate list to the user before writing tests — e.g. "Categories 2 (empty), 4 (duplicate), and 5 (permission) seem relevant here; boundary values and concurrency don't apply to this function. Want to proceed with those three?"
4. Once agreed, take them one at a time as separate Red → Green cycles — never write tests for multiple categories at once (this would violate the vertical-slicing rule in `tdd.md`).