This document gathers my findings about gcc c++ name mangling.

It is to be considered as supplementaty materials to the [Itanium C++ ABI's mangling section](https://mentorembedded.github.io/cxx-abi/abi.html#mangling), especially it explores what accounts as a symbol to be substituted in the case of regular functions, templates and abbreviations.

### Preambule
```
void foo()
```
`foo` is mangled as `_Z3foov`
- `_Z` preambule is always here, it starts the mangled name, for OSX it would be `__Z`.
- `3foo` function name, length encoded.
- `v` no parameter is encoded as a `void` parameter.

Note: the return type is not encoded here (although there are cases where it is encoded: function pointers and funtion template instances)

### Substitutions

To save space a compression scheme is used where symbols that appears multiple times are then substituted by an item from the sequence : `S_`, `S0_`, `S1_`, `S2_`, etc ...

eg.
```
void foo(void*, void*)
```
`foo` would be encoded as `_Z3fooPvS_`. To be decomposed as 
- `_Z`
- `3foo`
- `Pv` stands for "pointer to void". Since it's not a basic type it's accounted as a symbol.
- `S_` refers to the first symbol encoded, here `Pv`.

Note: `foo` is a declaration, not a type and so it doesn't account as a substituable symbol.

### namespace

namespaces are considered as symbols.

```
namespace a {
	struct A{};
	void foo(A){}
}
```
`foo` would be encoded as `_ZN1a3fooENS_1AE`
- `a::foo` is encoded as `N1a3fooE`
  - It is enclosed by `N`..`E` (symbol is nested, non const and namespace is not `std`)
  - `a` is encoded `1a`
  - `foo` is encoded `3foo`
- `a::A` is encoded `NS_1AE`
  - enclosed in `N`..`E` (symbol is nested, non const and not in `std`)
  - `a` is encoded `S_`
  - `A` is encoded `1A`

Note: if namespace is `std` then it is abbreviated and nested symbol are no more enclosed in `N`..`E`
```
namespace std {
	struct A{};
	void foo(A){}
}
```
`foo` is encoded as `_ZSt3fooSt1A`
 - `std::foo` is encoded as `St3foo`
 - `std::A` is encoded as `St1A`

Note: `std` is not substituted since it is an abbreviation.
