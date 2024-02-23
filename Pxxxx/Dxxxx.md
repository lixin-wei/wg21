Improving diagnostics for sender expressions
============================================

Synopsis
--------

This paper aims to improve the user experience of the sender framework by moving the diagnosis of invalid sender expression earlier, when the expression is constructed, rather than later when it is connected to a receiver. A trivial change to the sender adaptor algorithms makes it possible for the majority of sender expressions to be type-checked early.


Executive Summary
-----------------

Below are the specific changes this paper proposes in order to make early type-checking of sender expressions possible:

1. Define a "non-dependent sender" to be one whose completions are knowable without an environment.
2. Add support for calling `get_completion_signatures` without an environment argument.
3. Tweak the `get_completion_signatures` customization point to check, in order:
	* ... whether the sender has a nested `::completion_signatures` type alias,
	* ... whether the sender defines a 1-argument (w/o env) overload of `get_completion_signatures`,
	* ... whether the sender defines a 2-argument (w/ env) overload of `get_completion_signatures`.
4. Change the definition of the `completion_signatures_of_t` alias template to default the `Environment` template parameter to a hidden *`none-such`* type, and interpret it as a request to compute the non-dependent signatures, if such exist.
5. Require the sender adaptor algorithms to preserve the "non-dependent sender" property  wherever possible.
6. Add "Mandates:" paragraphs to the sender adaptor algorithms to require them to hard-error when passed non-dependent senders that fail type-checking.


Problem Description
-------------------

Type-checking a sender expression involves computing its completion signatures. In the general case, a sender's completion signatures may depend on the receiver's execution environment. For example, the sender:

```c+
read(get_stop_token)
```

when connected to a receiver `rcvr` and started, will fetch the stop token from the receiver's environment and then pass it back to the receiver, as follows:

```c++
auto st = get_stop_token(get_env(rcvr));
set_value(move(rcvr), move(st));
```

Without an execution environment, the sender `read(get_stop_token)` doesn't know how it will complete.

The type of the environment is known rather late, when the sender is connected to a receiver. This is often far from where the sender expression was constructed. If there are type errors in a sender expression, those errors will be diagnosed far from where the error was made, which makes it harder to know the source of the problem.

It would be far preferable to issue diagnostics while *constructing* the sender rather than waiting until it is connected to a receiver.

### Non-dependent senders

The majority of senders have completions that don't depend on the receiver's environment. Consider `just(42)` -- it will complete with the integer `42` no matter what receiver it is connected to. If a so-called "non-dependent" sender advertised itself as such, then sender algorithms could eagerly type-check the non-dependent senders they are passed, giving immediate feedback to the developer.

For example, this expression should be immediately rejected:

```c++
just(42) | then([](int* p) { return *p; })
```


The `then` algorithm can reject `just(42)` and the above lambda because the arguments don't match: an integer cannot be passed to a function expecting an `int*`. The `then` algorithm can do that type-checking only when it knows the input sender is non-dependent. It couldn't, for example, do any type-checking if the input sender were `read(get_stop_token)` instead of `just(42)`.

And in fact, some senders *do* advertise themselves as non-dependent, although P2300 does not currently do anything with that extra information. A sender can declare its completions signatures with a nested type alias, as follows:

```c++
template <class T>
struct just_sender {
  T value;

  using completion_signatures =
    std::execution::completion_signatures<
      std::execution::set_value_t(T)
    >;

  // ...
};
```


Senders whose completions depend on the execution environment cannot declare their completion signatures this way. Instead, they must define a `get_completion_signatures` customization that takes the environment as an argument.

We can use this extra bit of information to define a `non_dependent_sender` concept as follows:

```c++
template <class Sndr>
concept non_dependent_sender =
  sender<Sndr> &&
  requires {
    typename remove_cvref_t<Sndr>::completion_signatures;
  };
```

A sender algorithm can use this concept to conditionally dispatch to code that does eager type-checking.


Suggested Solution
------------------

The authors suggests that this notion of non-dependent senders be given fuller treatment in P2300. Conditionally defining the nested typedef in generic sender adaptors -- which may adapt either dependent or non-dependent senders -- is awkward and verbose. We suggest instead to support calling `get_completion_signatures` either with _or without_ an execution environment. This makes it easier for authors of sender adaptors to preserve the "non-dependent" property of the senders it wraps.

We suggest that a similar change be made to the `completion_signatures_of_t` alias template. When instantiated with only a sender type, it should compute the non-dependent completion signatures, or be ill-formed.

Proposed Wording
----------------

Change [exec.syn] as follows:

<blockquote>
<pre>
...

template&lt;class Sndr, class<ins>\...</ins> Env <del>= empty_env</del>>
  concept sender_in = <em>see below</em>;

\...

template&lt;class Sndr, class<ins>\...</ins> Env <del>= empty_env</del>>
  requires sender_in&lt;Sndr, Env<ins>\...</ins>>
using completion_signatures_of_t = call-result-t&lt;get_completion_signatures_t, Sndr, Env<ins>\...</ins>>;

\...
</pre>
</blockquote>


Change [exec.snd.concepts] as follows:

<blockquote>
<pre>
template&lt;class Sndr, class<ins>...</ins> Env <del>= empty_env</del>>
  concept sender_in =
    sender&lt;Sndr> &&
    <ins>(</ins>queryable&lt;Env><ins> &&...)</ins> &&
    requires (Sndr&& sndr, Env&& env) {
      { get_completion_signatures(
           std::forward&lt;Sndr>(sndr), std::forward&lt;Env>(env)<ins>...</ins>) }
        -> <em>valid-completion-signatures</em>;
    };
</pre>
</blockquote>



We would like `get_completion_signatures(s)` to handle the case where `s` is awaitable in *any* coroutine type (modulo those with `await_transform`s that prevent `s` from being awaitable). We need to extend the awaitable traits to handle this case. Change [exec.awaitables] as follows:


<blockquote>

1. The sender concepts recognize awaitables as senders. For this clause ([exec]), an ***awaitable*** is an expression that would be well-formed as the operand of a `co_await` expression within a given context.

2. For a subexpression `c`, let `GET-AWAITER(c, p)` be expression-equivalent to the series of transformations and conversions applied to `c` as the operand of an *await-expression* in a coroutine, resulting in lvalue `e` as described by [expr.await]/3.2-4, where `p` is an lvalue referring to the coroutine’s promise type, `Promise`. This includes the invocation of the promise type’s `await_transform` member if any, the invocation of the `operator co_await` picked by overload resolution if any, and any necessary implicit conversions and materializations. <ins>Let `GET-AWAITER(c)` be expression-equivalent to `GET-AWAITER(c, q)` where `q` is an lvalue of an unspecified empty class type *`none-such`* that lacks an `await_transform` member, and where `coroutine_handle<none-such>` behaves as `coroutine_handle<void>`.</ins>

3. Let *`is-awaitable`* be the following exposition-only concept:

	<pre>
	template&lt;class T>
	concept <em>await-suspend-result</em> = <em>see below</em>;
	
	template&lt;class A, class<ins>...</ins> Promise>
	concept <em>is-awaiter</em> = <em>// exposition only</em>
	  requires (A& a, coroutine_handle&lt;Promise<ins>...</ins>> h) {
	    a.await_ready() ? 1 : 0;
	    { a.await_suspend(h) } -> <em>await-suspend-result</em>;
	    a.await_resume();
	  };
	
	template&lt;class C, class<ins>...</ins> Promise>
	concept <em>is-awaitable</em> =
	  requires (C (*fc)() noexcept, Promise&<ins>...</ins> p) {
	    { GET-AWAITER(fc(), p<ins>...</ins>) } -> is-awaiter&lt;Promise<ins>...</ins>>;
	  };
	</pre>

	`await-suspend-result<T>` is `true` if and only if one of the following is `true`:

	* `T` is `void`, or
	* `T` is `bool`, or
	* `T` is a specialization of `coroutine_handle`.
	
4. For a subexpression `c` such that `decltype((c))` is type `C`, and an lvalue `p` of type `Promise`, `await-result-type<C, Promise>` denotes the type `decltype(GET-AWAITER(c, p).await_resume())` <ins>, and `await-result-type<C>` denotes the type `decltype(GET-AWAITER(c).await_resume())`</ins>.
</blockquote>




Change [exec.getcomplsigs] as follows:

<blockquote>

1. `get_completion_signatures` is a customization point object. Let `sndr` be an expression such that `decltype((sndr))` is `Sndr` <del>, and let `env` be an expression such that `decltype((env))` is `Env`</del>. <ins>Then `get_completion_aignatures<Sndr>` is expression-equivalent to:</ins>

	1. <ins>`remove_cvref_t<Sndr>::completion_signatures{}` if that expression is well-formed,</ins>

	2. <ins>Otherwise, `tag_invoke_result_t<get_completion_signatures_t, Sndr>{}` if that expression is well-formed,</ins>

	3. <ins>Otherwise, if `is-awaitable<Sndr>` is `true`, then:</ins>

		<ins>
		
			completion_signatures<
			  SET-VALUE-SIG(await-result-type<Sndr>), // see [exec.snd.concepts]
			  set_error_t(exception_ptr),
			  set_stopped_t()>{}
		</ins>
	4. <ins>Otherwise, `get_completion_signatures(sndr)` is ill-formed.</ins>

2. <ins>Let `env` be an expression such that `decltype((env))` is `Env`.</ins> Then `get_completion_signatures(sndr, env)` is expression-equivalent to:

	1. <ins>`remove_cvref_t<Sndr>::completion_signatures{}` if that expression is well-formed,</ins>

	2. <ins>Otherwise, </ins> `tag_invoke_result_t<get_completion_signatures_t, Sndr, Env>{}` if that expression is well-formed,

    <!-- -->
	2. <del>Otherwise, `remove_cvref_t<Sndr>::completion_signatures{}` if that expression is well-formed,</del>

	3. Otherwise, if `is-awaitable<Sndr, env-promise<Env>>` is `true`, then:

			completion_signatures<
			  SET-VALUE-SIG(await-result-type<Sndr, env-promise<Env>>), // see [exec.snd.concepts]
			  set_error_t(exception_ptr),
			  set_stopped_t()>{}

	4. Otherwise, `get_completion_signatures(sndr, env)` is ill-formed.

2. <ins></ins>

2. Let `rcvr` be an rvalue receiver of type `Rcvr`, and let `Sndr` be the type of a sender such that `sender_in<Sndr, env_of_t<Rcvr>>` is true. Let `Sigs...` be the template arguments of the `completion_signatures` specialization named by `completion_signatures_of_t<Sndr, env_of_t<Rcvr>>`. Let `CSO` be a completion function. If sender `Sndr` or its operation state cause the expression `CSO(rcvr, args...)` to be potentially evaluated ([basic.def.odr]) then there shall be a signature `Sig` in `Sigs...` such that `MATCHING-SIG(tag_t<CSO>(decltype(args)...), Sig)` is `true` ([exec.general]).
</blockquote>


