This document gathers my findings about gcc c++ name mangling.

It is to be considered as supplementaty materials to the [Itanium C++ ABI's mangling section](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#mangling), especially it explores what accounts as a symbol to be substituted in the case of regular functions, templates and abbreviations.

## Mangling basics

### Global variable declaration
As in `C` name mangling, it is just the name of the variable.

eg. `int bar;` is mangled as `bar`.

eg. `void(*baz)(int);` is mangled as `baz`.

### Const or nested variable declaration

eg. `int* const bar;` is mangled as `_ZL3bar`
- `_Z` preambule starts the mangled name.
- `L` indicates the variable is `const`
  - Only applies to types with indirections (pointer / reference) since `const <value_type>` is mangled as `<value_type>`
- `3bar` variable name, length encoded.

eg. `void(* const baz)(int) = nullptr;` is mangled as `_ZL3baz`
- `_Z` preambule starts the mangled name.
- `L` indicates the variable is `const`
- `3baz` variable name, length encoded.

eg. `namespace a { int bar; }` is mangled as `_ZN1a3barE`
- `_Z` preambule starts the mangled name.
- `N1a3barE` encoded symbol. It is enclosed in `N`..`E` because the symbol is within a scope
  - `1a` namespace name, length encoded.
  - `3bar` variable name, length encoded.

**Note**: the `std` namespace is special. It is abbreviated `St` and remove the need for `N`..`E` enclosing.

eg. `namespace std { int bar; }` is mangled as `_ZSt3bar`

### Function declaration
```
 _Z <declaration> (<parameter>+ | v )
```

`<parameter>` is defined as `([PR]K?)*(<basic_type>|<function>|<user_type>)`
with:
 - `P` for pointer
 - `R` for reference
 - `K` for const
 - `<basic_type>` for one of [C++ basic types](http://en.cppreference.com/w/cpp/language/types)
 - `<function>` are encoded between `F`..`E`, return type of the function is encoded before parameters
 - `<user_type>` are encoded between `N`..`E` when nested (TODO add definition) and describe the whole hierarchy of types.

eg. `void foo()` is mangled as `_Z3foov`
- `_Z` preambule is always here.
- `3foo` function name, length encoded.
- `v` no parameter is encoded as a single `void` parameter.

**Note**: the return type is not encoded here (although there are cases where it is encoded: function pointers and funtion template instances)

### Function template instance declaration
```
 _Z <declaration> I<template_parameter>+E <template_return_type> (<parameter>+ | v )
```
eg.
```
template <typename A> void foo(A);
template <> void foo(int) {}
```
is mangled `_Z3fooIiEvT_`:
- `_Z` preambule is always here, it starts the mangled name, for OSX it would be `__Z`.
- `3foo` function name, length encoded.
- `IiE` template parameter `int` is enclosed in `I`..`E`
- `v` the return type `void`.
- `T_` reference to the first template parameter. Second would be `T0_`, third `T1_`, fourth `T2_`, etc ...

### Declaration and user defined type encoding

Declaration and user defined types are encoded with their scope

```
namespace a {
    struct S {
        void foo();
        void const_foo() const;
    };
}
```
- `foo` is mangled `_ZN1a1S3fooEv`:
  - `N1a1S3fooE`: `a::S::foo`, foo is nested since it's within `S` and `a` so it is enclosed in `N`..`E`
    - `1a`
    - `1S`
    - `3foo`
  - `v`: because there is no parameters (encoded as a single `void` parameter).
- `const_foo` is mangled `_ZNK1a1S9const_fooEv`:
  - `NK1a1S9const_fooE`: `a::S::foo`, foo is nested since it's within `S` and `a` so it is enclosed in `N`..`E`
    - `K` because type `S` is `const` in the context of `foo`
    - `1a`
    - `1S`
    - `9const_foo`

### Substitutions

To save space a compression scheme is used where symbols that appears multiple times are then substituted by an item from the sequence : `S_`, `S0_`, `S1_`, `S2_`, etc ...

**The main added value of this document is to identify what is to be counted as a substitution.**

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

### Abbreviations

Some symbols are recognized as special and are never substituted
```
St = ::std::
Sa = ::std::allocator
Sb = ::std::basic_string
Ss = ::std::basic_string<char, ::std::char_traits<char>, ::std::allocator<char> >
Si = ::std::basic_istream<char, ::std::char_traits<char> >
So = ::std::basic_ostream<char, ::std::char_traits<char> >
Sd = ::std::basic_iostream<char, ::std::char_traits<char> >
```

### More on substitutions in function parameters

Function parameters are either basic types, user defined types or indirections to basic or user defined types.

- Basic types are encoded using a single letter. See [Itanium C++ ABI's types mangling](https://mentorembedded.github.io/cxx-abi/abi.html#mangling-type). Basic types are never substituable.
  - eg. `void foo(int)` is encoded `_Z3fooi`

- No parameter is seen by the compiler as a single `void` parameter.
  - eg. `void foo()` is seen as `void foo(void)` and encoded `_Z3foov`

- Parameters are encoded one after the other.
  - eg. `void foo(char, int, short)` is encoded `_Z3foocis`. None of `char`, `int` or `short` are substituable.

*****

- Indirections (pointer/reference) and type qualifiers are prepended to the type. Each indirection / type qualifier accounts for a new symbol.
  - eg. `void foo(int)` is encoded `_Z3fooi`.
    - No substitution.
  - eg. `void foo(const int)` is encoded `_Z3fooi`.
    - No substitution.
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

*****

- Functions are encoded between `F`..`E` and prepended with `P` for function pointer (`R` for function reference), return type of the function is encoded.
  - eg. `void foo(void(*)(int))` is encoded `_Z3fooPFviE`
    - `FviE` becomes `S_`
    - `PFviE` becomes `S0_`
	- eg. `void foo(void*(*)(void*),void*(*)(const void*),const void*(*)(void*));` is encoded `_Z3fooPFPvS_EPFS_PKvEPFS3_S_E` with the following substitutions
```
    _Z3fooPFPvS_EPFS_PKvEPFS3_S_E
S_          ^^                    : Pv          void*
S0_        ^^^^^^                 : FPvS_E      void*()(void*)
S1_       ^^^^^^^                 : PFPvS_E     void*(*)(void*)
S2_                   ^^          : Kv          const void
S3_                  ^^^          : PKv         const void*
S4_               ^^^^^^^         : FS_PKvE     void*()(const void*)
S5_              ^^^^^^^^         : PFS_PKvE    void*(*)(const void*)
S6_                       ^^^^^^^ : FS3_S_E     const void*()(void*)
S7_                      ^^^^^^^^ : PFS3_S_E    const void*(*)(void*)
```

### More on substitutions in scopes

#### namespace
```
namespace a {
	struct A{};
	void foo(A) {}
}
```
`foo` would be encoded as `_ZN1a3fooENS_1AE`
- `a::foo` is encoded as `N1a3fooE`
  - It is enclosed by `N`..`E` (symbol is nested and not in `std`)
  - `a` is encoded `1a`
  - `foo` is encoded `3foo`
- `a::A` is encoded `NS_1AE`
  - enclosed in `N`..`E` (symbol is nested and not in `std`)
  - `a` is encoded `S_`
  - `A` is encoded `1A`

*****

**Note**: if namespace is `std` then it is abbreviated and nested symbol are no more enclosed in `N`..`E`
```
namespace std {
	struct A{};
	void foo(A) {}
}
```
`foo` is encoded as `_ZSt3fooSt1A`
 - `std::foo` is encoded as `St3foo`
 - `std::A` is encoded as `St1A`

**Note**: `std` is not substituted since it is an abbreviation.

#### struct / classes

```
struct A {
    struct B{};
    void foo(B);
};
```
`void A::foo(A::B)` is encoded `_ZN1A3fooENS_1BE`
- `N1A3fooE`: declaration
  - `1A`: `A` is now as `S_`
  - `3foo`: name (not substituable)
- `NS_1BE`: single parameter of type `B`
  - `S_`: `A`
  - `1B`: name, `B` is now `S0_`

### More on substitutions in templates

Template instances account for one substitution.
```
template<typename T>
struct A {
    void foo(A);
};

template<> void A<int>::foo(A<int>) {}
```
`template<> void A<int>::foo(A<int>)` is encoded as `_ZN1AIiE3fooES0_`
- `N1AIiE3fooE`: declaration
  - `1AIiE`: is the template instantiation
    - `1A`: is now `S_`
    - `IiE`: one `int` template parameter
    - It accounts for a substitution, it is now `S0_`
  - `3foo`: name (not substituable)
- `S0_`: refers to `A<int>`

*****

Templated function parameters are substituted with `T[0-9]*_` on first use and then substituted again normally.
```
struct A {
    template<typename T>
    void foo(T, T);
};

template<> void A::foo<int>(int, int) {}
```
`template<> void A::foo<int>(int, int)` is encoded as `_ZN1A3fooIiEEvT_S1_`
- `N1A3fooIiEE`: declaration
  - `1A`: is now `S_`
  - `3foo`: name (not substituable)
  - `IiE`: one `int` template parameter, is now `T_`
  - `N1A3fooIiEE` the full instantiated template is now `S0_`
- `v`: template instance return type
- `T_S1_` function parameters
  - `T_`: refers to the first template parameter, is now `S1_`
  - `S1_`

*****

Templated function parameters are substituted only if the declaration is templated.

```
template<typename T>
struct A {
    void foo(T, T);
};

template<> void A<int>::foo(int, int) {}
```
`template<> void A<int>::foo(int, int)` is encoded as `_ZN1AIiE3fooEii`
- `N1AIiE3fooE`: declaration
  - `1A`: is now `S_`
  - `IiE`: one `int` template parameter (basic types are not substituable)
  - `1AIiE3`: is now `S0_`
  - `3foo`: name (not substituable)
- `ii`: first and second parameter refers to the first template parameter but is not substituted.

*****

```
struct B {};

template<typename T>
struct A {
    void foo(T, T);
};

template <> void A<B>::foo(B, B) {}
```
`template <> void A<B>::foo(B, B)` is encoded as `_ZN1AI1BE3fooES0_S0_`
- `N1AI1BE3fooE`: declaration
  - `1A`: is now `S_`
  - `I1BE`: one `B` template parameter is now `S0_`
  - `1AI1BE`: is now `S1_`
  - `3foo`: name (not substituable)
- `S0_S0_`: `S0_` refers to `B`.

*****

Function template parameters are substituted.

```
template<typename A, typename B> A foo(B, A, B);
template<> int foo(char, int, char) {}
template<> int foo(int, int, int) {}
```
`template<> int foo(char, int, char)` is encoded as `_Z3fooIicET_T0_S0_S1_`
- `3fooIicE`: is now `S_`
- `T_`: The return type of the function. `int` is now `S0_`
- `T0_`: `char` is now `S1_`

`template<> int foo(int, int, int)` is encoded as `_Z3fooIiiET_T0_S0_S1_`
- `3fooIiiE`: is now `S_`
- `T_`: The return type of the function. `int` is now `S0_`
- `T0_`: `int` is now `S1_`

