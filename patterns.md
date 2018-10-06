---
title: "Pattern Matching"
document: D1260R0
date: 2018-05-22
audience: Evolution
author:
  - name: Michael Park
    email: <mcypark@gmail.com>
---

# Introduction

As algebraic data types gain better support in C++ with facilities such as
`tuple` and `variant`, the importance of mechanisms to interact with them have
increased. While mechanisms such as `apply` and `visit` have been added, their
usage is quite complex and limited even for simple cases. Pattern matching is
a widely adopted mechanism across many programming languages to interact with
algebraic data types that can help greatly simplify C++. Examples of programming
languages include text-based languages such as SNOBOL back in the 1960s,
functional languages such as Haskell and OCaml, and "mainstream" languages such
as Scala, Swift, and Rust.

Inspired by P0095 [@P0095] --- which proposed pattern matching and
language-level variant simulteneously --- this paper explores a possible
direction for pattern matching only, and does not address language-level
variant design. This is in correspondence with a straw poll from Kona 2015,
which encouraged exploration of a full solution for pattern matching.
SF: 16, WF: 6, N: 5, WA: 1, SA: 0.

# Motivation and Scope

Virtually every program involves branching on some predicates applied to a value
and conditionally binding names to some of its components for use in subsequent
logic. Today, C++ provides two types of selection statements: the `if` statement
and the `switch` statement.

Since `switch` statements can only operate on a _single_ integral value and
`if` statements operate on an _arbitrarily_ complex boolean expression, there is
a significant gap between the two constructs even in inspection of
the "vocabulary types" provided by the standard library.

In C++17, structured binding declarations [@P0144] introduced the ability to
concisely bind names to components of a `tuple`-like value. The proposed
direction of this paper aims to naturally extend this notion by performing
__structured inspection__ prior to forming the __structured bindings__ with
a third selection statement: the `inspect` statement. The goal of the `inspect`
statement is to bridge the gap between `switch` and `if` statements with
a __declarative__, __structured__, __cohesive__, and __composable__ mechanism.

# Before/After Comparisons

## Matching Integrals

+------------------------------------------------+-------------------------------------------------+
| __Before__                                     | __After__                                       |
+================================================+=================================================+
| ```cpp                                         | ```cpp                                          |
| switch (x) {                                   | inspect (x) {                                   |
|   case 0: std::cout << "Got zero";             |   0: std::cout << "Got zero";                   |
|   case 1: std::cout << "Got one";              |   1: std::cout << "Got one";                    |
|   default: std::cout << "Don't care";          |   _: std::cout << "Don't care";                 |
| }                                              | }                                               |
| ```                                            | ```                                             |
+------------------------------------------------+-------------------------------------------------+

## Matching Strings

+--------------------------------------+----------------------------------------+
| __Before__                           | __After__                              |
+======================================+========================================+
| ```cpp                               | ```cpp                                 |
| if (s == "A") {                      | inspect (s) {                          |
|   std::cout << "Got A";              |   "A": std::cout << "Got A";           |
| } else if (s == "B") {               |   "B": std::cout << "Got B";           |
|   std::cout << "Got B";              | }                                      |
| }                                    | ```                                    |
| ```                                  |                                        |
+--------------------------------------+----------------------------------------+

## Matching Tuples

+--------------------------------------+----------------------------------------+
| __Before__                           | __After__                              |
+======================================+========================================+
| ```cpp                               | ```cpp                                 |
| auto&& [x, y] = t;                   | inspect (t) {                          |
| if (x == 0 && y == 0) {              |   [0, 0]: std::cout << "On origin";    |
|   std::cout << "On origin";          |   [0, y]: std::cout << "On x-axis";    |
| } else if (x == 0) {                 |   [x, 0]: std::cout << "On y-axis";    |
|   std::cout << "On x-axis";          |   [x, y]: std::cout << x << ',' << y;  |
| } else if (y == 0) {                 | }                                      |
|   std::cout << "On y-axis";          | ```                                    |
| } else {                             |                                        |
|   std::cout << x << ',' << y;        |                                        |
| }                                    |                                        |
| ```                                  |                                        |
+--------------------------------------+----------------------------------------+

\pagebreak

## Matching Variants

+------------------------------------------------+-------------------------------------------------+
| __Before__                                     | __After__                                       |
+================================================+=================================================+
| ```cpp                                         | ```cpp                                          |
| struct visitor {                               | inspect (v) {                                   |
|   void operator()(int i) const {               |   <int> i: strm << "Got int: " << i;            |
|     strm_ << "Got int: " << i;                 |   <float> f: strm << "Got float: " << f;        |
|   }                                            | }                                               |
|   void operator()(float f) const {             | ```                                             |
|     strm_ << "Got float: " << f;               |                                                 |
|   }                                            |                                                 |
|   std::ostream& strm_;                         |                                                 |
| };                                             |                                                 |
| std::visit(visitor{strm}, v);                  |                                                 |
| ```                                            |                                                 |
+------------------------------------------------+-------------------------------------------------+

## Evaluating Expressions

Given the following definition:

```cpp
struct Expr;
struct Neg { std::shared_ptr<Expr> expr; };
struct Add { std::shared_ptr<Expr> lhs, rhs; };
struct Mul { std::shared_ptr<Expr> lhs, rhs; };
struct Expr : std::variant<int, Neg, Add, Mul> {
  using variant::variant;
};
```

+--------------------------------------------+-------------------------------------------------+
| __Before__                                 | __After__                                       |
+============================================+=================================================+
| ```cpp                                     | ```cpp                                          |
| int eval(const Expr& expr) {               | int eval(const Expr& expr) {                    |
|   struct visitor {                         |   inspect (expr) {                              |
|     int operator()(int i) const {          |     <int> i: return i;                          |
|       return i;                            |     <Neg> [e]: return -eval(*e);                |
|     }                                      |     <Add> [l, r]: return eval(*l) + eval(*r);   |
|     int operator()(const Neg& n) const {   |     <Mul> [l, r]: return eval(*l) * eval(*r);   |
|       return -eval(*n.expr);               |   }                                             |
|     int operator()(const Add& a) const {   | }                                               |
|       return eval(*a.lhs) + eval(*a.rhs);  | ```                                             |
|     }                                      |                                                 |
|     int operator()(const Mul& m) const {   |                                                 |
|       return eval(*m.lhs) * eval(*m.rhs);  |                                                 |
|     }                                      |                                                 |
|   };                                       |                                                 |
|   return std::visit(visitor{}, expr);      |                                                 |
| }                                          |                                                 |
| ```                                        |                                                 |
+--------------------------------------------+-------------------------------------------------+

# Design Overview

## Basic Syntax

> | `inspect (` _init-statement~opt~_ _condition_ `) {`
> |     _pattern_ _guard~opt~_ `:` _statement_
> |     _pattern_ _guard~opt~_ `:` _statement_
> |     ...
> | `}`

> | _guard:_
> |     `if (` _expression_ `)`

## Basic Model

Within the parentheses, the `inspect` statement is equivalent to `if` and
`switch` statements except that no conversion nor promotion takes place
in evaluating the value of its condition.

When the `inspect` statement is executed, its condition is evaluated and matched
in order (first match semantics) against each pattern. If a pattern successfully
matches the value of the condition and the boolean expression in the guard
evalutes to `true` (or if there is no guard at all), control is passed to the
statement following the matched pattern label. If the guard expression evaluates
to `false`, control flows to test the subsequent pattern. If no pattern matches,
none of the statements are executed.

## Types of Patterns

### Primary Patterns

#### Identifier Pattern

The identifier pattern has the form:

> _identifier_

_Requirements:_ None.

_Matches:_ Any value.

The introduced names are lvalues referring to corresponding components.
The name is in scope from its point of declaration until the end of
the statement following the pattern label. Identifiers therefore cannot be
repeated within the same pattern but can reused in the subsequent pattern.

```cpp
inspect (v) {
    x: cout << x;
//  ^ identifier pattern
}
```

[ _Note:_ If the identifier pattern is used as a top-level pattern,
          it has the same syntax as a `goto` label. ]

#### Constant Pattern

The constant pattern has the form:

> _constant-expression_

except the _id-expression_, due to its ambiguity with the identifier pattern.

`(id)` or `+id` can be used for disambiguation.

Let `c` be the constant expression and `v` the value being matched.

_Requires:_ The expression `strong_equal(c, v)` is valid.

_Matches:_ If `strong_equal(c, v) == strong_equality::equal` is `true`.

```cpp
inspect (n) {
    0: cout << "got zero!";
    1: cout << "got one!";
//  ^ constant pattern
}
```

### Compound Patterns

#### Structured Binding Pattern

The structured binding pattern has the form:

> `[` _pattern_~0~`,` _pattern_~1~`,` ...`,` _pattern_~N~` ]`

Let `v` be the value being matched.

_Requires:_ The declaration `auto&& [e`~0~`, e`~1~`,` ...`, e`~N~`] = v;`
            is valid, where each `e`_~i~_ is a unique _identifier_.

_Matches:_ If _pattern_~i~ matches `e`~i~ for all $0 \leq i \leq N$ in
           `auto&& [e`~0~`, e`~1~`,` ...`, e`~N~`] = v;`.

```cpp
inspect (point) {
    [0, 0]: cout << "origin\n";
    [0, y]: cout << "on x-axis\n";
//      ^ identifier pattern
    [x, 0]: cout << "on y-axis\n";
//      ^ constant pattern
    [x, y]: cout << x << ',' << y << '\n';
//  ^^^^^^ structured binding pattern
}
```

#### Alternative Pattern

The alternative pattern has the form:

> `<Alternative>` _pattern_

Let `v` be the value being matched and `V` be its type.

__Case 1: Variant-Like__

> If `std::variant_size<V>` is a complete type, the expression `std::variant_size<V>::value`
> shall be a well-formed integral constant expression.
>
> Let `D(v)` be a member `v.discriminator()` or else a non-member ADL-only `discriminator(v)`.
>
> Let `G<I>(v)` be a member `v.get<I>()` or else a non-member ADL-only `get<I>(v)`.
>
> [ _Note:_ These are similar to how `get` is looked up for structured binding declarations. ]
>
> Let `d` be the value of `D(v)`, and `alternative` be a reference to the stored
> alternative of type `std::variant_alternative_t<d, V>` initialized by `G<d>(v)`.
>
> _Matches:_ We have the following 4 cases:
>
> __Case 1.1: `Alternative` is a value__
>
> > If `d` has the same value as `Alternative` and _pattern_ matches `alternative`.
>
> __Case 1.2: `Alternative` is a type__
>
> > If `std::is_same<std::variant_alternative_t<d, V>, Alternative>::value` is `true` and _pattern_ matches `alternative`.
>
> __Case 1.3: `Alternative` is a concept__
>
> > If `Alternative<std::variant_alternative_t<d, V>>()` is `true` and _pattern_ matches `alternative`.
>
> __Case 1.4: `Alternative` is `auto`__
>
> > If _pattern_ matches `alternative`.

__Case 2: Polymorphic Type__

...

# Impact on the Standard

This is a language extension to introduce a new selection statement: `inspect`.

# Proposed Wording

## Syntax

Add to __\S8.4 [stmt.select]__ of ...

\pnum{1}Selection statements choose one of several flows of control.

> | _selection-statement:_
> |     `if constexpr`_~opt~_ `(` _init-statement~opt~_ _condition_ `)` _statement_
> |     `if constexpr`_~opt~_ `(` _init-statement~opt~_ _condition_ `)` _statement_ `else` _statement_
> |     `switch (` _init-statement~opt~_ _condition_ `)` _statement_
> |     \added `inspect (` _init-statement~opt~_ _condition_ `) {` _inspect-case-seq_ `}` \unchanged
>
> \added
> | _inspect-case-seq:_
> |     _inspect-case_
> |     _inspect-case-seq_ _inspect-case_
>
> | _inspect-case:_
> |     _attribute-specifier-seq~opt~_ _inspect-pattern_ _inspect-guard~opt~_ `:` _statement_
>
> | _inspect-pattern:_
> |     _constant-pattern_
> |     _identifier_
> |     _wildcard-pattern_
> |     _structured-binding-pattern_
> |     _alternative-pattern_
>
> | _inspect-guard:_
> |     `if (` _condition_ `)`
> \unchanged

# Design Decisions

## Conceptual Model: Extending Structured Bindings

The design intends to be consistent and naturally extend the notions introduced
by structured bindings. That is, The subobjects are __referred__ to rather than
being assigned into new variables.

## `inspect` vs `switch`

This proposal introduces a new `inspect` statement rather than trying to extend
the `switch` statement for the following reasons:

  - `switch` allows the `case` labels to appear anywhere, which hinders pattern
    matching's aim for __structured__ inspection.
  - The fall-through semantics of `switch` generally results in `break` being
    attached to every case.
  - `switch` is purposely restricted to integrals for __guaranteed__ efficiency.
    The primary goal of pattern matching in this paper is expressivity, while
    being at least as efficient as the naively hand-written code.

## Statement vs Expression

This paper diverges from P0095 [@P0095] in that it proposes to add `inspect` as
a statement only rather than trying to double as a statement and an expression.

The main reason here is that the semantic differences between the statement and
expression forms are not trivial.
  1. In the case where none of the cases match, the statement form simply skips
     over the entire statement à la `switch`, whereas the expression form throws
     an exception since it is required to yield a value.
  2. Resulting type of the statement form of `inspect` within an immediately-
     invoked-lambda is required to be explicitly specified, or is determined by
     the first `return` statement. In contrast, the expression form will
     probably need to use `std::common_type_t<Ts...>` where `Ts...` are types of
     `N` expressions to be consistent with the ternary operator.

While an expression form of `inspect` would be useful, the author believes that
it can and should be introduced later, with different syntax such as
`x inspect { /* ... */ }`. The proposed syntax in this paper is consistent with
every other statement in C++ today.

## Language vs Library

There have been three popular pattern matching libraries in existence today.
  - Mach7
  - Simple Match by jbandela
  - MPark.Patterns

The issue of introducing identifiers is burdensome enough that I believe it
justifies a language feature.

## Optimizations

Comparison elision?

...

## Ranges

...

## User-defined Patterns

...

# Examples

## Matching strings

```cpp
std::string s = "hello";
inspect (s) {
    "hello": std::cout << "hello";
    "world": std::cout << "world";
}
```

# Other Languages and Libraries

## C\#

## Rust

Constants: https://github.com/rust-lang/rfcs/blob/master/text/1445-restrict-constants-in-patterns.md

### Intersection of semantic / structural equality

## Scala

Scala Tutorial - Pattern Matching: https://www.youtube.com/watch?v=ULcpWn23waw
Matching Objects with Patterns: https://infoscience.epfl.ch/record/98468/files/MatchingObjectsWithPatterns-TR.pdf

### Extractors

## F\#

### Active Patterns

# Future Work

# Acknowledgements

Thank you to Agustín Bergé, Ori Bernstein, Alexander Chow, Louis Dionne,
Michał Dominiak, Eric Fiselier, Zach Laine, Jason Lucas, David Sankel,
Tony Van Eerd, and everyone else who contributed to the discussions, and
encouraged me to write this paper.

---
references:
  - id: N3449
    title: "Open and Efficient Type Switch for C++"
    issued:
      year: 2012
      month: 9
    author:
      family: Stroustrup
      given: Bjarne
    container-title: N3449
    URL: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3449.pdf
  - id: N3627
    title: "Relaxed switch statement"
    issued:
      year: 2013
      month: 2
    author:
      family: Yuan
      given: Zhihao
    container-title: N3627
    URL: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3627.html
  - id: P0095
    title: "Pattern Matching and Language Variants"
    issued:
      year: 2016
      month: 5
    author:
      family: Sankel
      given: David
    container-title: P0095
    URL: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0095r1.html
  - id: P0144
    title: "Structured bindings"
    issued:
      year: 2016
      month: 3
    author:
      - family: Sutter
        given: Herb
      - family: Stroustrup
        given: Bjarne
      - family: Dos Reis
        given: Gabriel
    container-title: P0144
    URL: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0144r2.pdf
---

<!--
Consistent Comparisons: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0515r3.pdf
Class Types in Non-Type Template Parameters: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0732r0.pdf
From F# to Scala Extractors: https://theburningmonk.com/2017/01/from-f-to-scala-extractors/


```
inspect (int i = 42) {
  0: std::cout << "foo";
  1: std::cout << "bar";
}
```

```
[&] -> std::string {
  inspect (i) {
    0: return "foo";
    1: return "bar";
  }
}()
```

```
inspect (std::lock_guard lock(mutex); i) {
  0 => std::cout << "foo";
  1 => std::cout << "bar";
}
```

```
inspect (pair<int, int> p = ...; p) {
  [x, 0]: { std::cout << "foo"; x; y; }
  [0, y]: std::cout << "bar";
  _: /* ... */;
}
```

```
inspect (std::variant<int, double> v = ...; v) {
  [0]: std::cout << "foo";
  // <pair<int, int>> [x, y]: std::cout << "bar";
}
```

```
inspect (std::variant<int, pair<int, int>> v = ...; v) {
  <int> x: std::cout << "foo";
  <pair<int, int>> [x, y]: std::cout << "bar";
}
```

```
inspect (std::variant<int, int> v = ...; v) {
  <0> x: std::cout << "foo";
  <1> x: std::cout << "bar";
}
```

```
inspect (const Base& b = ...) {
  <D1> [x] => std::cout << "foo";
  <D2> [x, y] => std::cout << "bar";
}
```

inspect (f()) {
  pair [int x, int y]

```
inspect (std::variant<int, pair<int, int>> v = ..., w = ...; std::make_tuple(v, w)) {
  [<0> x, <int> y]: std::cout << "foo";
  [<int> x, <pair<int, int>> [y, 0]]: std::cout << "bar";
  [<pair<int, int>> p, <int> y]: std::cout << "bar";
  [<pair<int, int>> [x, y], <pair<int, int>> p]: std::cout << "bar";
}
```

```
inspect (s) {
  "hello": ...;
  "foobar": ...;
  "(foo)*bar"reg: ...;
  std::string_view sv:
}
```

```
inspect (s) {
  email [local, domain]: ...;
  us_phone_number [area_code, _, _]: ...;
}
```

optional<Email> email(string_view);
optional<PhoneNumber> us_phone_number(string_view);

variant_of(email, us_phone_number);

variant<Email, PhoneNumber> parse(string_view sv) {
  if (auto x = email(sv)) {
    return *x;
  } else if (auto y = us_phone_number(sv)) {
    reutrn *y;
  }
}

- Syntax for matching `std::variant` and base class.
  - Given `inspect (v) { int n: stmt1; double d : stmt2;}`,
    `int n` and `double d` looks like declarations, but they are not.
- In order, determine if it's a match (e.g., dynamic type checking), then
  transform (e.g., dynamic_cast), coupling the "is-as" pattern.
  So, given a string, why not match if it's the form of email, then transform,
  into 2 parts then continue matching?

  if (is_email(s)) {  // is
    auto [local, domain] = s;  // as
  }

```
  auto [__x, __y] = t;
  switch (__x) {
    case 0: {
      auto&& y = __y;
      // A
      break;
    }
    default: {
      switch (__y) {
        case 0: {
          auto&& x = __x;
          // B
          break;
        }
        default: {
          // C
        }
      }
    }
  }

  if (__x == 0) {
    auto&& y = __y;
    // A
  } else if (__y == 0) {
    auto&& x = __x;
    // B
  } else {
    // C
  }

  inspect (t) {
    [0, y]: /* A */
    [x, 0]: /* B */
    [_, _]: /* C */
  }
```

```
  if (holds_alternative<int>(v)) {  // is
    int x = get<int>(v);  // as
  }

  Base& b = ...;
  if (auto* d_ptr = dynamic_cast<const Derived*>(&b)) {  // almost is-as
    const Derived& d = *d_ptr;  // as
  }

  variant<int, string> v = ...;
  if (int* i = get_if<int>(&v)) {  // almost is-as
    int x = *i;  // as
  }
```

```
template <typename T>
auto variant_index(T* p) { return p != nullptr; }

template <typename T>
auto variant_index(optional<T> const& o) { return o.has_value(); }

template <typename... Ts>
auto variant_index(std::variant<Ts...> const& v) { return v.index(); };

template <typename T>
struct variant_access<optional<T>, none> { static constexpr int index = 0; };

template <typename T>
struct variant_access<optional<T>, some> {
  static constexpr int index = 1;

  auto&& get(optional<T> const& o) { return *o; }
};

template <typename... Ts, typename Alt>
struct variant_access<variant<Ts...>, Alt>
  static constexpr int index = index_of<Alt, list<Ts...>>;

  auto&& get(variant<Ts...> const& v) { return get<Alt>(v); }
};

template <typename... Ts, size_t I>
struct variant_access<variant<Ts...>, I>
  static constexpr int index = I;

  auto&& get(variant<Ts...> const& v) { return get<I>(v); }
};

inspect (v) {
  <int> x: ...
  <string>: ...
}

inspect (tuple{o, o}) {
  [<some> x, <some> y]: ...
  none: ...
}
```
-->