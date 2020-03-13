# A proposal for modular macros

*Document number: P0877R0*
</br>
*Audience: EWG*
</br>
*[Bruno Cardoso Lopes](mailto:blopes@apple.com)
</br>
*2018-02-06*

---

## Motivation & Background

In Albuquerque's paper [P0841R0](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/p0841r0.html), we discussed Apple's experience with modules on its software ecosystem. This paper continue the discussion but focus specifically on addressing the support for macros.

As previously presented, we currently use [Clang Modules](https://clang.llvm.org/docs/Modules.html) and rely on a couple of restrictions from our codebase:

  - Headers needs to expose content and be shared among 4 different C based languages: C, C++, Objective-C and Objective-C++.
  - The SDK is comprised of frameworks, whose interfaces are modular on top of headers (Header Modules).
  - Macros exists and communicate functionality throughout frameworks, which have header dependencies among each others.

<!--
EWG poll from Albuquerque is: 8 | 5 | 7 | 2 | 2
-->

## Examples of Macros in the SDK

Given that Apple library interfaces support different C-based languages, we rely on macros in order to correctly gather availability information, heterogeneous platform support and to reason on top of features. Because Apple's SDK is entirely modularized, these macros are available for consumption at any library interface level, being critical in Apple's chain of module imports.

For example, as one can see in `LinearAlgebra/base.h` shipped with the SDK in macOS 10.13, the header use macros to control availability information for a library:

```cpp
/*  Define abstractions for a number of attributes that we wish to be able to 
concisely attach to functions in the LinearAlgebra library.             */
#define LA_AVAILABILITY  __OSX_AVAILABLE_STARTING(__MAC_10_10,__IPHONE_8_0)
...
#define LA_FUNCTION      OS_EXPORT OS_NOTHROW
#define LA_CONST         OS_CONST
```

Note that `__MAC_10_10` is available through another module that provides macros from `Availability.h`. Apple's customers depend on macros with modules like these.

## Problems with the lack of Macros in the Modules TS

The [Modules TS](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/n4681.pdf) lacks support for macros. According to Section 3.2 in [p0142r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0142r0.pdf):

> ... because the preprocessor is largely independent of the core language, it is impossible for a tool to understand (even grammatically) source code in header files without knowing the set of macros and configurations that a source file including the header file will activate. It is regrettably far too easy and far too common to under-appreciate how much macros are (and have been) stifling development of semantics-aware programming tools and how much of drag they constitute for C++, compared to alternatives...

Using Modules as a way to break away from macros seems fair but we believe a transition path is necessary, since no support for macros blocks Apple's path to adopting C++ Modules. Additionally, concerns from others were already outlined in [P0273R1](http://open-std.org/JTC1/SC22/WG21/docs/papers/2016/p0273r1.pdf).

We propose support for macros with the intent of helping with migration. To achieve it, we suggest adding extra syntax that:

- Adds on top of existing [Modules TS](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/n4681.pdf) syntax.
- Can easily be later deprecated and removed, without polluting the reserved keyword namespace.

## Proposed macros syntax

To export macros defined in a module, we propose augmenting the *module-declaration* in a *module interface unit* with a submodule-like specification:

```c++
export module M; // declare module M
...
export module M.__macros; // M export all macros
```

In the code snippet above, all macros in `M`'s *module interface unit* are exported. To select macros to export, a macro identifier can be explicitly specified with `export module M.__macros.<MACRO_NAME>` as if accessing a reserved submodule:

```c++
export module M.__macros.INFINITY; // M exports macro INFINITY
...
export module M.__macros.HUGE_VAL; // M exports macro HUGE_VAL
```

On the module consumer side, no fine grained approach is available and one can only import the complete set of macros exported by module `M`:

```c++
import M.__macros; // import all macros exported in module M
```

Note that exporting all macros in module `M` doesn't by default make the macros visible to importers of `M`. Importers must explicitly use `import M._macros` to make macros defined in `M` visible. This enforces judicious use of macros in a modular world.

It's also important to notice that the dotted module names in the Modules TS don't mean any module-submodule relationship or filename hierarchy, which also has been the subject of other papers (see X & Y). However, we propose that the suffix `.__macros` has special meaning, regardless of the amount of dots in prior to the end of the module name.

The use of `import` for macros is only allowed *after* a *module-declaration* in the same file. Likewise `export` is only valid after a *module-import-declaration* in the same file.

Representing macros by wrapping them up under a specific submodule-like addressing provides the necessary syntactic sugar that allows for deprecation of the mechanism later on; there's no pollution of the top-level reserved keywords.

## Wording

## Acknowledgments
