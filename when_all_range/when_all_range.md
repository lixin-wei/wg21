---
title: "`when_all_range` Sender Adaptors"
document: PXXXXR0
date: today
audience:
  - "Library Working Group"
author:
  - name: Lixin Wei
    email: <lixin.wei@gmail.com>
toc: false
---

# Abstract

This paper proposes to add a new sender adaptor algorithm `when_all_range` to the
`std::execution` namespace, targeting C++29. It accepts a `std::ranges::range` of senders,
allowing user to wait for multiple senders which are only known at runtime.

# Description

## The Problem Now

[@P2300R10] provides `std::execution::when_all`, which allows user to wait multiple
senders which are known at compile time. But there still lacks a way
to wait for multiple senders which are **only known at runtime**.

For example:

```c++
int main(int argc, char** argv) {
  int runtime_known_number = std::atoi(argv[1]);
  std::vector<MySender> senders;
  for (int i = 0; i < runtime_known_number; ++i) {
    senders.push_back(CreateAsyncWork());
  }

  ex::when_all(senders); // this line will fail to compile
}
```

In the above example, `when_all` doesn't work, because its signature is `when_all(ex::sender auto... senders)`
which requires the number of senders to be known at compile time. We can't pass a dynamic container to it.

## Proposed Solution

To resolve this problem, this paper proposes to add a new sender adaptor algorithm
called `when_all_range` to the `std::execution` namespace, which accepts a `std::ranges::range` of senders.

```c++
int main(int argc, char** argv) {
  int runtime_known_number = std::atoi(argv[1]);
  std::vector<MySender> senders;
  for (int i = 0; i < runtime_known_number; ++i) {
    senders.push_back(CreateAsyncWork());
  }
  
  ex::when_all_range(senders); // proposed new adaptor
}
```

Like `when_all`, `when_all_range` adapts all senders in the range into a sender that
completes when all input senders have completed. `when_all_range` only accepts senders
with a single value completion signature and on success concatenates all the input
senders' value result datums into a new `std::ranges::range` and pass it to its own value completion operation.

Let's call the range passed to the value completion operation of the returned sender as `output range`.

The output range should conform `std::ranges::random_access_range`.
The i-th element in the output range should be the value result datum of the i-th sender(iteration order) in the input range.

# Proposed Wording

[Change [execution.syn] as follows:]{.ednote}
```default
  struct when_all_t { unspecified };
  struct when_all_with_variant_t { unspecified };
  struct into_variant_t { unspecified };
  @@[`struct when_all_range_t { unspecified };`]{.add}@@
```

```default
  inline constexpr when_all_t when_all{};
  inline constexpr when_all_with_variant_t when_all_with_variant{};
  inline constexpr into_variant_t into_variant{};
  @@[`inline constexpr when_all_range_t when_all_range{};`]{.add}@@
```
