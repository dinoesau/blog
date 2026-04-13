---
title: "Stop Validating Everywhere: An Architectural Guide to Error Handling in TypeScript"
description: "Learn how to build resilient TypeScript applications by distinguishing between validation and assertions, implementing the Result pattern, and using Branded Types to make invalid states unrepresentable."
date: 2026-04-13
categories:
    - Software Architecture
    - TypeScript
    - Development Patterns
tags:
    - Error Handling
    - Type Safety
    - Functional Programming
    - Zod
    - Branded Types
    - Clean Code
---


> *A junior developer writes code that assumes everything will go right (no checks).*  
> *A mid-level developer writes code that assumes everything will go wrong (defensive checks and assertions everywhere).*  
> *A senior developer uses the type system to make going wrong impossible, deleting 80% of those checks entirely.*<br>
> — <cite>Software Engineering Proverb</cite>

<!--more-->

## 📌 TL;DR

* **Validate at the edges (Defensive):** Expect bad data from the outside world. Handle it gracefully using the `val / ok` pattern (the `Result` pattern) instead of throwing unpredictable `try/catch` errors.
* **Assert in the core (Offensive):** Expect perfect data internally. If your internal state is wrong, it’s a system bug. Use TypeScript's `asserts` keyword to crash immediately (fail-fast).
* **Parse, Don't Validate:** Don't just check data; transform it at the boundary into **Branded Types** (e.g., `EmailAddress`, `ValidatedUserId`).
* **The Result:** Your core business logic functions *only* accept Branded Types. This makes invalid states mathematically impossible to represent, allowing you to delete hundreds of lines of defensive `if (!data)` checks.

---

If you’ve spent enough time writing Node.js or TypeScript, you’ve likely written a function that looks exactly like this:

```typescript
// ❌ The Mid-Level "Defensive" Controller
async function processRefund(req: Request, res: Response) {
  try {
    const body = await req.json(); // ⚠️ Throws if malformed JSON!
    
    // Defensive checks everywhere...
    if (!body || typeof body !== 'object') {
      throw new Error("Invalid payload");
    }
    if (!body.userId || typeof body.userId !== 'string') {
      throw new Error("Missing UserId");
    }
    if (typeof body.amount !== 'number' || body.amount <= 0) {
      throw new Error("Invalid amount");
    }

    const user = await db.getUser(body.userId);
    if (!user) {
      throw new Error("User not found"); // Expected error? Or DB bug?
    }

    // Finally... the actual business logic
    await gateway.refund(user.stripeId, body.amount);
    
    return res.status(200).send("Success");
  } catch (error) {
    // ⚠️ Is this a 400 bad request? Or a 500 DB failure? We don't know anymore.
    return res.status(400).send({ error: error.message }); 
  }
}
```

This code is exhausting to read. The core business logic is buried under a mountain of defensive `if/throw` statements. Furthermore, the `catch` block has no idea if the error was a user making a typo (400 Bad Request) or the database catching fire (500 Internal Error).

Error handling isn’t just about catching mistakes; **it’s about system architecture**.

To fix this, we need to understand the fundamental difference between **Validation** and **Assertion**, adopt the **Result pattern**, and use **Branded Types** to push errors to the absolute edges of our system.

## Rule 1: Learn the Difference Between Validation and Assertion

The most important architectural distinction you can make is understanding the difference between validating data and asserting state.

Think of your application like an exclusive Nightclub.

| Feature | Validation | Assertion |
| :--- | :--- | :--- |
| **Location** | The Front Door (API/IO Boundary) | The VIP Room (Core Logic) |
| **Expectation** | Bad data is *expected* | Data is *trusted* |
| **Philosophy** | Defensive (Bouncer) | Offensive (Security Guard) |
| **Outcome** | Graceful Recovery (400 Bad Request) | Immediate Crash (500 System Error) |

* **Validation is the bouncer at the front door.** 
The bouncer *expects* people to hand him fake IDs or be underage. 
He calmly turns them away. 
**Validation inspects untrusted external input and recovers gracefully.**

* **Assertion is the security guard deep inside the VIP room.** 
The guard *expects* everyone in the room to have a VIP wristband. 
If someone is in the room without one, the bouncer system has fundamentally failed. 
The guard stops the music and shuts down the party. 
**Assertions inspect internal logic and fail fast.**

When you mix these up, systems become fragile. If you crash the app when a user types a bad email, you have a terrible UX. If you try to "gracefully recover" when a database query returns an impossible state—like a negative account balance—you silently corrupt your data.

## Rule 2: At the Edge, Treat Errors as Values

When data arrives from the outside world (API input, DB reads), it is untrusted. Because we *expect* bad data, throwing exceptions is an anti-pattern. `throw` is essentially a hidden `GOTO` statement that destroys TypeScript's type safety (the error in a catch block is always typed as `unknown`).

Instead, we use the **`val / ok` pattern** (the `Result` pattern), heavily inspired by Go and Rust. We treat errors as standard return values.

```typescript
// The Result Type Blueprint
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

// ✅ Wrap the chaotic edge in a predictable Result
async function parseJson(req: Request): Promise<Result<unknown, string>> {
  try {
    return { ok: true, value: await req.json() };
  } catch (err) {
    return { ok: false, error: "Malformed JSON" };
  }
}
```

By returning a `Result`, the TypeScript compiler *forces* the caller to handle the failure before they are allowed to access `value`. We've eliminated the silent boundary leak.

## Rule 3: Parse, Don't Validate

Now that we have safely parsed the JSON, we need to ensure it has the correct shape. But traditional validation has a massive flaw: **it doesn't leave a receipt.**

If you write a function `isValidEmail(input): boolean`, and it returns `true`, TypeScript still just sees a `string`. If you pass that string down through five other files, none of those files *know* it was validated. So, mid-level developers defensively re-validate the string everywhere.

To fix this, we use the **"Parse, Don't Validate"** paradigm [^1]. 
We use schema parsers (like Zod [^2]) combined with **Branded Types** to make invalid states *impossible to represent*.

```typescript
import { z } from "zod";

// 1. The Branded Types (The VIP Wristbands)
type ValidatedUserId = string & { readonly __brand: unique symbol };
type PositiveAmount = number & { readonly __brand: unique symbol };

// 2. The Smart Constructor (The Bouncer)
// This schema does not just check data; it transforms and Brands it.
const RefundSchema = z.object({
  userId: z.string().uuid().transform(val => val as ValidatedUserId),
  amount: z.number().positive().transform(val => val as PositiveAmount)
});

// A helper to turn Zod into our Result pattern
function parseRefundRequest(data: unknown): Result<z.infer<typeof RefundSchema>, string> {
  const parsed = RefundSchema.safeParse(data);
  if (!parsed.success) return { ok: false, error: parsed.error.message };
  
  return { ok: true, value: parsed.data };
}
```

## Rule 4: Inside the Core, Assert and Crash

Once data has passed the Smart Constructor, it is inside our trusted domain. We shouldn't be handling expected validation errors anymore. We are dealing with internal system invariants.

If an assumption is wrong here, we *want* to crash. TypeScript makes this powerful with the `asserts` keyword, which permanently narrows types for the remainder of a scope.

```typescript
// A utility to bridge Result types back into assertions for trusted zones
function assertOk<T>(result: Result<T, Error>): asserts result is { ok: true; value: T } {
  if (!result.ok) {
    // CRASH! The VIP guard shuts down the party.
    throw result.error; 
  }
}
```

## The Grand Architecture: Layers of Trust

Let's look at the "Before" code from the beginning of this article, refactored into the **Layers of Trust** architecture.

```typescript
// --- 1. THE CORE DOMAIN (Zero validation checks!) ---
// Because we demand Branded Types, it is mathematically impossible 
// to call this function with dirty data.
async function executeRefund(userId: ValidatedUserId, amount: PositiveAmount) {
  const dbResult = await db.getUser(userId);
  
  // ASSERTION: We assume the DB works and the user exists. 
  // If not, our system is broken. Fail-fast!
  assertOk(dbResult); 
  
  // TypeScript guarantees dbResult.value is our User object.
  await gateway.refund(dbResult.value.stripeId, amount);
}

// --- 2. THE EDGE CONTROLLER ---
async function processRefundRoute(req: Request, res: Response) {
  // Layer 1: Parse the chaotic outside world safely
  const jsonResult = await parseJson(req);
  if (!jsonResult.ok) {
    return res.status(400).send({ error: jsonResult.error });
  }

  // Layer 2: Smart Constructor (Validate and Brand)
  const payloadResult = parseRefundRequest(jsonResult.value);
  if (!payloadResult.ok) {
    return res.status(400).send({ error: payloadResult.error });
  }

  // Layer 3: Execute Core Logic
  try {
    // TS knows payloadResult.value contains our Branded Types!
    await executeRefund(payloadResult.value.userId, payloadResult.value.amount);
    return res.status(200).send("Success");
  } catch (error) {
    // If we end up here, an internal ASSERTION tripped. 
    // This is a real bug. Page the developer!
    logCriticalBug(error);
    return res.status(500).send("Internal Server Error");
  }
}
```

## The Takeaway

Look at the `executeRefund` function above. 
It is completely pure. 
There are no `if` statements checking for empty strings. 
There are no `typeof` checks. 
**Your cognitive load drops to zero.**

To stop fighting TypeScript and start leveraging it as an architectural tool, memorize this paradigm:

1. **Validation is Defensive.** You expect the outside world to be messy. Use the `val / ok` pattern to parse data, gracefully recover, and return user-friendly 400-level errors.
2. **Assertion is Offensive.** You expect your internal logic to be flawless. Use `asserts` to fail-fast, crash, and return 500-level errors when system invariants are broken.
3. **Encode Trust in Types.** Push validation to the edges, use Smart Constructors to create Branded Types, make invalid states unrepresentable, and delete the rest of your runtime checks.

Correctness is so important that any violation of internal logic is a bug. 
<mark>Build strict boundaries, trust your types, and watch your codebase become infinitely more resilient.</mark>

[^1]: This concept was popularized by Alexis King in her seminal essay [Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).
[^2]: [Zod](https://zod.dev/) is a TypeScript-first schema declaration and validation library.
