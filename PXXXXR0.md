---
title: "Relax narrowing for lossless integer → ISO/IEC 60559 binary floating conversions"
document: P0000R0
date: today
audience:
  - Evolution Working Group
author:
  - name: Dmytro Ovdiienko
    email: Dmytro.Ovdiienko@gmail.com
toc: true
toc-depth: 2
---

# PXXXXR0 — Relax narrowing for lossless integer → ISO/IEC 60559 binary floating conversions

## Abstract

This paper proposes relaxing list-initialization narrowing rules so that an implicit conversion from an integer type to a floating-point type is not considered narrowing when the destination floating-point type adheres to ISO/IEC 60559, uses a binary radix, and has sufficient precision to exactly represent all values of the source integer type.

## Motivation

List-initialization prohibits narrowing conversions to prevent silent loss of information. Under current wording, conversions from integer types to floating-point types are generally considered narrowing, except in the case of constant expressions whose values round-trip exactly.

In practice, this rule leads to surprising and overly restrictive behavior in generic code once values are forwarded through an intermediate object or function parameter and therefore cease to be constant expressions. For example:

```cpp
template<typename T>
struct X {
  T value;

  template<typename U>
  X(U const& other) : value{other} {}
};

X<double> v = 1; // ill-formed: narrowing conversion from int to double
```

Although the value `1` is exactly representable in `double`, the conversion is rejected because `other` is a named lvalue and the constant-expression exception in [dcl.init.list] no longer applies. As a result, code that is provably lossless and routinely relied upon in practice becomes ill-formed solely due to the mechanics of value propagation, not due to any actual risk of information loss. This pattern arises naturally in forwarding constructors, wrapper types, and generic abstractions that propagate values through intermediate parameters.

On platforms where the destination floating-point type follows ISO/IEC 60559 binary semantics, it is a well-understood property that all values of an integer type whose number of value bits does not exceed the floating-point precision are exactly representable. In such cases, integer-to-floating conversions are provably lossless for all possible runtime values, independent of whether the source expression is a constant expression or an lvalue.

## Design

This proposal addresses the mismatch described in the Motivation section by refining the narrowing rules to recognize certain integer-to-floating conversions as non-narrowing based on a simple, type-based criterion. Doing so restores the intended meaning of brace-initialization as a guard against actual loss of information, while avoiding unnecessary rejection of correct and widely-used generic code.

### Scope and non-goals

This proposal is intentionally limited in scope. It does not attempt to address cases such as:

```cpp
X<short> v = 1;
```

where `1` is an `int` literal and the conversion to `short` may be lossy for some values. While it could be desirable in principle for the language to select a “least applicable” integer type or otherwise reason about literal ranges, such behavior is not the intent of this proposal. More importantly, changing the rules for integer-to-integer list-initialization in this way could render existing, currently well-formed code ill-formed. This paper therefore confines itself to integer-to-floating conversions where a lossless conversion for all values can be established purely at the type level.

The proposal relies on existing semantic properties of integer and floating-point types as defined by the core language and the floating-point model referenced by the standard (in particular ISO/IEC 60559). For clarity and conciseness, this paper refers to these properties using the corresponding library traits (`numeric_limits<T>::digits`, `numeric_limits<T>::radix`, and `numeric_limits<T>::is_iec559`), but these names are used for *exposition only*; the intent is to rely on the underlying language-defined properties, not to introduce a dependency of the core language rules on the standard library.

For ISO/IEC 60559 binary floating-point types, the condition that the precision of the destination floating-point type is at least the number of value bits of the source integer type guarantees exact representability of all values of the integer type `I` in the floating-point type `F`.

Throughout the remainder of this paper (including examples and discussion sections), references to `numeric_limits`, `digits(F)`, and related library traits are used purely as concise, expository shorthand for these underlying core-language properties, and do not imply a dependency of the language rules on the standard library.

## Proposal

An integer-to-floating conversion is not narrowing when:

- the destination floating-point type conforms to ISO/IEC 60559,
- the destination floating-point type has a binary radix, and
- the destination floating-point type has a precision of at least as many value bits as the source integer type.

## Proposed wording

### [dcl.init.list]

Replace bullet (7.3) with the following:

> (7.3) from an integer type or unscoped enumeration type to a floating-point type, except where
>
> (7.3.1) the source is a constant expression and the actual value after conversion will fit into the target type and will produce the original value when converted back to the original type, or
>
> (7.3.2) the source is an integer type *I* and the destination is a floating-point type *F* such that *F* conforms to ISO/IEC 60559, has a binary radix, and has a precision of at least as many value bits as the source integer type *I*.

## Feature-test macro

For consistency with other core-language changes, this proposal introduces a feature-test macro to allow programs to detect support for the relaxed narrowing rules in [dcl.init.list].

```cpp
#define __cpp_narrowing_integer_to_floating 20YYMML
```

The macro is defined if and only if the implementation applies the updated narrowing rules for integer-to-floating list-initialization as specified in this paper.

## Examples

```cpp
int32_t i = runtime();
double d{i};   // well-formed under this proposal

int64_t j = runtime();
double e{j};   // still narrowing
```

The following example demonstrates a converting constructor that accepts an `Other` type and uses list-initialization internally:

```cpp
struct Wrapper {
  double value;

  template <class Other>
  explicit Wrapper(Other other)
    : value{other} {} // OK when Other is an integer type with digits <= digits(double)
};

int32_t k = runtime();
Wrapper w{k};          // well-formed under this proposal
```

## Discussion: radix restriction and non-binary floating-point

This proposal intentionally restricts its scope to ISO/IEC 60559 binary floating-point types. For binary floating-point, the relationship between significand precision and exact integer representability is simple and well-defined. For non-binary radices, including IEC 60559 decimal formats, exact representability depends on additional format properties not captured solely by `digits`.

Extending this proposal to non-binary floating-point formats would require a different and more complex criterion and is therefore left as future work.

## Portability considerations

The proposal does not render any currently well-formed program ill-formed. It may, however, make some previously ill-formed brace-initializations well-formed on implementations where the integer-to-floating conversion is provably lossless for all values of the source type. Programs that require portability across implementations with differing floating-point semantics may continue to use explicit casts or non-list-initialization forms.

It is worth noting that, on implementations where `double` conforms to ISO/IEC 60559 (which is the case for the vast majority of contemporary platforms), developers already routinely rely on integer-to-`double` conversions being exact and use `static_cast` or non-list-initialization to silence narrowing diagnostics. Such code already encodes an assumption about the floating-point semantics of the target platform. When ported to an implementation that does not provide ISO/IEC 60559 semantics, this assumption may no longer hold and code may continue to compile but exhibit different or incorrect behavior, regardless of this proposal. The present change does not introduce a new portability hazard; it makes an existing, widely relied-upon assumption explicit and checkable at the language level.

As with existing narrowing rules, acceptance of certain list-initializations may differ across implementations. Existing C++ already permits platform-dependent well-formedness for list-initialization. For example, the following code may be well-formed on an implementation where `int` is at least 32 bits, but ill-formed on an implementation where `int` is 16 bits, because the conversion from `std::int32_t` to `int` becomes narrowing for non-constant expressions:

```cpp
#include <cstdint>

std::int32_t runtime32();

std::int32_t i = runtime32();
int x{i}; // may be ill-formed if int is 16 bits
```

## Relationship to `<stdfloat>` and `std::floatNN_t`

C++23 introduces fixed-width floating-point types in `<stdfloat>`, such as `std::float16_t`, `std::float32_t`, `std::float64_t`, and `std::float128_t`, when provided by the implementation. These types are specified as extended floating-point types corresponding to ISO/IEC 60559 interchange formats.

Where such types are available, they provide a clear and portable way to refer to floating-point formats with known precision and representation properties. On these implementations, the conditions of this proposal naturally permit integer-to-floating list-initialization when the source integer type has no more value bits than the precision of the corresponding `std::floatNN_t` type.

This proposal does not special-case `std::floatNN_t`, but instead relies on semantic properties expressed via `std::numeric_limits`. As a result, the rule applies uniformly to `std::floatNN_t` and to standard floating-point types (`float`, `double`, `long double`) when they satisfy the same ISO/IEC 60559 and precision requirements. This avoids fragmenting the language rules while still allowing users who require predictable floating-point formats to prefer `<stdfloat>` types where available.

## Related work and prior art

The narrowing rules for list-initialization were originally introduced as part of C++11, with the goal of preventing silent loss of information during initialization. At that time, the rules were intentionally conservative, particularly for conversions from integer types to floating-point types, and included a limited exception for constant expressions whose values could be proven to round-trip exactly.

Since then, practice has converged further on ISO/IEC 60559 binary floating-point for mainstream targets, and the ecosystem has accumulated substantial experience with list-initialization. This makes it easier to justify refining the original rule without increasing risk for existing code. Several accepted proposals in this area have focused on making floating-point properties explicit and usable by programs. Notably, the ISO/IEC TS 18661 series (integrated into C++17 and later) and subsequent core and library work established a well-defined model for IEC 60559 floating-point behavior. More recently, proposals leading to C++23’s `<stdfloat>` header (for example, P1467 and related papers) introduced fixed-width floating-point types corresponding to IEC 60559 interchange formats, making precision and representation properties explicit and portable.

In parallel, a number of proposals have explored *library-level* facilities for reasoning about value-preserving and narrowing conversions. Proposal P2509 focuses on value-preserving conversions by providing traits that allow programs to determine whether a conversion preserves the represented value. Proposal P0870 addresses narrowing conversions by proposing traits that allow programs to reason about whether a conversion may be narrowing.

These library facilities are complementary to the present proposal. They enable developers to write more robust and explicit code by detecting and constraining conversions at the library level. However, they cannot integrate with the core language’s narrowing rules or affect the semantics of brace-initialization itself.

This proposal addresses that remaining gap by embedding a value-preserving criterion directly into the core language’s narrowing rules for a specific, well-defined class of conversions. In this sense, the present proposal acts as a *last-level guard*: it ensures that brace-initialization continues to enforce the absence of information loss by default, while library-level traits such as those proposed in P2509 and P0870 provide additional tools for developers to express and enforce conversion policies explicitly in generic code.

This paper therefore builds on the original intent of list-initialization narrowing by refining the rules to distinguish between potentially lossy conversions and those that are guaranteed to be exact for all values, using properties already defined by the core language and the ISO/IEC 60559 floating-point model.

## Why a library-only solution is insufficient

It is possible to express the predicate underlying this proposal as a constrained template function (for example, a hypothetical `lossless_cast<F>(i)` that is only enabled when the destination floating-point type can exactly represent all values of the source integer type). Such a utility can be useful as an illustration of the rule or as a workaround on older language versions.

However, a library-only approach cannot replace a core-language change in this area. List-initialization is the only mechanism in C++ that enforces non-narrowing conversions by default, and it is widely used as a semantic signal that an initialization is intended to be free of information loss. Requiring an explicit helper function at each call site would make this guarantee opt-in and verbose, weakening the safety-by-default property that list-initialization was designed to provide.

This proposal therefore targets the language rule itself, so that brace-initialization continues to express a uniform and immediate guarantee of lossless conversion without requiring additional boilerplate or user intervention.


