---
title: 'A Concept and Template Meta-programming Approach to Session Types in C++'
date: 2023-05-20
permalink: /posts/2023/05/cpp-session-types
tags:
  - types
  - session types
  - c++
  - template metaprogramming
---

# Introduction
Programs communicate -- whether with other programs or
humans. Software developers write programs with a protocol in
mind. Sometimes there's documentation for the protocol. But there's no
mechanism that keeps implementation and documentation in sync. Bugs
occur when protocols diverge.

Many of us already use type systems. But naive approaches to typing
fall short of guaranteeing that an implementation speaks a
protocol. For example: Suppose two threads `T1` and `T2` communicate
over a channel `chan`. `T1` and `T2` play a guessing game. `T1`
guesses a number (`int`) and `T2` informs `T1` if the guess is right
(`bool`). We might type `chan` as `Chan<std::variant<int,
bool>>`. This isn't helpful, though. If `T1` sends a `bool` the
program should not compile, yet it does.

[Session types](https://en.wikipedia.org/wiki/Session_type) are a tool
that solves this problem. This post discusses an implementation of
session types in C++. You'll learn more about how you can use session
types to specify protocols. You'll also see some features in C++
(concepts and template meta-programming) you might not know how to use
today.

All code is available [on GitHub](https://github.com/obicons/session-types).

## Motivating Example: IO
Instead of two threads playing a guessing game, let's make a game for
humans. First, the computer generates a random number between 1
and 100. Second, the computer prompts the user to guess the
number. Then, the user enters a guess. Next, the computer evaluates
the user's guess. If the guess is correct then the program sends a
congratulatory message and exits. If the guess is wrong then the
program asks the user if they give up. The user keeps guessing the
generated number until they get it right or give up.

This listing shows how we might specify this protocol with session types:
```
using GuessingGameProtocol =
    Rec<Choose<QueryUserProtocol<Choose<KeepPlayingProtocol, Var<Z>>>,
               ExitProtocol>>;

template <HasDual P>
using QueryUserProtocol = Send<std::string, Recv<int, P>>;

using KeepPlayingProtocol = Send<std::string, Recv<std::string, Var<Z>>>;

using ExitProtocol = Choose<ExitUserLost, ExitUserWon>;
using ExitUserLost = Send<std::string, Send<int, Send<std::string, Send<std::ostream&(std::ostream&), Z>>>>;
using ExitUserWon = Send<std::string, Send<std::ostream&(std::ostream &), Z>>;
```

Let's unpack:
- `Rec<P>` introduces a recursive protocol. It allows the protocol to repeat itself using `Var`.
- `Choose<P1, P2>` allows the implementation to make a choice between
  protocols `P1` and `P2`. `Choose<QueryUserProtocol<...>,
  ExitProtocol>` represents a choice between asking the user for
  another guess and terminating.
- `Send<T1, P>` represents that the implementation sends a value of
  type `T1` then executes the protocol `P`. Similarly, `Recv` receives.
- `Var<N>` accepts a natural number -- either `Z` or `Succ<M>` --
  and returns to the recursive environment `N` levels out.

Here's what an implementation of this protocol might look like:

```
int main() {
  std::default_random_engine generator;
  generator.seed(time(nullptr));

  std::uniform_int_distribution distribution(1,10);
  const auto the_number = distribution(generator);

  auto keep_going = true;
  auto guess = 0;

  Chan<GuessingGameProtocol, decltype(&std::cin), decltype(&std::cout)> chan(&std::cin, &std::cout);

  while (keep_going) {
    auto c1 = chan.enter().choose1();
    auto c2 = c1 << "Guess: ";
    auto c3 = c2 >> guess;

    keep_going = guess != the_number;
    if (keep_going) {
      auto c4 = c3.choose1();
      auto c5 = c4 << "Incorrect. Keep playing? (y/n) ";
      std::string response;
      auto c6 = c5 >> response;
      keep_going = response != "n";

      chan = c6.ret();
    } else {
      chan = c3.choose2().ret();
    }
  }

  if (guess != the_number) {
    auto ce = chan.enter().choose2().choose1();
    ce << "You lose. I was thinking of " << the_number << "." << std::endl;
  } else {
    auto ce = chan.enter().choose2().choose2();
    ce << "You win!" << std::endl;
  }
}
```

Some explanations are in order:
- The `Chan` type represents a session typed communication channel. It
  encapsulates some other input and output mechanisms. In this case,
  `cin` and `cout`.
- Programs operate on a `Chan` by calling methods. Following a method
  call, it is illegal to reuse the `Chan` -- doing so triggers a
  run time error. Operations return new channels that speak the proper protocol.
- `chan.enter()` enters a recursive context.
- `Chan<Choose<P1, P2>>::choose1()` returns a channel that speaks
  `P1`. `Chan<Choose<P1, P2>>::choose2()` returns a channel that speaks `P2`.
- `Chan<Recv<T, P>>::operator>>(T &t)` reads a value from the
  channel's input stream into `t`. It returns a channel that speaks
  `P`. `operator<<(const T &t)` behaves similarly.
- `Chan<Var<N>>::ret()` returns a channel that speaks the `N`th recursive protocol defined in the original type.

Combined, this provides a stronger guarantee than what we had before:
**Programs always send the right shaped data for the protocol, or send nothing.**

## Motivating Example: Multithreaded Communication
When two threads communicate over a channel it's important that they
speak the same protocol. Our intuition tells us that every `Send<T, ...>` 
should have a corresponding `Recv<T, ...>`, etc. We call this
*duality*. We desire that our type system only allow two threads to
communicate over the channel if they are each other's duals.

This next listing shows part an implementation of program with two
threads: `T1` and `T2`. `T1` sends a value to `T2`, who responds with
that value doubled.

```
#include <cstdio>
#include <iostream>
#include <memory>
#include "sesstypes.hh"
#include "concurrentmedium.hh"

using Protocol = Rec<Send<int, Recv<int, Var<Z>>>>;

void log(const std::string &tname, const std::string &action, int val) {
    printf("%s %s %d\n", tname.c_str(), action.c_str(), val);
}

void log(const std::string &tname, const std::string &action) {
    printf("%s %s\n", tname.c_str(), action.c_str());
}

struct {
    template <typename CommunicationMedium>
    void operator()(Chan<Protocol, CommunicationMedium, CommunicationMedium> chan) {
        int val;

        auto c = chan.enter();
        for (int i = 0; i < 5; i++) {
            auto c1 = c << i;
            log("T1", "sent", i);

            int val;
            auto c2 = c1 >> val;
            log("T1", "received", val);

            c = c2.ret().enter();
        }
        log("T1", "done", -1);
    }
} t1;


int main() {
    auto chan = std::make_shared<ConcurrentMedium<ProtocolTypes<Protocol>>>();
    auto threads = connect<Protocol>(t1, t2, chan);
    threads.first.join();
    threads.second.join();
}
```

Critically, we are only allowed to call `connect<Protocol>(t1, t2)` if
`t2` is the dual of `t1`. This requirement is enforced at compile time.


# A C++ Implementation
Now that we have a better idea about what session types are, let's see
how they are implemented.

## Session Types

### Duality with C++ Concepts
Duality is critical to our concurrent motivating example. The idea
that a type has a dual can be captured using a
`concept`. [Concepts](https://en.cppreference.com/w/cpp/language/constraints)
are named boolean predicates that restrict template parameters.

Take the definition of the `Recv` type:
```
template <typename T, HasDual P>
struct Recv {
    using dual = Send<T, typename P::dual>;
};
```

`Recv` defines `dual` as its opposite, `Send`. Since `Recv` requires
that the protocol `P` has a dual, we constrain `P` to types where
`HasDual` evaluates to true.

Here's the implementation of `HasDual`:
```
template <typename T>
concept HasDual = requires { typename T::dual; };
```

This introduces another new feature of C++: The `requires`
expression. `requires { typename T::dual; }` evaluates to `true` if
`typename T::dual` compiles. Otherwise, it evaluates to `false`. (By
the way, it's illegal for a requires expression to always fail to
compile.)

Concepts are great because they improve compiler error messages. We've
all seen the error vomit C++ compilers produce when template expansion
fails. Concepts eliminate much of the noise to help us debug.

### Natural Numbers with Template Meta-Programming
Remember that `Var` uses a natural number to decide how many levels of
recursion to return from. Let's see how our natural numbers are implemented.

Here's a naive way to implement natural numbers:
```
struct Z {};

template <typename T>
struct Succ {};
```

This definition allows us to write real natural numbers like
`Succ<Succ<Z>>`. The problem is that it also allows us to write things
that aren't natural numbers, like `Succ<int>`. Given that this post is
about radical type checking, we should not be satisfied with this.

Instead, we use template metaprogramming to enforce that a type is a
natural number. There are two ways for a type to be a natural number:
1. It is `Z`.
2. It is `Succ<M>` and `M` is a natural number.

Here's how we define a concept `IsNat` to check that a type is a natural number:
```
template <typename T>
struct IsNatImpl : std::false_type {};

template <>
struct IsNatImpl<Z> : std::true_type {};

template <typename M>
struct IsNatImpl<Succ<M>>
    : std::conditional_t<
                IsNatImpl<M>::value,
                std::true_type,
                std::false_type
      > {};

template <typename T>
concept IsNat = IsNatImpl<T>::value;
```

The `type_traits` header provides `std::true_type` and
`std::false_type` as canonical representations of `true` and `false`
at the type level. The default implementation of `IsNatImpl` inherits
from `false_type`, so its `value` member is `false`. The `Z`
specialization inherits from `true_type`, so its `value` member is
true.

The last specialization is kind of tricky. `conditional_t<Condition,  A, B>` 
is `A` when `Condition` is `true` and `B` otherwise. So we recursively check that 
`IsNatImpl<M>::value` is `true`. If so, then `Succ<M>` is a natural number,
and so we inherit from `true_type`.

This lets us write a more correct version of natural numbers:
```
template <typename T>
struct Succ;

// Code for IsNat.

template <>
struct Succ<Z> {};

template <IsNat M>
struct Succ<M> {};
```

### The Chan Type
Here we discuss the implementation of the `Chan` type. 
Since recursion is the hardest thing that we have to support
we'll describe it first. It has far-reaching implications.

The idea is to represent a channel as a `Chan<Protocol, E>`.
`Protocol` is the protocol type. For example, `Recv<int, Send<int, Z>>`. 
`E` (for environment) is kind of like a stack. Here's what I mean:

```
template <HasDual P, typename IT, typename OT, typename E>
class Chan<Rec<P>, IT, OT, E> : ChanBase<IT, OT> {
public:
    using ChanBase<IT, OT>::ChanBase;

    Chan<P, IT, OT, std::pair<Rec<P>, E>> enter() {
        // Implementation not shown.
    }
};
```

So, `Chan` is specialized on recursive protocols. It provides only one method, `enter`.
This makes it impossible to try to read from a recursive protocol, for example. 
The `enter` method for a protcol `Rec<P>` pushes `P` onto a stack. Since this all occurs in 
the type system, we represent the stack as a `std::pair`.

This allows us to define `Var<N>`, which pops `N` levels from the environment:
```
template <HasDual P, typename IT, typename OT, typename E>
class Chan<Var<Z>, IT, OT, std::pair<P, E>> : ChanBase<IT, OT> {
public:
    using ChanBase<IT, OT>::ChanBase;

    Chan<P, IT, OT, E> ret() {
        // Implementation not shown.
    }
};

template <typename T, HasDual P, typename IT, typename OT, typename E>
class Chan<Var<Succ<T>>, IT, OT, std::pair<P, E>> : ChanBase<IT, OT> {
public:
    using ChanBase<IT, OT>::ChanBase;

    Chan<Var<T>, IT, OT, E> ret() {
        // Implementation not shown.
    }
};

```
This is sort of recursive. In the base case, `ret` returns a channel whose 
protocol is the top of the environment stack. Otherwise, for `Var<N>`, `ret` returns a channel 
that also speaks `Var`. Only this time, it's `Var<N - 1>`. 

`Chan` is specialized for all of the types with duals. For example, here's `Chan<Recv<...>, ...>`:
```
template <typename T, HasDual P, typename IT, typename OT, typename E>
class Chan<Recv<T, P>, IT, OT, E> : ChanBase<IT, OT> {
public:
    using ChanBase<IT, OT>::ChanBase;

    Chan<P, IT, OT, E> operator>>(T &t) {
        if (ChanBase<IT, OT>::used) {
            throw ChannelReusedError();
        }

        ChanBase<IT, OT>::used = true;
        (*ChanBase<IT, OT>::input) >> t;
        return Chan<P, IT, OT, E>(ChanBase<IT, OT>::input, ChanBase<IT, OT>::output);
    }
};
```

Since it's specialized, the only thing we can do with a `Chan<Recv<...>>` is 
use `operator>>`. This prevents a large number of mistakes -- we can't send
an integer at an unexpected time, for example.


## Concurrent Communication Primitive
The second motivating example uses `ConcurrentMedium` to create a
`Chan`, instead of `cin` and `cout`. This allows two threads to
communicate over a channel. This section describes the design of `ConcurrentMedium`.

### Guarantees
1. Two threads can both read and write data to a `Chan`.
2. Threads do not read their own write. If a thread attempts to read
   its own write, it blocks until another write is available.
3. Threads may only read a write once. If a thread attempts to read a
   write twice, it blocks until a new write is available.
4. Every write is observed by the next read. If a thread attempts to
   write data before the last write is read, it blocks until a read occurs.

### Storage
We store writes in a `std::variant`. This is a type-safe union. So,
the type `ConcurrentMedium<std::variant<int, std::string>>` can
communicate values with types of `int` or `std::string`.

This listing shows this implementation:
```
template <typename... Ts>
class ConcurrentMedium<std::variant<Ts...>> {
public:
    ConcurrentMedium()
        : was_read(true), writers_waiting(0), readers_waiting(0) {}

    template <typename T>
    ConcurrentMedium& operator<<(const T &value) {
        std::unique_lock held_lock(lock);
        while (!was_read) {
            // Needs to be in a while loop to ignore "spurious wakeups".
            // https://en.cppreference.com/w/cpp/thread/condition_variable/wait
            writers_waiting++;
            writer_cv.wait(held_lock);
            writers_waiting--;
        }

        data = value;
        was_read = false;
        write_source = std::this_thread::get_id();

        if (readers_waiting > 0) {
            reader_cv.notify_one();
        }

        return *this;
    }

    template <typename T>
    ConcurrentMedium& operator>>(T &datum) {
        std::unique_lock held_lock(lock);
        while (write_source == std::this_thread::get_id() || was_read) {
            readers_waiting++;
            reader_cv.wait(held_lock);
            readers_waiting--;
        }

        datum = std::get<T>(data);
        was_read = true;

        if (writers_waiting > 0) {
            writer_cv.notify_one();
        }

        return *this;
    }

private:
    std::mutex lock;

    int readers_waiting;
    std::condition_variable reader_cv;

    int writers_waiting;
    std::condition_variable writer_cv;

    std::variant<Ts...> data;
    std::thread::id write_source;
    bool was_read;
};
```

### Problem 1: How to Ensure Reads/Writes are Type Safe?
You may notice a small problem with `operator>>` and `operator<<`:
They accept *any* type `T`, but we are only able to read/write `T` if
it is part of the variant. 

The way we're going to solve this problem is to create a concept
`AssignableToVariant<T, V>` that is true whenever `T` can be written
to the variant `V`. `AssignableToVariant` is written by using a
template meta-program called `OneOf`.  Here are the implementations:

```
template <typename T, typename V>
struct OneOf : public std::false_type {};

template <typename T, typename... Ts>
struct OneOf<T, std::variant<Ts...>> : public std::conditional_t<
        (std::is_same_v<T, Ts> || ...),
        std::true_type,
        std::false_type
    >
{};

template <typename T, typename V>
concept AssignableToVariant = OneOf<T, V>::value;
```

This is similar to `IsNatImpl`. The syntax `(std::is_same_v<T, Ts> || ...)`
is called a [*fold expression*](https://en.cppreference.com/w/cpp/language/fold). 
It essentially rewrites the original expression into
`(std::is_same_v<T, Ts[0]> || ... || std::is_same_v<T, Ts[N]>)`,
although `Ts[0]` is not real syntax.

These are the updated signatures for `operator<<` and `operator>>`:
```
template <AssignableToVariant<std::variant<Ts...>> T>
ConcurrentMedium& operator<<(const T &value);

template <AssignableToVariant<std::variant<Ts...>> T>
ConcurrentMedium& operator>>(T &value);
```

### Problem 2: How to Create a ConcurrentMedium For an Arbitrary Protocol?
`ConcurrentMedium` is hard to use. If we have a protocol `Send<int, Read<std::string, ...>>`, 
it is time-consuming and error-prone to keep writing `ConcurrentMedium<std::variant<int, std::string, ...>>`.
Plus, we have to exert effort to keep the variant and the protocol in sync.
To solve this problem, we'll create another template meta-program called `ProtocolTypes`. 
`ProtocolTypes<Send<int, Read<std::string, ...>>>` automatically creates a `std::variant<int, std::string, ...>`.

Here's the implementation:
```
template <typename Variant, typename T>
struct ProtocolTypesImpl;

template <typename... Ts, typename T, HasDual P>
struct ProtocolTypesImpl<std::variant<Ts...>, Recv<T, P>> {
    using type = typename ProtocolTypesImpl<std::variant<T, Ts...>, P>::type;
};

template <typename... Ts>
struct ProtocolTypesImpl<std::variant<Ts...>, Z> {
  using type = std::variant<Z, Ts...>;
};

template <typename... Ts, typename T, HasDual P>
struct ProtocolTypesImpl<std::variant<Ts...>, Send<T, P>> {
    using type = typename ProtocolTypesImpl<std::variant<T, Ts...>, P>::type;
};

template <typename... Ts, HasDual P>
struct ProtocolTypesImpl<std::variant<Ts...>, Rec<P>> {
    using type = typename ProtocolTypesImpl<std::variant<Ts...>, P>::type;
};

template <typename... Ts, IsNat N>
struct ProtocolTypesImpl<std::variant<Ts...>, Var<N>> {
    using type = std::variant<Ts...>;
};

template <HasDual P>
using ProtocolTypes =  ProtocolTypesImpl<std::variant<>, P>::type;
```

Of course, there's a small problem with this implementation. Namely, 
if we have a protocol `Send<int, Recv<int, Z>>`, we create
a `std::variant<int, int>`. Then, `std::get<int>(data)` is illegal because
the type `int` does not uniquely index the variant. We need _all_ types to be
unique.

Once again, we use a template meta-program to implement this idea:
```
template <typename T, typename... Ts>
struct Unique : std::type_identity<T> {};

template <typename... Ts, typename U, typename... Us>
struct Unique<std::variant<Ts...>, U, Us...>
    : std::conditional_t<(std::is_same_v<U, Ts> || ...),
                         Unique<std::variant<Ts...>, Us...>,
                         Unique<std::variant<Ts..., U>, Us...>> {};

template <typename T>
struct MakeUniqueVariantImpl;

template <typename... Ts>
struct MakeUniqueVariantImpl<std::variant<Ts...>> {
    using type = typename Unique<std::variant<>, Ts...>::type;
};

template <typename T>
using MakeUniqueVariant = typename MakeUniqueVariantImpl<T>::type;
```

And we revise `ProtocolTypes`:
```
using ProtocolTypes = MakeUniqueVariant<ProtocolTypesImpl<std::variant<>, P>::type>;
```

Now we can easily type a `ConcurrentMedium`: `ConcurrentMedium<ProtocolTypes<Protocol>>`.

# Reflections
## Soundness
There are (at least) two important ways this approach is not sound:
- **Users can accidentally reuse channels**. We mitigate this risk by
  raising an exception in this event. An affine type system could help
  solve this problem. Someone [has already shown how to implement this
  in
  C++](https://github.com/IFeelBloated/Type-System-Zoo/blob/master/substructural%20type%20(affine%20type).cxx).
  I left this unimplemented for this post for two reasons. First, the
  approach isn't sound either. It relies on automatic template type
  inference. But if users manually specify template types then the code
  passes all type checking incorrectly. Second, it's complex. It's too distracting 
  for this blog post.
- **Liveliness is not guaranteed**. Programs could simply never send
  (or receive) values over a channel. This can be mitigated by linear
  types, which require that a variable be used exactly once.
## Completeness
This approach does help us describe communications between exactly
two entities. But here are some scenarios that this specific approach doesn't
help:
- **Communications between several threads**.
- **Asynchronous communication**.
