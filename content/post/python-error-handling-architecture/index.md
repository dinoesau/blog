---
title: "Stop Validating Everywhere: An Architectural Guide to Error Handling in Python"
description: "Learn how to build resilient Python applications by distinguishing between validation and assertions, implementing the Result pattern, and using NewTypes to make invalid states unrepresentable."
date: 2026-04-13
categories:
    - Software Architecture
    - Python
    - Development Patterns
tags:
    - Error Handling
    - Type Safety
    - Functional Programming
    - Pydantic
    - NewType
    - Clean Code
---


> *A junior developer writes code that assumes everything will go right (no checks).*  
> *A mid-level developer writes code that assumes everything will go wrong (defensive checks and assertions everywhere).*  
> *A senior developer uses the type system to make going wrong impossible, deleting 80% of those checks entirely.*<br>
> — <cite>Software Engineering Proverb</cite>

<!--more-->

## TL;DR

* **Validate at the edges (Defensive):** Expect bad data from the outside world. Handle it gracefully using the `Ok / Err` pattern (the `Result` pattern) instead of throwing unpredictable `try/except` errors.
* **Assert in the core (Offensive):** Expect perfect data internally. If your internal state is wrong, it's a system bug. Use a custom `assert_ok` to crash immediately (fail-fast).
* **Parse, Don't Validate:** Don't just check data; transform it at the boundary into typed domain primitives using `NewType` (e.g., `ValidatedUserId`, `PositiveAmount`).
* **The Result:** Your core business logic functions *only* accept domain-branded types. This makes invalid states mathematically impossible to represent, allowing you to delete hundreds of lines of defensive `if not data` checks.

---

If you've spent enough time writing Python web services, you've likely written a function that looks exactly like this:

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()


@app.post("/refund")
async def process_refund(req: Request):
    # ❌ The Mid-Level "Defensive" Controller
    try:
        body = await req.json()  # ⚠️ Raises if malformed JSON!

        # Defensive checks everywhere...
        if not body or not isinstance(body, dict):
            raise ValueError("Invalid payload")
        if "userId" not in body or not isinstance(body["userId"], str):
            raise ValueError("Missing UserId")
        if "amount" not in body or not isinstance(body["amount"], (int, float)) or body["amount"] <= 0:
            raise ValueError("Invalid amount")

        user = await get_user(body["userId"])
        if not user:
            raise ValueError("User not found")  # Expected error? Or DB bug?

        # Finally... the actual business logic
        await gateway.refund(user.stripe_id, body["amount"])

        return JSONResponse({"message": "Success"}, status_code=200)
    except Exception as e:
        # ⚠️ Is this a 400 bad request? Or a 500 DB failure? We don't know anymore.
        return JSONResponse({"error": str(e)}, status_code=400)
```

This code is exhausting to read. The core business logic is buried under a mountain of defensive `if/raise` statements. Furthermore, the `except` block has no idea if the error was a user making a typo (400 Bad Request) or the database catching fire (500 Internal Error).

> **A note on FastAPI:** Yes, FastAPI can handle Pydantic validation automatically in the route signature (e.g., `async def process_refund(payload: RefundInput):`). We are doing it manually here to clearly demonstrate the architectural boundary between the Web framework, the Parser, and the Core Domain. In a real app you'd likely use FastAPI's integration — but the architectural pattern remains identical.

Error handling isn't just about catching mistakes; **it's about system architecture**.

To fix this, we need to understand the fundamental difference between **Validation** and **Assertion**, adopt the **Result pattern**, and use **Branded Types** (`NewType`) to push errors to the absolute edges of our system.

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

When data arrives from the outside world (API input, DB reads), it is untrusted. Because we *expect* bad data, raising exceptions is an anti-pattern. `raise` is essentially a hidden `GOTO` statement that destroys Python's type safety (the exception in an `except` block has no static type information about where it came from).

Instead, we use the **`Ok / Err` pattern** (the `Result` pattern), heavily inspired by Go and Rust. We treat errors as standard return values.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Ok[T]:
    """A successful result wrapping a value of type T."""
    value: T


@dataclass(frozen=True)
class Err[E]:
    """A failed result wrapping an error of type E."""
    error: E


# Python 3.12+ type alias with generic parameters
type Result[T, E] = Ok[T] | Err[E]


# ✅ Wrap the chaotic edge in a predictable Result
async def parse_json(req: Request) -> Result[dict, str]:
    try:
        return Ok(await req.json())
    except Exception:
        return Err("Malformed JSON")
```

By returning a `Result`, the type checker *forces* the caller to handle the failure before they are allowed to access `value`. We've eliminated the silent boundary leak.

Usage with structural pattern matching (Python 3.10+):

```python
match await parse_json(req):
    case Ok(body):
        # body is fully typed as `dict` here
        ...
    case Err(error):
        return JSONResponse({"error": error}, status_code=400)
```

## Rule 3: Parse, Don't Validate

Now that we have safely parsed the JSON, we need to ensure it has the correct shape. But traditional validation has a massive flaw: **it doesn't leave a receipt.**

If you write a function `is_valid_email(input: str) -> bool`, and it returns `True`, Python still just sees a `str`. If you pass that string down through five other files, none of those files *know* it was validated. So, mid-level developers defensively re-validate the string everywhere.

To fix this, we use the **"Parse, Don't Validate"** paradigm [^1].
We use schema parsers (like Pydantic [^2]) combined with **Branded Types** (`NewType`) to make invalid states *impossible to represent*.

```python
import uuid
from dataclasses import dataclass
from typing import NewType

from pydantic import BaseModel, field_validator, ValidationError

# 1. The Branded Types (The VIP Wristbands)
#    NewType is zero-cost at runtime — it only exists for the type checker.
ValidatedUserId = NewType("ValidatedUserId", str)
PositiveAmount = NewType("PositiveAmount", float)


# 2. The Pydantic schema (validates raw input)
class _RefundInput(BaseModel):
    user_id: str
    amount: float

    @field_validator("user_id")
    @classmethod
    def _must_be_uuid(cls, v: str) -> str:
        uuid.UUID(v)  # raises ValueError if invalid
        return v

    @field_validator("amount")
    @classmethod
    def _must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("Amount must be positive")
        return v


# 3. The Trusted Domain Model (only constructible from validated data)
@dataclass(frozen=True)
class TrustedRefund:
    user_id: ValidatedUserId
    amount: PositiveAmount


# 4. The Smart Constructor (The Bouncer)
#    This function does not just check data; it transforms and Brands it.
def parse_refund_request(data: dict) -> Result[TrustedRefund, list[dict]]:
    try:
        raw = _RefundInput.model_validate(data)  # Parse & validate
        return Ok(TrustedRefund(
            user_id=ValidatedUserId(raw.user_id),   # Brand it
            amount=PositiveAmount(raw.amount),       # Brand it
        ))
    except ValidationError as e:
        return Err(e.errors(include_input=False))
```

**Why this works:** After data passes through `parse_refund_request`, the returned `TrustedRefund.user_id` is typed as `ValidatedUserId`, not `str`. Any function in your core domain that accepts `ValidatedUserId` is mathematically guaranteed to only ever receive a validated UUID. You cannot accidentally pass a raw, unvalidated string — the type checker will reject it.

## Rule 4: Inside the Core, Assert and Crash

Once data has passed the Smart Constructor, it is inside our trusted domain. We shouldn't be handling expected validation errors anymore. We are dealing with internal system invariants.

If an assumption is wrong here, we *want* to crash. Python's `TypeGuard` makes this powerful by permanently narrowing types for the remainder of a scope.

```python
from typing import TypeGuard, TypeVar

T = TypeVar("T")
E = TypeVar("E")


def is_ok(result: Result[T, E]) -> TypeGuard[Ok[T]]:
    """Type-narrowing predicate: after `if is_ok(x)`, x is `Ok[T]`."""
    return isinstance(result, Ok)


class InternalError(Exception):
    """Raised when a system invariant is violated. Always a 500."""


def assert_ok(result: Result[T, E]) -> Ok[T]:
    """Fail-fast assertion for trusted zones.

    Unlike Python's builtin `assert`, this is NOT disabled by -O.
    If this fires, it means your validation boundary has a hole.
    """
    if not isinstance(result, Ok):
        raise InternalError(str(result.error))
    return result
```

**A note on `assert`:** Python's built-in `assert` statement is stripped when the interpreter runs with `-O` (optimizations). Never use it for system invariants. Always use a custom assertion function like `assert_ok` above that unconditionally raises.

## The Grand Architecture: Layers of Trust

Let's look at the "Before" code from the beginning of this article, refactored into the **Layers of Trust** architecture.

```python
import logging
from typing import TypeVar, TypeGuard

logger = logging.getLogger(__name__)


# --- Shared infrastructure ---

T = TypeVar("T")
E = TypeVar("E")


def is_ok(result: Result[T, E]) -> TypeGuard[Ok[T]]:
    return isinstance(result, Ok)


def assert_ok(result: Result[T, E]) -> Ok[T]:
    if not isinstance(result, Ok):
        raise InternalError(str(result.error))
    return result


def log_critical_bug(error: Exception) -> None:
    logger.critical("System invariant violated", exc_info=error)


# ----------------------------------------------------------------
# --- 1. THE CORE DOMAIN (Zero validation checks!)             ---
#     Because we demand Branded Types, it is mathematically      ---
#     impossible to call this function with dirty data.          ---
# ----------------------------------------------------------------

async def execute_refund(user_id: ValidatedUserId, amount: PositiveAmount) -> None:
    # get_user has also been refactored to return Result[User, DbError]
    # — DB reads are at the "edge" too, so they get the same treatment.
    db_result = await get_user(user_id)

    # ASSERTION: We assume the DB works and the user exists.
    # If not, our system is broken. Fail-fast!
    ok_user = assert_ok(db_result)

    # Type checker guarantees ok_user.value is our User object.
    await gateway.refund(ok_user.value.stripe_id, amount)


# ----------------------------------------------------------------
# --- 2. THE EDGE CONTROLLER                                   ---
# ----------------------------------------------------------------

@app.post("/refund")
async def process_refund_route(req: Request):
    # Layer 1: Parse the chaotic outside world safely
    json_result = await parse_json(req)
    if not is_ok(json_result):
        return JSONResponse({"error": json_result.error}, status_code=400)

    # Layer 2: Smart Constructor (Validate and Brand)
    payload_result = parse_refund_request(json_result.value)
    if not is_ok(payload_result):
        return JSONResponse({"error": payload_result.error}, status_code=400)

    # Layer 3: Execute Core Logic
    try:
        # Pydantic guarantees payload_result.value contains our Branded Types!
        refund = payload_result.value
        await execute_refund(refund.user_id, refund.amount)
        return JSONResponse({"message": "Success"}, status_code=200)
    except InternalError as error:
        # If we end up here, an internal ASSERTION tripped.
        # This is a real bug. Page the developer!
        log_critical_bug(error)
        return JSONResponse({"error": "Internal Server Error"}, status_code=500)
```

## The Takeaway

Look at the `execute_refund` function above.
It is completely pure.
There are no `if` statements checking for empty strings.
There are no `isinstance` checks.
**Your cognitive load drops to zero.**

To stop fighting Python and start leveraging it as an architectural tool, memorize this paradigm:

1. **Validation is Defensive.** You expect the outside world to be messy. Use the `Ok / Err` pattern to parse data, gracefully recover, and return user-friendly 400-level errors.
2. **Assertion is Offensive.** You expect your internal logic to be flawless. Use `assert_ok` to fail-fast, crash, and return 500-level errors when system invariants are broken.
3. **Encode Trust in Types.** Push validation to the edges, use Smart Constructors to create Branded Types (`NewType`), make invalid states unrepresentable, and delete the rest of your runtime checks.

Correctness is so important that any violation of internal logic is a bug.
<mark>Build strict boundaries, trust your types, and watch your codebase become infinitely more resilient.</mark>

[^1]: This concept was popularized by Alexis King in her seminal essay [Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).
[^2]: [Pydantic](https://docs.pydantic.dev/) is the de-facto Python library for data validation using type annotations. It is the Python ecosystem's equivalent of Zod.
