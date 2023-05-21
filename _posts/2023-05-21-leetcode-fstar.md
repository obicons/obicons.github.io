---
title: 'Using F* to Formally Verify a LeetCode Problem'
date: 2023-05-20
permalink: /posts/2023/05/fstar-leetcode
tags:
  - programming puzzles
  - fstar
---

Formal methods are currently not widely embraced due to their
perceived difficulty. However, the landscape is changing with the
emergence of new technologies that make formal methods more accessible
than ever before. [F\*](https://www.fstar-lang.org/), developed by Microsoft Research, is a
groundbreaking functional language that combines dependent types and
proof-oriented features. By bridging the gap between programming and
proving, F\* facilitates a gradual adoption of formal methods by
developers. In this post, I will provide an introduction to the
fundamentals of F* and demonstrate how we can leverage its
capabilities to verify a solution to a LeetCode problem. I can't cover
all the background material needed to understand F* in this post. I
assume that you have some experience in a language like Haskell or
OCaml.

This writing has three goals. First, I want to showcase how
far formal methods have come. Second, there is not a lot of material
discussing how to use formal methods. I hope others are able to learn
from my mistakes, and newcomers can pick up some proof-engineering
strategies. Finally, I want to draw attention to some current
pain-points for the sake of improving current formal methods
research.

Contents:
1. [Basics of F*](#fstar-basics)
  * [Functions](#fstar-basics-functions)
  * [Dependent Types](#dependent-types)
  * [Assertions and Tactics](#assertions-and-tactics)
2. [LeetCode Problem](#leetcode-problem)
  * [Solution Design](#solution-idea)
  * [Modeling the Problem in F\*](#fstar-modeling)
  * [Proof of Correctness](#fstar-proof)
3. [Takeaways](#takeaways)

## Basics of F* <a name="fstar-basics"></a>

F\* is a complex language, and I am but a journeyman. The purpose of
this section is only to familiarize you, gentle reader, with enough
F\* to broadly understand this post's verification efforts. If you are
interested in learning more, check out the [F\* tutorial](http://www.fstar-lang.org/tutorial/).

### Functions <a name="fstar-basics-functions"></a>

F\* is inspired by ML languages. You can define simple functions like
this:
```
let double (x: int) : int
    = x + x
```
This just defines a function called `double` that accepts an `int` as
a parameter, and returns an `int`. Note that in F\* `int` refers to a
*mathematical integer*, not a fixed-size integer as in C. This means
that the value of `x` can be arbitrarily large (small).

Note that we may want to define double like:
```
let double (x: int) : int
    = x * 2
```
But this simple definition won't work because `*` is reserved by F\*
for constructing tuples. F\* tells us this fact with an informative
error message:
```
(Error 189) Expected expression of type "Type"; got expression "x" of
type "Prims.int"
```

Instead, we have to import a definition that
redfines `*` to refer to multiplication. We do this by *opening* a
module. This definition works:
```
open FStar.Mul

let double (x: int) : int
    = x * 2
```

### Dependent Types <a name="dependent-types"></a>

In a dependently typed programming language, types are permitted to
depend on values. Let's consider the `double` example:
```
let double (x: int) : (result: int{result = x + x})
    = x * 2
```
We changed the return type of `double` to `(result: int{result = x +
x})`. This is called a *refinement type*. This is a dependent type
because the type *depends* on the value of `x` (as well as the return
value of `double`). Note that there is nothing special about the name
`result` -- we just needed a name to refer to the return value of
`double` in the refinement type. Any name would work. 

Interestingly, notice that `x + x` is not syntactically the same as
`x * 2`. F\* is aware of the semantics of the `*` operator and the `+`
operator, and automatically proved that `x * 2 = x + x`. This
highlights the power of F\*: Many facts can be proven with little
effort.


### Assertions and Tactics <a name="assertions-and-tactics" />
In F\*, `assert` statements check that a condition is true at
*proof-time* (i.e., before the code runs). This is done by proving the
condition asserted. Here is a simple example:
```
let _ = assert (true)
```
Of course, the proposition `true` is always provable (`true` is
`true`).

Here's an example of a proposition that cannot be proved:
```
let _ = assert (false)
```
This produces this error message:
```
(Error 19) assertion failed; The SMT solver could not prove the query. Use --query_stats for more details.
```
Of course `false` cannot be proved (`false` is never `true`).

These examples are rather boring. Let's consider an example that uses
more interesting pieces of logic:

```
let _ = assert (forall (x: nat) (y: nat) .
                y >= x ==> 
                    (exists (z: nat) .
                        y = x + z))
```
In more familiar logic, we'd write this as $\forall x, y . y >= x
\implies \exists z . y = x + z$.

But if we try to verify our assertion with F\*, it fails:
```
(Error 19) assertion failed; The SMT solver could not prove the query. Use --query_stats for more details.
```

Under the hood, F\* uses the [Z3 SMT
solver](https://github.com/Z3Prover/z3) to perform proofs. While Z3 is
powerful, no theorem prover can automatically prove all theorems. Z3
appears stuck here. Let's try adding hints to help Z3 get unstuck:

```
open FStar.Mul
open FStar.Tactics

let _ = assert (forall (x: nat) (y: nat) .
                y >= x ==> 
                    (exists (z: nat) .
                        y = x + z))
        by (
            let x = forall_intro () in
            let y = forall_intro () in
            let imp = implies_intro () in
                witness (`(`#y - `#x));
                dump "after witness"
        )
```
We provide hints by using *tactics*, which are programs that
manipulate proofs. Every proof has 1 or more *goals*, or statements
that we need to show are true. Tactics use known facts to simplify
goals. This example shows a few tactics: 
- `forall_intro` introduces the first variable quantified by forall to
  the set of known facts (i.e., the variable exists and has the
  specified type). As a really simple example, `forall_intro`
  transforms a goal like $\vdash \forall (x: \mathbb{N}) . x = x$ into $(x : \mathbb{N})
  \vdash x = x$.
- `implies_intro` adds the antecedent of an implication to the set of facts known to
    the theorem prover. To prove $\Gamma \vdash a \implies b$, it is
    sufficient to show $\Gamma, a \vdash b$.
- `witness` helps us manipulate existence quantifiers. `witness` adds
  a term that shows an object with a given property exists. Here, our
  witness to the existential quantifier is `y - x`. 
- `dump` is an extremely useful tactic. It shows the current goals
  that need to be proved.

Dump shows us this message:
```
Goal 1/2:
(x: Prims.nat), (x'0: Prims.nat), (_: x'0 >= x) |- _ : Prims.squash (x'0 - x >= 0 == true)

Goal 2/2:
(x: Prims.nat), (x'0: Prims.nat), (_: x'0 >= x) |- _ : Prims.squash
(x'0 = x + (x'0 - x))
```

If you read these goals for a second, they should seem obviously
true. F\* is quite easy to use: If you think something is obvious,
just stop talking and see if F\* completes the proof:

```
open FStar.Mul
open FStar.Tactics

let _ = assert (forall (x: nat) (y: nat) .
                y >= x ==> 
                    (exists (z: nat) .
                        y = x + z))
        by (
            let x = forall_intro () in
            let y = forall_intro () in
            let imp = implies_intro () in
                witness (`(`#y - `#x))
        )
```

In this case, it does.


## [LeetCode Problem: Capacity to Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) <a name="leetcode-problem"/>
The problem we're going to solve and verify is the [Capacity to
Ship Packages within D
Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/). You're
given `weights` (an array of positive numbers representing the weights of
items), and `days` (the maximum number of days you have to ship all
the items). These items must be loaded onto a ship with a capacity of
`capacity`. The challenge is to find the smallest value of `capacity`
so that the number of days required to ship the items is less than or
equal to `days`. Check out the LeetCode description for more details.

### Solution Design <a name="solution-idea"></a>
Clearly, the minimum capacity that *might* work is the maximum element
of `weights`. For, if the capacity were any smaller, it would be
impossible to ship the largest item. The largest capacity we should
consider is the sum of the item weights. Any larger capacity is
superfluous, since a ship with this capacity can already ship all the
items in 1 day. The correct capacity is therefore somewhere in the
range $[maximum\\_element~ weights,~ sum~ weights]$.

The naive approach is to simply check every weight in this range. But
this number could be quite large -- for instance, when the number of
items is large but the maximum weight is small. A smarter approach is
to use binary search to find the correct capacity.

To be frank, I find that getting the bounds of binary search right to
be a little tricky. For tricky loop bounds, I craft [loop
invariants](https://en.wikipedia.org/wiki/Loop_invariant) to help me
write the code. Let $min\\_elt$ denote the smallest capacity that
maybe could ship the items, and $max\\_elt$ denote the largest
capacity we should consider. We will maintain two key invariants:
1. $\forall x . x < min\\_elt \implies time\\_to\\_ship~ weights~ x >
   days$.
   
2. $\forall x . x >= max\\_elt \implies time\\_to\\_ship~ weights~ x <=
   days$.

Under these invariants, when $min\\_elt = max\\_elt$, the correct
capacity to return is $min\\_elt$ (or, equivalently, $max\\_elt$).

### Modeling the Problem in F* <a name="fstar-modeling" />

#### Days to Ship Items Given a Capacity
Let's start by computing the number of days it takes to ship items
with `weights` weights given a capacity `capacity`. We'll represent
`weights` as a non-empty list of natural numbers. F*
already provides a theory of lists, so we'll use that.

```
module Capacity

open FStar.List
open FStar.List.Tot
open FStar.Tactics

let weight_list = (l:list nat{not (isEmpty l)})
```

The syntax `list nat` describes a list of natural numbers. We use a
refinement type to specify that the list is non-empty.

Here's a function definition that returns the number of days it takes
to ship some items:
```
let rec max_elt (l: weight_list) : nat =
  match l with
    | [x] -> x
    | (x::xs) -> 
      let max' = max_elt xs in
        if x >= max' then x
        else max'

let rec days_to_ship' (weights: weight_list)
                      (capacity: nat{capacity >= (max_elt weights)}) 
                      (current_cap: nat) 
                      : (x: nat{x >= 1})
  = 
  match weights with
    | [x] -> 
      if x <= current_cap then 1
      else 2
    | (x::xs) ->
      if x <= current_cap then
        days_to_ship' xs capacity (current_cap - x)
      else
        1 + (days_to_ship' xs capacity capacity)

let days_to_ship (weights: weight_list) 
                 (capacity: nat{capacity >= (max_elt weights)}) 
                 : (x: nat{x >= 1})
  = days_to_ship' weights capacity capacity
```

A few notes about these functions:
* In F\*, we must explicitly denote when functions are recursive by
  using the `let rec` syntax.
* The `match` syntax performs pattern-matching. Inside of `max_elt`,
  `[x]` matches with a list containing exactly 1 item. The second
  match case `(x::xs)` matches with an item consed into any list. Note
  that these patterns are exhaustive since a weight list is
  non-empty. Also note that F\* verifies this exhaustivity for us, automatically.
* Notice the use of the refinement type on the `capacity`
  parameter. This is applying our earlier argument: The minimum
  capacity we can use to ship the items is the maxmium weight of the
  items.

#### Defining the Solution Function
Here's the implementation of our solution function in F\*:
```
let nat_sum (a: nat) (b: nat) : nat = a + b

let sum_of_weights (weights: weight_list) : nat = 
  List.Tot.fold_left nat_sum (hd weights) (tl weights)

let rec lemma_sum_of_weights_is_gte_max (weights: weight_list) :
  Lemma (ensures (sum_of_weights weights) >= max_elt weights)
  =
  match weights with
    | [w] -> ()
    | (x::xs) ->
        FStar.List.Tot.Properties.fold_left_monoid nat_sum 0 xs;
        lemma_sum_of_weights_is_gte_max xs

let min_bound (weights: weight_list) : nat = max_elt weights

let max_bound (weights: weight_list) : nat = sum_of_weights weights

// Returns the minimum capacity necessary to ship all the items in `days` days.
// Note that we have to specify that we decrease the difference between max_cap and min_cap.
let rec ship_within_days' (weights: weight_list) 
                          (days: nat{days > 0})
                          (min_cap: nat{min_cap >= min_bound weights})
                          (max_cap: nat{max_cap >= min_cap})
                          : Tot (n:nat{n >= min_cap /\ n <= max_cap}) (decreases max_cap - min_cap)
    =
    if min_cap = max_cap then
      min_cap
    else
      let middle_cap = (min_cap + max_cap) / 2 in
      let total_days = days_to_ship weights middle_cap in
      if total_days > days then
        ship_within_days' weights days (middle_cap + 1) max_cap
      else
        ship_within_days' weights days min_cap middle_cap

let ship_within_days (weights: weight_list) (days: nat{days > 0}) 
  : (n:nat{n >= min_bound weights /\ n <= max_bound weights})
  = lemma_sum_of_weights_is_gte_max weights;
    ship_within_days' weights 
                      days
                      (max_elt weights)
                      (sum_of_weights weights)
```

The heart of the implementation is `ship_within_days'`, so we'll start
there. This is a fairly simple binary search implementation. Again,
we're just maintaining the 2 invariants discussed in the [Solution
Design](#solution-idea) subsection. Try to go through the logic and see
why those invariants are maintained.

The first bit of new syntax we'll discuss is the return type of
`ship_within_days'`. It returns `Tot (n:nat{n >= min_cap /\ n <=
max_cap}) (decreases max_cap - min_cap)`. In F\*, all functions must
be total -- meaning they must terminate. So, really, the type of `double` [from
earlier](fstar-basics-functions) is 
```
val double (x: int) : Tot int
```
But F\* nicely writes `Tot` for us. Unfortunately, F\* doesn't know
*why* the function `ship_within_days'` terminates. We explain it:
Because `max_cap - min_cap` always decreases. F\* *can* see that this
statement is true, and then accepts our function as terminating. If we
delete `(decreases max_cap - min_cap)` from our code, F\* produces
this error:
```
Could not prove termination of this recursive call; The SMT solver could not prove the query. Use --query_stats for more details.
```
This is our cue to add the `decreases` expression.

Our primary solution function is `ship_within_days`. There's one bit
of magic in it: The application of the lemma
`lemma_sum_of_weights_is_gte_max`. This is required because we used a
refinement type for `max_cap` that requires `max_cap >=
min_cap`. Unfortunately, F\* cannot automatically prove that
`(sum_of_weights weights) >= (max_elt weights)`, so type checking
fails if we delete the application of the lemma:
```
Subtyping check failed; expected type max_cap: Prims.nat{max_cap >= max_elt weights}; got type Prims.nat; The SMT solver could not prove the query.
```
In general, F\* cannot automatically prove propositions that require
induction. But once we apply the lemma, F\* can easily verify that the
types are correct.

Now, let's discuss our `max_bound` implementation for a moment. As we
mentioned in [the Solution Design](#solution-idea), the maximum bound on the
weights is just the sum of all weights. To sum the weights, we use the standard
`fold_left` function that should be familiar to functional
programmers. Note that we **cannot** write `sum_of_weights` like this:
```
// Error
let sum_of_weights (weights: weight_list) : nat = 
  List.Tot.fold_left (+) (hd weights) (tl weights)
```
This is because the type of `+` is `int -> int -> int`. While `nat` is
a subtype of `int`, F*'s type checking algorithm does not induce `int
-> int -> int` will produce a `nat`. To solve this problem, we
explicitly define `nat_sum`.

Finally, `lemma_sum_of_weights_is_gte_max` procedes by induction. We
use the `Lemma (...)` type because the function is a proof. In the
case where this is exactly 1 item in the list, we produce the value
`()`. This term has a type of `unit`. In F\*, the type `Lemma (ensures
(sum_of_weights weights) >= max_elt weights)` is really just a synonym
for the type `u:unit{(sum_of_weights weights) >= max_elt
weights}`. So, F\* will automatically try (and succeed!) to show our
lemma is true.

In the case when there is more than 1 item in the list, we first apply
[`FStar.List.Tot.Properties.fold_left_monoid`](https://github.com/FStarLang/FStar/blob/f781540098be4090ecca612fcb9cc1879aec6b62/ulib/FStar.List.Tot.Properties.fst#L840). This
establishes the fact that `nat_sum (x::xs) = x + nat_sum xs`. The
following line (`lemma_sum_of_weights_is_gte_max xs`) convinces F\*
that the lemma holds by induction. As an exercise: Look at lemma
`fold_left_monoid` provides and consider why we didn't use this
definition:
```
// Error
let sum_of_weights (weights: weight_list) : nat = 
  List.Tot.fold_left nat_sum 0 weights
```

### Proof of Correctness <a name="fstar-proof" />
There are two facts we want to prove:
1. Our solution ships all the items within `days` days.
2. Any capacity smaller than the one returned by our solution does not
   ship items within `days` days. I.e., our solution is minimal.

In fact, these statements are direct consequences of the 2 invariants
we constructed in [our design subsection](#solution-idea). So, let's
start by writing these invariants in F\*:
```
let min_bound_invariant (weights: weight_list) 
                        (cap: nat{cap >= min_bound weights})
                        (days: nat{days > 0})
  = forall (x : nat) . x >= min_bound weights /\ x < cap ==> days_to_ship weights x > days

let max_bound_invariant (weights: weight_list)
                        (cap: nat{cap >= min_bound weights}) 
                        (days: nat{days > 0})
  = forall (x : nat) . x >= cap ==> days_to_ship weights x <= days
```

Let's also define the concept of minimality:
```
let is_minimal (w: weight_list) (c: nat{c >= min_bound w}) (days: nat{days > 0}) = 
  c = min_bound w \/ (c > min_bound w /\ days_to_ship w (c - 1) > days)
```

The proof follows from induction. We'll start by drawing the outline
of the proof, then fill in details until it is complete. To start the
proof:
```

let rec lemma_ship_within_days'_ships_within_days (weights: weight_list) 
                                                  (days: nat{days > 0})
                                                  (min_cap: nat{min_cap >= min_bound weights})
                                                  (max_cap: nat{max_cap >= min_cap})
  : Lemma 
    (requires min_bound_invariant weights min_cap days /\ 
              max_bound_invariant weights max_cap days)
    (ensures (days_to_ship weights (ship_within_days' weights days min_cap max_cap)) <= days /\
             is_minimal weights (ship_within_days' weights days min_cap max_cap) days)
    (decreases max_cap - min_cap)
  = 
  if min_cap = max_cap then
     ()
  else 
     admit ()
```

Notice the new `requires` component of the `Lemma` type. The requires
and ensures clauses of Lemma are
[preconditions](https://en.wikipedia.org/wiki/Precondition) and
[postconditions](https://en.wikipedia.org/wiki/Postcondition)
respectively. Our strategy is to require that our 2 invariants hold at
each call to `lemma_ship_within_days'_ships_within_days`. Then, it
is obvious that the postconditions hold. Indeed: Notice that F\*
automatically finds a proof when `min_cap = max_cap`. On the other
hand, we use `admit ()` in the `else` branch. F\* programs that
contain `admit ()` aren't proofs at all - `admit ()` forces F\* to
accept the current goals as true (even if it they are false). However, it's
invaluable when building proofs.

Let's zoom in further by applying the definition of `ship_within_days`
in the else branch:
```
// Error
let rec lemma_ship_within_days'_ships_within_days (weights: weight_list) 
                                                  (days: nat{days > 0})
                                                  (min_cap: nat{min_cap >= min_bound weights})
                                                  (max_cap: nat{max_cap >= min_cap})
  : Lemma 
    (requires min_bound_invariant weights min_cap days /\ 
              max_bound_invariant weights max_cap days)
    (ensures (days_to_ship weights (ship_within_days' weights days min_cap max_cap)) <= days /\
             is_minimal weights (ship_within_days' weights days min_cap max_cap) days)
    (decreases max_cap - min_cap)
  = 
  if min_cap = max_cap then
     ()
  else 
    let middle_cap = (min_cap + max_cap) / 2 in
    let total_days = days_to_ship weights middle_cap in
    if total_days > days then (
       lemma_ship_within_days'_ships_within_days weights days (middle_cap + 1) max_cap
    ) else (
        admit ()
    )
```

Unfortunately, verification fails at this point:
```
(Error 19) assertion failed; The SMT solver could not prove the query. Use --query_stats for more details.
```

Frankly, this error message is pretty awful. Hopefully, it is clear
that if `lemma_ship_within_days'_ships_within_days` *can* be applied
in the body of `if total_days > days` then postcondition holds. This
should lead us to suspect that the problem is that F\* cannot prove
the preconditions of `lemma_ship_within_days'_ships_within_days` holds
at this point. Let's add a temporary assert statement to check on the
precondition:
```
// Error
let rec lemma_ship_within_days'_ships_within_days (weights: weight_list) 
                                                  (days: nat{days > 0})
                                                  (min_cap: nat{min_cap >= min_bound weights})
                                                  (max_cap: nat{max_cap >= min_cap})
  : Lemma 
    (requires min_bound_invariant weights min_cap days /\ 
              max_bound_invariant weights max_cap days)
    (ensures (days_to_ship weights (ship_within_days' weights days min_cap max_cap)) <= days /\
             is_minimal weights (ship_within_days' weights days min_cap max_cap) days)
    (decreases max_cap - min_cap)
  = 
  if min_cap = max_cap then
     ()
  else 
    let middle_cap = (min_cap + max_cap) / 2 in
    let total_days = days_to_ship weights middle_cap in
    if total_days > days then (
       assert (min_bound_invariant weights (middle_cap + 1) days);
       lemma_ship_within_days'_ships_within_days weights days (middle_cap + 1) max_cap
    ) else (
        admit ()
    )
```

F\* still prints an assertion failed error, but now it points to the
line checking the precondition. So, we know that the problem is that
F\* cannot prove `min_bound_invariant` on `(middle_cap + 1)`. We know
that `maximum_bound_invariant` must continue to hold.

Observe that `min_bound_invariant` holds because `days_to_ship` is
decreasing: If we decrease the capacity, we will increase the days to
ship, and the condition `if total_days > days` already has proven that
we cannot ship at the capacity `middle_cap`. We just need to show F\*
these facts are true:
```
let rec lemma_days_to_ship_is_decreasing'' (weights: weight_list) 
                                           (cap: nat{cap >= (max_elt weights)})
                                           (ccap: nat{ccap <= cap})
                                           (ccap1: nat{ccap1 > ccap /\ ccap1 <= cap + 1})
  : Lemma (ensures days_to_ship' weights (cap + 1) ccap1 <= (days_to_ship' weights cap ccap))
  =
  match weights with
    | [w] -> ()
    | x::xs -> 
      if x <= ccap && x <= ccap1 then
        lemma_days_to_ship_is_decreasing'' xs cap (ccap - x) (ccap1 - x)
      else if x > ccap && x <= ccap1 then
        lemma_days_to_ship_is_decreasing' xs cap cap (ccap1 - x)
      else if x > ccap && x >= ccap1 then
        lemma_days_to_ship_is_decreasing'' xs cap cap (cap + 1)

and lemma_days_to_ship_is_decreasing' (weights: weight_list) 
                                      (cap: nat{cap >= (max_elt weights)})
                                      (ccap: nat{ccap <= cap})
                                      (ccap1: nat{ccap1 <= cap + 1})
  : Lemma (ensures days_to_ship' weights (cap + 1) ccap1 <= 1 + (days_to_ship' weights cap ccap))
  =
  match weights with
    | [w] -> ()
    | x::xs ->
      if x <= ccap && x <= ccap1 then
        lemma_days_to_ship_is_decreasing' xs cap (ccap - x) (ccap1 - x)
      else if x > ccap && x > ccap1 then
        lemma_days_to_ship_is_decreasing' xs cap cap (cap + 1)
      else if x > ccap && x <= ccap1 then
        lemma_days_to_ship_is_decreasing' xs cap cap (ccap1 - x)
      else
        // I.e., x <= ccap && x > ccap1
        lemma_days_to_ship_is_decreasing'' xs cap (ccap - x) (cap + 1)

let lemma_days_to_ship_is_decreasing (weights: weight_list) 
                                     (cap: nat{cap >= (max_elt weights)})
                                     (c_cap: nat{c_cap <= cap})
  : Lemma (ensures days_to_ship' weights (cap + 1) (c_cap + 1) <= days_to_ship' weights cap c_cap)
  =
  lemma_days_to_ship_is_decreasing'' weights cap c_cap (c_cap + 1)
```

Despite the coinductive proof, this is a simple argument. The theorem
that we are primarily interested in is
`lemma_days_to_ship_is_decreasing''`. This follows from
induction. There is a wrinkle, though: In the `else if x > ccap && x
<= ccap1` branch. In this case, the preconditions of
`lemma_days_to_ship_is_decreasing''` are no longer met. So, we use
coinduction to show that `days_to_ship' weights (cap + 1) ccap1 <= 1 +
(days_to_ship' weights cap ccap)`. Then, since `days_to_ship weights
cap ccap = 1 + days_to_ship xs cap cap`, F\* is automatically
able to cancel the `1`s and prove our theorem. A similar argument
applies to `lemma_days_to_ship_is_decreasing'`. 

But even armed with this theorem, F\* *still* can't prove the
precondition. Try it. We'll have to go even further:
```
let lemma_days_to_ship_is_decreasing_full (weights: weight_list) (cap: nat{cap >= (max_elt weights)})
  : Lemma (ensures days_to_ship weights (cap + 1) <= days_to_ship weights cap)
  = 
  lemma_days_to_ship_is_decreasing weights cap cap


let rec lemma_days_to_ship_is_decreasing2 (weights: weight_list) (c: nat{c >= min_bound weights})
  : Lemma (ensures (forall (x : nat) . x >= min_bound weights /\ x < c ==> 
                      days_to_ship weights x >= days_to_ship weights c))
  = if c > min_bound weights then (
       lemma_days_to_ship_is_decreasing_full weights (c -1);
       lemma_days_to_ship_is_decreasing2 weights (c - 1)
    )

let rec lemma_ship_within_days'_ships_within_days (weights: weight_list) 
                                                  (days: nat{days > 0})
                                                  (min_cap: nat{min_cap >= min_bound weights})
                                                  (max_cap: nat{max_cap >= min_cap})
  : Lemma 
    (requires min_bound_invariant weights min_cap days /\ 
              max_bound_invariant weights max_cap days)
    (ensures (days_to_ship weights (ship_within_days' weights days min_cap max_cap)) <= days /\
             is_minimal weights (ship_within_days' weights days min_cap max_cap) days)
    (decreases max_cap - min_cap)
  = 
  if min_cap = max_cap then
     ()
  else 
    let middle_cap = (min_cap + max_cap) / 2 in
    let total_days = days_to_ship weights middle_cap in
    if total_days > days then (
       lemma_days_to_ship_is_decreasing2 weights middle_cap;
       lemma_ship_within_days'_ships_within_days weights days (middle_cap + 1) max_cap
    ) else (
        admit ()
    )
```

As you might guess, F\* has a similar problem with the
`max_bound_invariant`. The problem is that the invariant requires
*all* capacities greater than `max_cap` to ship in less than or equal
to `days`, but our decreasing lemma only applies to `max_cap + 1`. Our
proof strategy is to use induction to extend our original decreasing lemma to show
$\forall k : \mathbb{N} . days\\_to\\_ship~ weights~ (capacity + k) <=
days\\_to\\_ship~ weights~ capacity$.
This argument convinces F\*:
```

let rec lemma_days_to_ship_is_decreasing3'' (w: weight_list) (c : nat{c >= min_bound w}) (k: nat)
  : Lemma (ensures days_to_ship w (c + k) <= days_to_ship w c)
  = 
  if k = 0 then ()
  else (
    lemma_days_to_ship_is_decreasing_full w (c + k - 1);
    lemma_days_to_ship_is_decreasing3'' w c (k - 1)
  )

let lemma_days_to_ship_is_decreasing3' (w: weight_list) (c : nat{c >= min_bound w})
  : Lemma (ensures forall (k : nat) . days_to_ship w (c + k) <= days_to_ship w c)
  = 
  assert (forall (w: weight_list) (c: nat{c >= min_bound w}) (k : nat) . 
            days_to_ship w (c + k) <= days_to_ship w c)
  by (
    let w = forall_intros () in
    mapply (`lemma_days_to_ship_is_decreasing3'' )
  )

let lemma_add_definition (c:nat)
  : Lemma (ensures (forall (x : nat) . x >= c ==> (exists (k : nat) . x = k + c)))
  =
  assert (forall (x : nat) . x >= c ==> x - c >= 0 /\ x - c + c = x)

let lemma_days_to_ship_is_decreasing3 (weights: weight_list) (c: nat{c >= min_bound weights})
  : Lemma (ensures forall (x : nat) . x >= c ==> days_to_ship weights x <= days_to_ship weights c)
  =
    lemma_days_to_ship_is_decreasing3' weights c;
    lemma_add_definition c


let rec lemma_ship_within_days'_ships_within_days (weights: weight_list) 
                                                  (days: nat{days > 0})
                                                  (min_cap: nat{min_cap >= min_bound weights})
                                                  (max_cap: nat{max_cap >= min_cap})
  : Lemma 
    (requires min_bound_invariant weights min_cap days /\ 
              max_bound_invariant weights max_cap days)
    (ensures (days_to_ship weights (ship_within_days' weights days min_cap max_cap)) <= days /\
             is_minimal weights (ship_within_days' weights days min_cap max_cap) days)
    (decreases max_cap - min_cap)
  = 
  if min_cap = max_cap then
     ()
  else 
    let middle_cap = (min_cap + max_cap) / 2 in
    let total_days = days_to_ship weights middle_cap in
    if total_days > days then (
       lemma_days_to_ship_is_decreasing2 weights middle_cap;
       lemma_ship_within_days'_ships_within_days weights days (middle_cap + 1) max_cap
    ) else (
      lemma_days_to_ship_is_decreasing3 weights middle_cap;
      lemma_ship_within_days'_ships_within_days weights days min_cap middle_cap
    )
```

As an exercise: It is up to the reader to demonstrate that the
`min_bound_invariant` and `max_bound_invariant` hold under the initial
conditions set by `ship_within_days`.

## Takeaways <a name="takeaways"></a>
I found F\* to be immensely usable. While error messages are not the
best, this is really a limitation of the underlying SMT solver. From
experience, Z3's unsatisfiable cores are complex to handle. And moving
back and forth from the high level language F\* provides and SMT is
challenging. But this definitely an area that needs improvement.

The ecosystem of F\* is young. The resources I've used are:
* The source code on [GitHub](https://github.com/FStarLang/FStar). The
  standard library is not really documented today. But, due to the
  presence of preconditions/postconditions, the source is quite
  readable. I have learned to make it a habit to consult the source
  code for lemmas, like `FStar.List.Tot.Properties.fold_left_monoid`.
* The F\* tutorial contains some decent examples.
* Read the papers. It's okay to not understand everything -- learn
  what you can, save the paper, and eventually you'll come back to and
  things will make more sense.
* The book "Types and Programming Languages" provides good background
  PL theory.
* The book "The Little Typer" provides a good background on dependent types.

I hope that this post has inspired you to give F\* a try.
