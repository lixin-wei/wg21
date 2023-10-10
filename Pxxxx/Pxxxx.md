---
title: "Sender Algorithm Customization"
subtitle: "Draft Proposal"
document: DxxxxR0
date: today
audience:
  - "LEWG Library Evolution"
author:
  - name: Eric Niebler
    email: <eric.niebler@gmail.com>
toc: true
---

Introduction
============

This paper proposes some design changes to P2300 to address some shortcomings in
how algorithm customizations are found.

The Issues
==========

Completion schedulers are unreliable
------------------------------------

In [@P2300R7], the sender algorithms (`then`, `let_value`, etc) are
customization point objects that internally dispatch via `tag_invoke` to the
correct algorithm implementation. Each algorithm has a default implementation
that is used if no custom implementation is found.

Custom implementations of sender algorithms are found by asking the input sender
for its completion scheduler and using the scheduler as a tag for the purpose of
tag dispatching. A _completion scheduler_ is a scheduler that refers to the
execution context on which that sender will complete.

A typical sender algorithm like `then` might be implemented as follows:

```cpp
/// @brief A helper concept for testing whether an algorithm customization
///   exists
template <class AlgoTag, class SetTag, class Sender, class... Args>
concept @_has-customization_@ =
  requires (Sender sndr, Args... args) {
    tag_invoke(AlgoTag(),
               get_completion_scheduler<SetTag>(get_env(sndr)),
               std::forward<Sender>(sndr),
               std::forward<Args>(args)...);
  };

/// @brief The tag type and the customization point object type for the
///   `then` sender algorithm
struct then_t {
  template <sender Sender, class Fun>
    requires /* requirements here */
  auto operator()(Sender&& sndr, Fun fun) const
  {
    // If the input sender has a completion scheduler, and if we can use
    // the completion scheduler to find a custom implementation for the
    // `then` algorithm, dispatch to that. Otherwise, dispatch to the
    // default `then` implementation.
    if constexpr (@_has-customization_@<then_t, set_value_t, Sender, Fun>)
    {
      auto&& env = get_env(sndr);
      return tag_invoke(*this,
                        get_completion_scheduler<set_value_t>(env),
                        std::forward<Sender>(sndr),
                        std::move(fun));
    }
    else
    {
      return @_default-then-implementation_@(std::forward<Sender>(sndr), std::move(fun));
    }
  }
};
```

This scheme has a number of shortcomings:

1. A simple sender like `just(42)` does not know its completion scheduler. It
   completes on the execution context on which it is started. That is not known
   at the time the sender is constructed, which is when we are looking for
   customizations.

2. For a sender like `on( sched, then(just(), fun) )`, the `then` sender is
   constructed independent of the scheduler passed to `on`. But the `then`
   sender will be executing on that scheduler's (`sched`'s) execution context,
   so it should dispatch to `sched`'s customization of `then`. But how?

3. A composite sender like `when_all(sndr1, sndr2)` cannot know its completion
   scheduler in the general case. Even if `sndr1` and `sndr2` both know their
   completion schedulers -- say, `sched1` and `sched2` respectively -- the
   `when_all` sender can complete on _either_ `sched1` _or_ `sched2` depending
   on which of `sndr1` and `sndr2` completes last. That is a dynamic property of
   the program's execution, not suitable for finding an algorithm customization.

In cases (1) and (2), the issue is that the information necessary to find the
correct algorithm implementation is not available at the time we look for
customizations. In case (3), the issue is that the algorithm semantics make it
impossible to know statically on what execution context the sender will complete.


> *Note:* On the need for async algorithms customization
>
> It's worth asking why async algorithms need the ability to be
> customized at all. After all, the classic STL algorithms need no
> customization; they dispatch using a fixed concept hierarchy to a closed set
> of possible implementations.
>
> The reason is because of the open and continually evolving nature of execution
> contexts. There is little hope of capturing every salient attribute of every
> possible execution model -- CPUs, GPUs, FPGAs, ... -- in a fixed ontology
> around which we can build named concepts and immutable basis operations.
> Instead we do the best we can and then hedge against the future by making the
> algorithms customizable.


This is a reference to [@stdexecgithub].

This is a reference to [@P2300R7].




---
references:
  - id: stdexecgithub
    citation-label: stdexecgithub
    title: "stdexec"
    url: https://github.com/NVIDIA/stdexec
  - id: P2300R7
    citation-label: P2300R7
    type: paper
    title: "`std::execution`"
    author:
      - given: Michał 
        family: Dominiak
        email:  griwes@griwes.info
      - given: Georgy 
        family: Evtushenko
        email:  evtushenko.georgy@gmail.com
      - given: Lewis 
        family: Baker
        email:  lewissbaker@gmail.com
      - given: Lucian Radu
        family: Teodorescu
        email: lucteo@lucteo.ro
      - given: Lee 
        family: Howes
        email:  xrikcus@gmail.com
      - given: Kirk 
        family: Shoop
        email:  kirk.shoop@gmail.com
      - given: Michael 
        family: Garland
        email:  mgarland@nvidia.com
      - given: Eric 
        family: Niebler
        email:  eric.niebler@gmail.com
      - given: Bryce Adelstein
        family: Lelbach
        email: brycelelbach@gmail.com
    url: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html
---
