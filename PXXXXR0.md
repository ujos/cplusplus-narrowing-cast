# PXXXXR0 — Relax narrowing for lossless integer → ISO/IEC 60559 binary floating conversions

## Abstract
This paper proposes relaxing list-initialization narrowing rules so that an implicit conversion from an integer type to a floating-point type is not considered narrowing when the destination floating-point type adheres to ISO/IEC 60559, uses a binary radix, and has sufficient precision to exactly represent all values of the source integer type.

## Motivation
List-initialization prohibits narrowing conversions to prevent silent loss of information. Under current wording, conversions from integer types to floating-point types are generally considered narrowing, except in the case of constant expressions whose values round-trip exactly.

On platforms where standard floating-point types or extended floating-point types conform to ISO/IEC 60559 and use a binary radix, the destination floating-point type can exactly represent all values of an integer type whose number of value bits does not exceed the floating-point precision. These conversions are provably lossless for all possible runtime values and should therefore not be considered narrowing.

## Design
The proposal relies exclusively on existing standard type traits:

- `std::numeric_limits<T>::digits`
- `std::numeric_limits<T>::radix`
- `std::numeric_limits<T>::is_iec559`

For ISO/IEC 60559 binary floating-point types, the condition `digits(F) >= digits(I)` guarantees exact representability of all values of the integer type `I` in the floating-point type `F`.

## Proposal
An integer-to-floating conversion is not narrowing when:

- the destination type `F` satisfies `numeric_limits<F>::is_iec559`,
- `numeric_limits<F>::radix == 2`, and
- `numeric_limits<F>::digits >= numeric_limits<I>::digits`, where `I` is the source integer type.

## Proposed wording

### [dcl.init.list]

Modify the definition of narrowing conversions as follows.

Replace the bullet covering conversions from integer types to floating-point types with wording equivalent to:

> A conversion from an integer type to a floating-point type is narrowing, except where:
> - the source is a constant expression and the converted value is exactly representable and round-trips correctly, or
> - the source is an integer type `I`, the destination is a floating-point type `F`, `numeric_limits<F>::is_iec559` is true, `numeric_limits<F>::radix == 2`, and `numeric_limits<F>::digits >= numeric_limits<I>::digits`.

## Examples

```cpp
int32_t i = runtime();
double d{i};   // well-formed under this proposal

int64_t j = runtime();
double e{j};   // still narrowing
```

## Discussion: radix restriction and non-binary floating-point

This proposal intentionally restricts its scope to ISO/IEC 60559 binary floating-point types. For binary floating-point, the relationship between significand precision and exact integer representability is simple and well-defined. For non-binary radices, including IEC 60559 decimal formats, exact representability depends on additional format properties not captured solely by `digits`.

Extending this proposal to non-binary floating-point formats would require a different and more complex criterion and is therefore left as future work.

## Portability considerations

This proposal intentionally makes the acceptance of certain list-initializations conditional on semantic properties of the destination floating-point type. This is consistent with existing narrowing rules, which already depend on implementation-defined characteristics such as floating-point precision, constant-expression evaluation, and representation details.

The proposal does not render any currently well-formed program ill-formed. It may, however, make some previously ill-formed brace-initializations well-formed on implementations where the integer-to-floating conversion is provably lossless for all values of the source type. Programs that require portability across implementations with differing floating-point semantics may continue to use explicit casts or non-list-initialization forms.

## Relationship to `<stdfloat>` and `std::floatNN_t`

C++23 introduces fixed-width floating-point types in `<stdfloat>`, such as `std::float16_t`, `std::float32_t`, `std::float64_t`, and `std::float128_t`, when provided by the implementation. These types are specified as extended floating-point types corresponding to ISO/IEC 60559 interchange formats.

Where such types are available, they provide a clear and portable way to refer to floating-point formats with known precision and representation properties. On these implementations, the conditions of this proposal naturally permit integer-to-floating list-initialization when the source integer type has no more value bits than the precision of the corresponding `std::floatNN_t` type.

This proposal does not special-case `std::floatNN_t`, but instead relies on semantic properties expressed via `std::numeric_limits`. As a result, the rule applies uniformly to `std::floatNN_t` and to standard floating-point types (`float`, `double`, `long double`) when they satisfy the same ISO/IEC 60559 and precision requirements. This avoids fragmenting the language rules while still allowing users who require predictable floating-point formats to prefer `<stdfloat>` types where available.

## Related work and prior art

The narrowing rules for list-initialization were originally introduced as part of C++11, with the goal of preventing silent loss of information during initialization. At that time, the rules were intentionally conservative, particularly for conversions from integer types to floating-point types, and included a limited exception for constant expressions whose values could be proven to round-trip exactly.

Since then, the C++ standard has evolved to expose more precise information about floating-point representations and semantics through library facilities such as `std::numeric_limits`, as well as through the introduction of extended and fixed-width floating-point types. Several proposals in this area have focused on making floating-point properties explicit and usable by programs, including work on extended floating-point types and on clarifying the requirements associated with ISO/IEC 60559 conformance.

However, the core language rules governing narrowing conversions in list-initialization have not been revisited to take advantage of this additional information. In particular, no prior proposal has addressed the case of integer-to-floating conversions that are provably lossless for all values of the source type based solely on the precision of the destination floating-point type.

This paper builds on the original intent of list-initialization narrowing by refining the rules to distinguish between potentially lossy conversions and those that are guaranteed to be exact for all values, using existing, standardized type traits.

## Why a library-only solution is insufficient

It is possible to express the predicate underlying this proposal as a constrained template function (for example, a hypothetical `lossless_cast<F>(i)` that is only enabled when the destination floating-point type can exactly represent all values of the source integer type). Such a utility can be useful as an illustration of the rule or as a workaround on older language versions.

However, a library-only approach cannot replace a core-language change in this area. List-initialization is the only mechanism in C++ that enforces non-narrowing conversions by default, and it is widely used as a semantic signal that an initialization is intended to be free of information loss. Requiring an explicit helper function at each call site would make this guarantee opt-in and verbose, weakening the safety-by-default property that list-initialization was designed to provide.

This proposal therefore targets the language rule itself, so that brace-initialization continues to express a uniform and immediate guarantee of lossless conversion without requiring additional boilerplate or user intervention.

