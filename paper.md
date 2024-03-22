---
title: Throwing from a `noexcept` function should be a contract violation.
document: D3205R0
date: today
audience:
  - SG21, EWG, LEWG
author:
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
toc: true
toc-depth: 2
---

# Introduction

Throwing an exception from a noexcept function currently has the effect of
calling `std::terminate()`. This has given rise to the Lakos rule, over which
many, many papers have been written.

This paper proposes to resolve the debate.

We contend that the call to `std::terminate()` was merely the best we could do
at the time to enforce the meaning of `noexcept`, and we can do better now that
we have contract violation handlers.

We propose to change the effect of throwing from a `noexcept` function be
a function postcondition violation, have configurable semantics as per the
Contracts MVP ([@P2900R5]), and therefore, in `observe` and `enforce` modes,
call the violation handler.

# Proposal

We propose that throwing from a `noexcept` function be treated as a violation
of a _postcondition assertion_, and not unconditionally call
`std::terminate()`.

# Rationale

The Lakos rule is there because testing defensive precondition checks in
function implementations by throwing exceptions is useful to a part of the C++
developer community. `noexcept` functions effectively make this technique
impossible.

`noexcept` is reflectable because it matters for exception safety. It means
that the function will not allow an exception to escape when called
in-contract, and thus there is no need to choose a more expensive algorithm to
achieve exception guarantees in the presence of possible exception throws.

This property is important in 

- move, construct, and destroy operations.
- async callbacks, where stack unwinding out of the callback would proceed to
  the runloop instead of being propagated to the continuation.
- correctness of exception-unsafe code that wants to ensure some
  dependency-injected component won't jeoperdise its correctness.

The `std::terminate()` semantic is unlikely to be relied upon as a matter of
deliberate control flow. It is quite clearly a stand-in for a postcondition
violation; people do rely on exit handlers for recovery `std::terminate`
happens to get called because of a bug - but the author finds it highly
doubtful that someone would rely on an exception calling `std::terminate()`
instead of calling `std::terminate()` explicitly.

If we instead redefine throwing from a `noexcept` function as a contract
violation, a violation handler could instead just let the exception propagate
and unwind, achieving the goal of negative testing, while still allowing the
required reflectable properties for code not under test.

## Example: negative testing through `noexcept` boundaries

In a unit test, one might install the following handler:

```cpp
void handle_contract_violation(contract_violation const& violation) {
    if (violation.detection_mode() != evaluation_exception) {
        // 1 - the precondition to be checked will emit this
        throw MyTestException(violation);
    } else {
        // we have an exception, see what it is
        try { // any noexcept barriers we hit should propagate test exceptions
            rethrow_exception(p);
        } catch (MyTestException const&) { // it's a test exception
            throw; // rethrow it
        }
    }
    invoke_default_violation_handler(); // different problem, don't propagate
}
```

The Lakos rule then has no further reason to exist, and we can use `noexcept`
to mean a reflectable postcondition of "function does not throw when called
in-contract" freely.

# `noexcept` and `[[throws_nothing]]`

`noexcept` is a reflectable property that _also_ places a postcondition of not
throwing on the function.

`[[throws_nothing]]` is the [@P2946R1]-proposed syntax of also placing such a
post-condition on a function, but with a default of the `ignore` semantic.

We should really unify this universe of exceptionless postconditions.

# Prior art

- [@N3248] discusses the reasons we need the Lakos rule, which are obviated by the proposed change
- [@P1656R2] discusses the actual desires of annotating functions that are prevented by the Lakos rule
- [@P2837R0] discusses why we need the Lakos rule
- [@P2900R5] is the current contracts proposal
- [@P2946R1] says that `[[throws_nothing]]` could imply a contract violation on throwing
- [@P3155R0] proposes the application of the Lakos rule in the standard library
- [@P3085R0] has a similar conception of what `noexcept` means.