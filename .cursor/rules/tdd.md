# Test-Driven Development (TDD)

Use this whenever building a new feature or fixing a bug in ENV-Portal test-first — especially for `lib/domain/**` business logic (CBE calculations, IT ticket logic, HR approval logic, etc.).

TDD is the **red → green → refactor** loop. This file is the reference that makes that loop produce tests worth keeping: what a good test is, where tests belong, common mistakes to avoid, and the rules of the loop itself. Consult this **before and during** the loop — not just once at the start and then forgotten.

---

## What a Good Test Actually Is

A good test verifies **behavior through the public interface**, not internal implementation details. The code inside a function can be rewritten entirely — the test shouldn't need to change, as long as the outward behavior stays the same.

A good test reads like a specification. Its name should describe **what capability exists**, not how it's built.

```typescript
// GOOD — describes WHAT, in plain business terms
test("reassigning a ticket updates its PIC to the new engineer", async () => {
  const ticket = createTicket({ itPicEmail: "naim.misrap@envirosgroup.com" });
  const updated = await reassignTicket(ticket.id, "syaza@envirosgroup.com");
  expect(updated.itPicEmail).toBe("syaza@envirosgroup.com");
});
```

```typescript
// BAD — describes HOW, coupled to internal steps
test("reassign calls assignItTicket with correct arguments", async () => {
  const spy = jest.spyOn(repo, "assignItTicket");
  await reassignTicket(ticket.id, "syaza@envirosgroup.com");
  expect(spy).toHaveBeenCalledWith(ticket.id, "syaza@envirosgroup.com");
});
```

The first test survives a refactor (e.g. changing which repository method gets called internally). The second one breaks the moment you refactor internals, even though nothing about actual behavior changed — a false alarm that trains you to stop trusting your own tests.

**Characteristics of a good test:**
- Tests behavior a real user or calling code actually cares about
- Uses only the public API (the exported function, the API route, the component's visible output) — never reaches into private/internal pieces
- Survives internal refactors untouched
- One clear, logical assertion focus per test
- Named like a sentence describing a capability, not a function's internal steps

---

## Seams — Where Tests Belong

A **seam** is a public boundary: the interface where you can observe behavior from the outside, without reaching inside to check internal steps. Tests live at seams — never against internals.

**In ENV-Portal, typical seams look like:**
- A pure function in `lib/domain/**` (e.g. `calculateVariance`, `addRemark`, `reassignTicket`) — the seam is its inputs and return value
- An API route in `app/api/**` — the seam is the request in, the response out (status code + JSON body)
- A React component — the seam is what renders on screen / what a user can click, not internal state variables

**Agree on the seam before writing the test.** Before writing any test, be clear (even just to yourself, or state it in your Cursor prompt) about exactly what boundary you're testing. You can't test everything — deciding the seam up front is what keeps testing effort on the behavior that actually matters (the CBE budget math, the reassignment logic, the approval state machine) instead of scattering effort across every trivial internal detail.

A useful question to ask yourself before every test: *"What's the public interface here, and what seam am I actually testing?"*

---

## Mocking — When to Fake Something

Mock only at **system boundaries** — the edges of your code where it talks to something outside your control:

- External APIs (Microsoft Graph, Power Automate webhooks, Azure Blob Storage)
- The database (`mssql` calls) — prefer a real test database/schema when practical; mock only when that's not feasible
- Time or randomness (e.g. `new Date()` in something like `signatureDates.ts`)
- The file system, if ever touched directly

**Do NOT mock:**
- Your own domain logic functions (`lib/domain/**`)
- Internal collaborators within the same module
- Anything you actually control and can just run for real in a test

**Plain rule of thumb:** if faking it means you'd be testing "did I call the fake correctly" instead of "did the real behavior happen," you're mocking the wrong thing.

### Designing code so it's easy to mock later

**1. Pass dependencies in, don't create them inside the function**

```typescript
// EASY to test — dependency passed in, can swap a fake in tests
function notifyTicketAssigned(ticket, mailer) {
  return mailer.send(ticket.requestorEmail, "Ticket assigned");
}

// HARD to test — creates its own dependency internally, can't substitute it
function notifyTicketAssigned(ticket) {
  const mailer = new GraphMailer(process.env.GRAPH_TOKEN);
  return mailer.send(ticket.requestorEmail, "Ticket assigned");
}
```

**2. Prefer specific, named functions over one generic catch-all**

```typescript
// GOOD — each call is independently mockable, one clear shape per function
const itApi = {
  getTicket: (id) => fetch(`/api/it/tickets/${id}`),
  assignTicket: (id, email) => fetch(`/api/it/tickets/${id}/assign`, { method: "PATCH", body: JSON.stringify({ itPicEmail: email }) }),
};

// BAD — one generic function, mocking requires conditional logic to figure out which "kind" of call it is
const itApi = {
  call: (endpoint, options) => fetch(endpoint, options),
};
```

---

## Anti-Patterns to Avoid

**Implementation-coupled tests** — mocking your own internal collaborators, testing private/unexported functions, or verifying success by checking a side channel (e.g. querying the SQL table directly) instead of going through the real interface.

```typescript
// BAD — bypasses the actual interface to check the database directly
test("reassignTicket saves to database", async () => {
  await reassignTicket(ticketId, newEmail);
  const row = await db.query("SELECT * FROM IT_TICKETS WHERE id = ?", [ticketId]);
  expect(row.it_pic_email).toBe(newEmail);
});

// GOOD — verifies through the same interface the app actually uses
test("reassignTicket makes the new PIC retrievable via getTicket", async () => {
  await reassignTicket(ticketId, newEmail);
  const ticket = await getTicket(ticketId);
  expect(ticket.itPicEmail).toBe(newEmail);
});
```
The tell-tale sign of this anti-pattern: the test breaks when you refactor, even though real behavior hasn't changed.

**Tautological tests** — the expected value is calculated the same way the code calculates it, so the test can never actually catch a wrong calculation. It passes by construction, not because the logic is correct.

```typescript
// BAD — expected value is computed with the same formula as the code under test
test("calculateVariance sums correctly", () => {
  const items = [{ budget: 100, actual: 80 }, { budget: 50, actual: 60 }];
  const expected = items.reduce((sum, i) => sum + (i.budget - i.actual), 0);
  expect(calculateVariance(items)).toBe(expected);
});

// GOOD — expected value is an independent, hand-picked known number
test("calculateVariance sums correctly", () => {
  const items = [{ budget: 100, actual: 80 }, { budget: 50, actual: 60 }];
  expect(calculateVariance(items)).toBe(10); // 20 + (-10) = 10, worked out by hand
});
```

**Horizontal slicing** — writing ALL the tests for a feature first, then writing ALL the implementation afterward. This is the single most common mistake an AI coding assistant makes with TDD if not explicitly told otherwise.

Why it's a trap: tests written this way verify *imagined* behavior — what you guessed the code should do — rather than behavior confirmed step by step as it's actually built. The tests tend to check the general *shape* of things rather than real edge cases, and by the time you write the implementation, you're locked into a test structure decided before you understood the problem.

**The correct approach — vertical slicing:** one seam → one test → the minimal code to pass it → repeat. Each cycle is a small, complete round trip, and each new test is informed by what the previous cycle taught you — not planned out in bulk beforehand.

---

## The Rules of the Loop

1. **Red before green.** Always write the failing test first. Run it. Confirm it actually fails before writing any implementation. Don't write test and implementation in the same breath.
2. **Minimal implementation only.** Write just enough code to make the current failing test pass — nothing more. Don't add functionality "while you're in there" that no test is asking for yet.
3. **One slice at a time.** One seam, one test, one minimal implementation, per cycle. Resist the urge to write several tests upfront (horizontal slicing).
4. **Confirm before moving on.** After Red, confirm the test actually fails. After Green, confirm it actually passes. Never assume — actually run `npx vitest run` at each step.
5. **Refactor is a separate, optional step — after green, not instead of green.** Once a test passes, you may clean up naming/structure without changing behavior, then re-run the test to confirm it's still green. Don't skip straight to "make it elegant" before it even works.

---

## Applying This Day-to-Day in ENV-Portal

When asking Cursor (or any AI assistant) to build something in this project:

- Point it at the specific seam first — e.g. "let's build `reassignTicket` in `lib/domain/it/`, the seam is: given a ticket ID and new PIC email, it returns the updated ticket with the new PIC set."
- Ask for ONE failing test only, and stop there.
- Run it yourself (`npx vitest run`) — don't just trust a claim that it "should fail."
- Once confirmed Red, ask for the minimal implementation — nothing extra.
- Run the test again yourself to confirm Green.
- Only then, optionally, ask for a refactor pass — and re-run the test afterward to confirm it's still passing.
- Repeat for the next behavior (e.g. "what happens if the new PIC email isn't in the engineer list" — that's the next seam/test, not something to add speculatively into the first implementation).