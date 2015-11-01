This document gathers my findings about gcc c++ name mangling.

It is to be considered as supplementaty materials to the [Itanium C++ ABI's mangling section](https://mentorembedded.github.io/cxx-abi/abi.html#mangling), especially it explores what accounts as a symbol to be substituted in the case of regular functions, templates and abbreviations.

# Mangling basics

### global variable declaration
As in `C` name mangling, it is just the name of the variable.

eg. `int bar;` is mangled as `bar`.

### `const` or `nested` variable declaration
```
 _Z <symbol>
```

eg. `int* const bar;` is mangled as `_ZL3bar`
- `_Z` preambule starts the mangled name, for OSX it would be `__Z`.
- `L` indicates the variable is `const`
  - Only applies to types with indirections (pointer / reference) since `const <value_type>` is mangled as `<value_type>`
- `3bar` variable name, length encoded.

eg. `namespace a { int bar; }` is mangled as `_ZN1a3barE`
- `_Z` preambule starts the mangled name, for OSX it would be `__Z`.
- `N1a3barE` encoded symbol. It is enclosed in `N`..`E` because the symbol is within a scope
  - `1a` namespace name, length encoded.
  - `3bar` variable name, length encoded.

**Note**: the `std` namespace is special. It is abbreviated `St` and remove the need for `N`..`E` enclosing.

eg. `namespace std { int bar; }` is mangled as `_ZSt3bar`

### Function declaration
```
 _Z <symbol> (<parameter>* | v )
```

eg. `void foo()` is mangled as `_Z3foov`
- `_Z` preambule is always here, it starts the mangled name, for OSX it would be `__Z`.
- `3foo` function name, length encoded.
- `v` no parameter is encoded as a single `void` parameter.

**Note**: the return type is not encoded here (although there are cases where it is encoded: function pointers and funtion template instances)

### Function template instance declaration
```
 _Z <symbol> I<template_parameter>+E <template_return_type> (<parameter>* | v )
```
eg.
```
template <typename A> void foo(A);
template <> void foo(int){}
```
is mangled `_Z3fooIiEvT_`:
- `_Z` preambule is always here, it starts the mangled name, for OSX it would be `__Z`.
- `3foo` function name, length encoded.
- `IiE` template parameter `int` is enclosed in `I`..`E`
- `v` the return type `void`.
- `T_` reference to the first template parameter. Second would be `T0_`, third `T1_`, fourth `T2_`, etc ...

# Substitutions basics

To save space a compression scheme is used where symbols that appears multiple times are then substituted by an item from the sequence : `S_`, `S0_`, `S1_`, `S2_`, etc ...

The main added value of this document is to identify what is to be counted as a substitution.

eg.
```
void foo(void*, void*)
```
`foo` would be encoded as `_Z3fooPvS_`. To be decomposed as 
- `_Z`
- `3foo`
- `Pv` stands for "pointer to void". Since it's not a basic type it's accounted as a symbol.
- `S_` refers to the first symbol encoded, here `Pv`.

**Note**: `foo` is a declaration, not a type and so it doesn't account as a substituable symbol.

## Parameters

- Basic types are encoded using a single letter. See [Itanium C++ ABI's types mangling](https://mentorembedded.github.io/cxx-abi/abi.html#mangling-type). Basic types are never substitutable.
  - eg. `void foo(int)` is encoded `_Z3fooi`

- No parameter is encoded as if a single `void` parameter were passed.
  - eg. `void foo()` is encoded `_Z3foov`

- Parameters are encoded one after the other.
  - eg. `void foo(char, int, short)` is encoded `_Z3foocis`. None of `char`, `int` or `short` are substitutable.

- Indirections (pointer/reference) and type qualifiers are prepended to the type. Each indirection / type qualifier accounts for a new symbol.
  - eg. `void foo(int)` is encoded `_Z3fooi`. No substitution.
  - eg. `void foo(const int)` is encoded `_Z3fooi`. No substitution.
  - eg. `void foo(const int*)` is encoded `_Z3fooPKi`
    - `Ki` becomes `S_`
    - `PKi` becomes `S0_`
  - eg. `void foo(const int&)` is encoded `_Z3fooRKi`
    - `Ki` becomes `S_`
    - `RKi` becomes `S0_`
  - eg. `void foo(const int* const*)` is encoded `_Z3fooPKPKi`
    - `Ki` becomes `S_`
    - `PKi` becomes `S0_`
    - `KPKi` becomes `S1_`
    - `PKPKi` becomes `S2_`
  - eg. `void foo(int*&)` is encoded `_Z3fooRPi`
    - `Pi` becomes `S_`
    - `RPi` becomes `S0_`

**Note**: `const int` alone is encoded as `int`, more generally constness of the type is not part of the signature (but constness of indirect types are).

- Functions are encoded between `F`..`E` and prepended with `P` for function pointer (`R` for function reference), return type of the function is encoded.
  - eg. `void foo(void(*)(int))` is encoded `_Z3fooPFviE`
    - `FviE` becomes `S_`
    - `PFviE` becomes `S0_`

## namespace

namespaces are considered as symbols.

```
namespace a {
	struct A{};
	void foo(A){}
}
```
`foo` would be encoded as `_ZN1a3fooENS_1AE`
- `a::foo` is encoded as `N1a3fooE`
  - It is enclosed by `N`..`E` (symbol is nested and not in `std`)
  - `a` is encoded `1a`
  - `foo` is encoded `3foo`
- `a::A` is encoded `NS_1AE`
  - enclosed in `N`..`E` (symbol is nested and not in `std`)
  - `a` is encoded `S_` (see substitution section)
  - `A` is encoded `1A`

**Note**: if namespace is `std` then it is abbreviated and nested symbol are no more enclosed in `N`..`E`
```
namespace std {
	struct A{};
	void foo(A){}
}
```
`foo` is encoded as `_ZSt3fooSt1A`
 - `std::foo` is encoded as `St3foo`
 - `std::A` is encoded as `St1A`

**Note**: `std` is not substituted since it is an abbreviation.

## Structs/Classes

Member functions are encoded as if they were in a namespace with the exception of const members which starts with a `K`.

```
class C {
	void foo() const {}
};
```
`foo` is encoded as `_ZNK1C3fooEv`
 - `C::foo` is encoded as `NK1C3fooE`
   - `K` is added at the beginning of the symbol because `foo` is `const`.
   - It is enclosed by `N`..`E` (symbol is nested)

## Template instance

Templated function instances have their template parameters mangled in a special way.
They are replaced in the parameters by an item from the sequence : `T_`, `T0_`, `T1_`, `T2_`, etc ...

```
template<typename T> T foo();
template<> int foo() {}
```
`foo` is encoded as `_Z3fooIiET_v`
 - The template parameters are encoded sequentially between `I`..`E`, here `int` encoded as `i`
 - Since `int` is the first parameter it is encoded as `T_` in the parameter list.
