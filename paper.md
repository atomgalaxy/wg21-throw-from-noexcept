---
title: Throwing from a `noexcept` function should be a contract violation.
document: D3205R0
date: today
audience:
  - SG21, EWG, LEWG
author:
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
  - name: Jeff Snyder
    email: <jeff@caffeinated.me.uk>
  - name: Andrei Zissu
    email: <andrziss@gmail.com>
toc: true
toc-depth: 2
---

<!--
TODO:
- remove the "Lakos Rule" references from the text.
- suggest throwing while an uncaught exception is in flight (such as from a
  destructor) is also a contract violation.
- bring up that we need to be able to enforce that handlers do not throw at
  compile time (Louis) - we have a paper for that, too
- mention that these kinds of postconditions need to be separately configurable
- research original motivation of terminate on noexcept - are we fully covering the original rationale with our new approach?
- do we need a new assertion_kind to disambiguate exceptions thrown from function or from CCA?
- does the new behavior make sense for all the different evaluation semantics? could we be breaking existing code in any of them? or possibly creating new security vulnerabilities (especially in ignore mode)?
- what source location will be provided to the violation handler? careful - binary inflation danger...
- if we opt to coalesce conflicting exceptions, do we do that ad infinitum or enforce some limit? we also need to define the coalesce mechanism
- talk to vendors - how will they handle linking TUs with different contract semantics?
- how will we handle stack unwind for noexcept functions which are now no longer guaranteed not to throw? can we do that without an impact on current optimizations (at least by default)?
- can we provide different semantics for exception-hit-noexcept vs other CCAs? possible mapping: ignore - terminate/Louis; observe - enforce; enforce - enforce; Louis - Louis
- Ville's objection - TU with ignore semantic calls noexcept function in another TU with enforce semantic, resulting in exception flying by the barrier unhindered. proposed resolution - allow the tradeoff of having no stack unwind destructor invocations, as the price for allowing negative testing in the presence of noexcept; anyone using a throwing violation handler in production gets what they bargained for; or as a matter of QoI there could be build flags canceling the noexcept optimizations, thus providing for throwing violation handlers in production for Bjarne's nuclear plant use case
 - in multiple exception situations, what does current_exception() return? can we get a current exception list of all of them? what if there are several exceptions of the same type? and how does catch() behave?
 - note: some of these points originate in this reflector thread: http://lists.isocpp.org/sg21/2024/03/7206.php
- from Bengt: It would be easier to understand the logic if you showed a noexcept function with a pre-condition and the sequence of events when a test case calls the function out of contract from this call until the test framework detects that the condition violation triggered.
- cite D3138R0 as an alternative to negative testing in most cases
- rephrase the paper in terms of unifying erroneous behaviour handling and response - and if the application wants to continue after erroneous behaviour, that should be a choice available to the programmer
- talk about having different tool switches for erroneout behaviour than for normal postconditions
- postcondition -> epilogue, and introduce that nomenclature
-->

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
of a _postcondition assertion_, instead of unconditionally calling
`std::terminate()`.

This postcondition would be the "last" (outermost) postcondition to evaluate,
so any exceptions that a violation handler throws due to other postconditions
still hit the `noexcept` barrier.

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
happens to get called because of a bug - but it seems doubtful that someone
would rely on an exception calling `std::terminate()` instead of calling
`std::terminate()` explicitly.

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
    }
    // 2 - if the exception-in-flight is the one we just threw in (1)
    try {
        throw;
    } catch (MyTestException const&) { // it's a test exception
        throw; // rethrow it
    } catch (...) {
        // for other exceptions, noexcept is still noexcept
        invoke_default_violation_handler(violation);
    }
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

# Viability of negative testing

Negative testing has to be done very carefully - after all, the test program
deliberately calls the function-under-test out-of-contract.

As an example, code that is exception-unsafe cannot be negative-tested using exceptions.

```cpp
void wrapper1(std::function<void>()noexcept f)
{
    std::lock_guard g(some_lock);
    ...
    std::unlock_guard(g);
     wrapper2(f);
}

void wrapper2(std::function<void>()noexcept f)
{
    some_lock.lock();
    f();
    some_lock.unlock();
}

wrapper1([]pre(false){}); // deadlocks
```

```cpp
std::mutex r;

template <typename F>
void with_something(F f) noexcept requires(noexcept(f()))
{
    r.lock();
    f();
    r.unlock()
}
```

Negative-testing `f()` _through_ `with_something` will deadlock the next test.
Note, however, that `f()` is deliberately invoked out-of-contract, and
therefore already requires extreme care. Having some tests is better than
having none, so this proposal still leaves the engineer in a more capable
position.

# ABI considerations

## Is this an ABI break?

Not... necessarily. Yes. But not a bad one.

Of course, we are changing semantics. However, `std::terminate` is a valid
implementation of an implementation-defined handler for a specific violation
semantic, so technically today's code is conforming.

The change comes if you set your own violation handler and set the semantic (in
an implementation-defined way) to not be the kind that calls
`std::terminate()`, but at that point you're already recompiling your code.

## Can I link with past code?

Su

# Prior art

- [@N3248] discusses the reasons we need the Lakos rule, which are obviated by the proposed change
- [@P1656R2] discusses the actual desires of annotating functions that are prevented by the Lakos rule
- [@P2837R0] discusses why we need the Lakos rule
- [@P2900R5] is the current contracts proposal
- [@P2946R1] says that `[[throws_nothing]]` could imply a contract violation on throwing
- [@P3155R0] proposes the application of the Lakos rule in the standard library
- [@P3085R0] has a similar conception of what `noexcept` means.

# FAQ

- If an assertion fails due to the predicate exiting via exception, the
  violation handler throws, does that exception still hit the function's
  `noexcept` barrier and `std::terminate()`?

Yes.


# Acknowledgements

- Jonas Persson contributed comments with regards to noexcept destructors,
  double throws, and unwinding code generation overhead.
