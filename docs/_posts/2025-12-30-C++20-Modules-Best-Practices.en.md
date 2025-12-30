---
layout: post
title: "C++20 Modules: Best Practices from a User's Perspective"
date: 2025-12-30 +0800
categories: C++
toc: true
---

*The post was written in [Chinese](./C++20-Modules-Best-Practices.html) and translated by LLM. Feel free to contact me if any phrasing seems unnatural.*

* TOC
{:toc}

Here in 2025, articles, talks, and discussions about C++20 Modules are not rare. However, most of them focus on toolchain discussions or complaints, with fewer exploring the topic from a user's perspective. On one hand, the toolchain is indeed crucial and fundamental to the widespread adoption of C++20 Modules. On the other hand, from a language feature standpoint, it's not an exaggeration to say that C++20 Modules are quite simple compared to features like Coroutines, Concepts, Reflection, and Contracts. But even if C++20 Modules are simple at the language level, sharing some practical experience should still be valuable.

The "user's perspective" in the title refers to not worrying about which compiler or build system to choose, how compilers implement modules, how they interact with build systems, or the different behaviors of various compilers. Instead, it's about how to use C++20 Modules at the language level, from the viewpoint of a C++ library maintainer or a manager of best practices for a C++ project. I emphasize the "user's perspective" because I want to summarize some things I find valuable but haven't been widely discussed, not to claim that the toolchain is all ready. (Although I still believe that C++20 Modules are usable on Linux with Clang).

This discussion can be divided into two parts: how to provide C++20 Modules wrappers for existing header-based projects (while still developing with header files), and how to organize code in a "Modules Native" way. "Modules Native" means writing code as if modules had existed in C++ from day one.

I have tried to make each section of this blog post self-contained, so readers with different interests can skip content that doesn't appeal to them. For example, if you want to start a new project using C++20 Modules, you can just read the sections on "Modules Native," which are actually the simplest part. Or, if you and your users don't care about ABI, you can skip the ABI-related sections.

ABI compatibility is a crucial aspect of C++. As a foundational new feature, C++20 Modules must account for ABI compatibility, which introduces significant complexity. However, many C++ users today don't concern themselves with ABI compatibility because they build everything from source—in such cases, much of this article can be skipped.

# Benefits of C++20 Modules

Before introducing the practices, let's first outline the benefits of C++20 Modules to provide context for the different approaches we'll discuss later. The main design goals of C++20 Modules are:

- Faster compilation speeds
- Avoiding ODR Violations
- Controlling API visibility
- Preventing macro pollution

The first two goals, faster compilation and avoiding ODR violations, are achieved by C++20 Modules' ability to provide a single, owning Translation Unit (TU) for each declaration.

## Faster Compilation Speeds (and Smaller Code Size)

Some have argued that C++20 Modules are nothing more than standardized Precompiled Headers (PCH) or standardized Clang Header Modules. This is incorrect. PCH or Clang Header Modules reduce compile times by avoiding repetitive preprocessing and parsing across different TUs.

C++20 Modules go a step further by also preventing repetitive optimization and compilation of the same declarations in the compiler backend. For many projects, backend optimization and code generation are the main sources of long compilation times.

For example:

```C++
// a.h
inline void func_a() {
    ...
}
```

This approach causes every TU that includes `a.h` and references `func_a()` to optimize and generate code for `func_a()`.

With modules:

```C++
export module a;
export int func_a() {
    ...
}
```

No matter how many TUs import and use `func_a()`, they will not repeatedly optimize and generate code for it during their own compilation. This is a key reason why C++20 Modules can improve compilation speeds more significantly than PCH or Clang Header Modules.

A more common case than global functions is `in-class inline functions`:

```C++
class A {
public:
    void a() { ... }
};
```

The C++20 standard specifies that member functions defined inside a class within a named module are no longer `implicitly inline`. This means that when `A::a()` is in a named module, its definition should only be placed in the object file corresponding to that named module, and it will not be repeatedly optimized/compiled by different consumers.

Besides explicit function definitions, other information like v-tables and debug info should follow the same principle. Such information should only be generated in the named module where the relevant definitions reside, avoiding redundant generation in every consumer, which wastes both time and space. Yes, in practice, we've found that using C++20 Modules not only reduces compilation time but also significantly helps in reducing the size of build artifacts.

## Avoiding ODR Violations

The ODR (One Definition Rule) states that every entity in a program should have exactly one definition. When an entity has multiple different definitions, the program violates the ODR and is ill-formed.

In practice, if an entity has multiple definitions with strong symbols, the linker will report a `multiple definition` error. If all definitions have weak symbols, the linker will arbitrarily pick one, typically the first one it encounters. (Ignoring the case of one strong symbol and multiple weak symbols, which is usually intentional). The former case, a linker error, is much safer and more robust than runtime undefined behavior.

The header file mechanism, due to its nature of being shared by many TUs without being a TU itself, naturally leads to most symbols within headers being designed as weak symbols, creating a significant risk of ODR violations. When a large project, for various reasons, includes different versions of the same third-party library, it can fall into the potential trap of ODR violations.

C++20 Modules, based on the principle that every entity has a unique owning TU, provide strong symbols for every entity, naturally preventing this type of ODR violation.

Furthermore, C++20 Modules introduce a unique name mangling mechanism that adds a suffix related to the module name to each entity within a named module. This helps prevent accidental name collisions between different library developers. For example:

```C++
export module M;
namespace NS {
  export int foo();
}
```

After demangling, the linkage name for `NS::foo()` will appear as `NS::foo@M()`, further reducing the probability of a name clash with a `foo()` function in another module.

As for module name collisions, C++20 Modules require each module unit to generate a strong-symbol module initializer to set up its internal state (even if there's nothing to initialize). This helps prevent a program from having module units with the same name.

# Module Interface File Extensions

Clang recommends using the `.cppm` suffix for module interface files. MSVC recommends `.ixx`. GCC has no specific preference.

Most users interact with the compiler through a build system, which typically knows which files are module interfaces and which are not. For this reason, many users feel this issue is unimportant and that any extension will do.

However, I still recommend using a suffix like `.cppm` or `.ccm` for module interface files. One reason is that it's more tool-friendly, and another is that it improves readability.

For tools like line-of-code counters, statistics can be more intuitive if they can be grouped by file extension. Even for more complex tools like `clangd`, if it can assume that all module interfaces end with `.cppm`, its processing speed can be improved. For instance, [CLion](https://www.jetbrains.com/help/clion/support-for-c-20-modules.html) assumes that only files ending in `.cppm` are module interface files. Also, as the maintainer of [Are We Modules Yet](https://arewemodulesyet.org/), it would be much simpler for me if I could assume all module interface files use the `.cppm` suffix.

In terms of code readability, a special suffix like `.cppm` is also helpful. For example:

```
.
├── network.cpp
└── network.cppm
```

When we see this file structure, we can easily understand that `network.cppm` declares the interface, and `network.cpp` provides the implementation. Additionally, tools like `clangd` offer features like "Switch Between Source/Header" (which should probably be renamed to "Impl/Interface") that allow us to quickly jump between `network.cppm` and `network.cpp`.

As for `.ixx`, I think it reads like a preprocessed file and doesn't look as nice as `.cppm`.

Therefore, I recommend using `.cppm` or `.ccm` as the suffix for module interface files.

# Providing a C++20 Modules Interface for Header-Based Projects

(A more concise version can be found at https://clang.llvm.org/docs/StandardCPlusPlusModules.html#transitioning-to-modules)

Let's use a simple project as an example:

```C++
// header.h
#pragma once

#include <cstdint>

namespace example {
class C {
public:
    std::size_t inline_get() { return 42; }
    std::size_t get();
};
}
```

```C++
// src.cpp
#include "header.h"

std::size_t example::C::get() {
    return 43 + inline_get();
}
```

For ABI compatibility, let's assume it distributes a `libexample.so` with the following exported symbols:

```
$nm -ACD libexample.so
libexample.so:                 w __cxa_finalize
libexample.so:                 w __gmon_start__
libexample.so:                 w _ITM_deregisterTMCloneTable
libexample.so:                 w _ITM_registerTMCloneTable
libexample.so:0000000000001130 W example::C::inline_get()
libexample.so:0000000000001110 T example::C::get()
```
(W indicates a weak symbol for `example::C::inline_get()`. T indicates a strong symbol for `example::C::get()`.)

For authors of header-only libraries and those who distribute source code without binaries, this case might still seem a bit complex. However, once you understand this simple case, wrapping module interfaces for simpler scenarios should be straightforward.

## The `export using` Style

The `export using` style is the simplest way to provide a C++20 Module interface for header files. It's the method used by libc++, libstdc++, and MSVC's STL. Most libraries that currently support C++20 Modules also use this approach.

The method looks like this:

```C++
// example.cppm
module;
#include "header.h"
export module example;
namespace example {
    export using example::C;
}
```

You include all the project's headers in the global module fragment and then use `export using` statements in the module purview to export the publicly visible declarations.

The biggest advantage of this approach is its simplicity and non-intrusiveness.

Because it's non-intrusive, if we find that a third-party library doesn't support modules but we want to import it, we can create a wrapper for it in our own project.

The main drawback is that the module wrapper and the original header's implementation are in different files. A maintainer might add, remove, or modify an exported interface in the header but forget to update `example.cppm`, causing a break. This problem can be mitigated or solved with scripts or tests. For example: https://github.com/llvm/llvm-project/blob/main/libcxx/utils/generate_libcxx_cppm_in.py

### Using `import` for Third-Party Libraries that Support C++20 Modules

If a third-party library used in your project supports C++20 Modules, you should use `import` instead of `#include` in your module interface. This helps improve your users' compilation speed. Due to current implementation limitations in the Clang compiler, avoiding `#include` when `import` is present can lead to greater compilation speedups.

We can treat the standard library as the most common third-party library. For the example above, we can modify `header.h` to:

```C++
// header.h
#pragma once

#ifdef USE_STD_MODULE_IN_HEADER
import std;
#else
#include <cstdint>
#endif

namespace example {
class C {
public:
    std::size_t inline_get() { return 42; }
    std::size_t get();
};
}
```

And change `example.cppm` to:

```C++
// example.cppm
module;
#define USE_STD_MODULE_IN_HEADER
#include "header.h"
export module example;
namespace example {
    export using example::C;
}
```

This can provide your users with a greater boost in compilation performance.

### ABI of the `export using` Style

Note that if your project distributes a binary, you need to compile `example.cppm` into that binary. The exported symbols in `libexample.so` would then be:

```
$llvm-nm -ACD libexample.so
libexample.so:                  w _ITM_deregisterTMCloneTable
libexample.so:                  w _ITM_registerTMCloneTable
libexample.so: 0000000000001050 T initializer for module example
libexample.so: 0000000000001140 W example::C::inline_get()
libexample.so: 0000000000001120 T example::C::get()
libexample.so:                  w __cxa_finalize@GLIBC_2.2.5
libexample.so:                  w __gmon_start__
```
(Using `llvm-nm` instead of `nm` because older versions of `nm` cannot demangle symbols related to C++20 Modules.)

Compared to the previous version, there is now an additional symbol: `initializer for module example`.

Similarly, even if your project doesn't distribute binaries but contains source files, your build script should compile module interfaces like `example.cppm` into the same library file as your sources. Logically, these files are all part of your project's interface.

For header-only libraries, the most convenient option for users is if you are willing to add module interfaces like `example.cppm` to a library (even if you don't actually distribute the binary).

However, if you simply distribute the source code for `example.cppm` as you did before, users will have to handle the corresponding object file themselves. If the user is an end-user with no downstream consumers, the problem is relatively simple: just compile `example.cppm` into an object file and link it. But if the user is a library author with their own downstream users, the best approach might be to not compile `example.cppm` into an object file and instead defer this task to the final binary user. The key here is that if `example.cppm` is not assigned to a library from the start, it lacks a true binary-level owner, and we can only hope that the final executable's user will handle it.

## The `export extern "C++"` Style

We can control all `#include` directives in our header files with a macro, like this:

```C++
// header.h
#pragma once

#ifndef IN_MODULE_WRAPPER 
#include <cstdint>
#endif

#ifdef IN_MODULE_WRAPPER
#define EXPORT export
#else
#define EXPORT
#endif

namespace example {
EXPORT class C {
public:
    std::size_t inline_get() { return 42; }
    std::size_t get();
};
}
```

Then, we wrap it in a module interface in `example.cppm` as follows:

```C++
// example.cppm
module;
#include <cstdint>
// In the general case, include all third-party headers in the global module fragment.
export module example;
#define IN_MODULE_WRAPPER
extern "C++" {
    #include "header.h"
}
```

If all our third-party libraries provide modules, we can go even further:

```C++
// example.cppm
export module example;
import std;
// and other third-party modules, if any
#define IN_MODULE_WRAPPER
extern "C++" {
    #include "header.h"
}
```

With this approach, after the initial setup, future maintenance in `example.cppm` is only needed at the file level. The visibility of different declarations is controlled by the `EXPORT` macro at the declaration site, which greatly improves maintainability compared to the `export using` style.

The `extern "C++"` in `example.cppm` is critical; it preserves the library's ABI. Removing `extern "C++"` turns this into the more aggressive ABI-breaking style. Thus, the `export extern "C++"` style can be seen as a preparatory step for more radical changes later.

Additionally, compared to the `export using` style, the `export extern "C++"` style allows for better compilation performance when headers contain `inline` global functions or `inline` global variables, especially those with dynamic initialization, by selectively applying `inline`.

For instance, suppose our header was:

```C++
// header.h
#pragma once

#include <cstdint>

namespace example {
inline int func() { return 43; }
inline int init() { return 43; }
inline int var = init();
}
```

We can refactor it to:

```C++
// header.h
#pragma once

#ifndef IN_MODULE_WRAPPER 
#include <cstdint>
#endif

#ifdef IN_MODULE_WRAPPER
#define EXPORT export
#else
#define EXPORT
#endif

#ifndef IN_MODULE_WRAPPER
#define INLINE inline
#else
#define INLINE
#endif

namespace example {
INLINE int func() { return 43; }
INLINE int init() { return 43; }
INLINE int var = init();
}
```

(The implementation of `example.cppm` remains unchanged, which serves as another example of the maintainability of the `export extern "C++"` style.)

In this case, consumers of `example.cppm` will not recompile `func`.

Note that this refactoring actually changes the ABI, turning what were weak symbols into strong symbols. This is fine in a well-defined project without ODR violations. However, if there were existing ODR violations related to these symbols, this change might alter the current behavior. If we are concerned about this, we can modify `header.h` to:

```C++
// header.h
#pragma once

#ifndef IN_MODULE_WRAPPER 
#include <cstdint>
#endif

#ifdef IN_MODULE_WRAPPER
#define EXPORT export
#else
#define EXPORT
#endif

#ifndef IN_MODULE_WRAPPER
#define INLINE inline
#else
#define INLINE __attribute__((weak))
#endif

namespace example {
INLINE int func() { return 43; }
INLINE int init() { return 43; }
INLINE int var = init();
}
```

Now, even within the module, `example::func`, `example::init`, and `example::var` will remain weak symbols. This reduces the likelihood of triggering a pre-existing ODR violation, but it's important to stress that if the project already has ODR violations on these symbols, this change only reduces the probability of triggering them and may still alter the program's behavior.

### Compilers Ignoring `inline` Linkage in Module Units

On this topic, I'd like to discuss compiler handling. During my implementation work, several people suggested that compilers should ignore the `inline` keyword in module units and directly generate strong or corresponding weak symbols, rather than the current inline linkage. They argued that this would help avoid some of C++'s early design problems in modules (as I understand it, many current ODR issues stem from the original design of header files). However, I believe that compatibility is very important, and the choice should be left to the user as much as possible.

### ABI of the `export extern "C++"` Style

In terms of ABI, without the `inline` modifications described above, the ABI of the `export extern "C++"` style should be completely identical to that of the `export using` style.

## ABI-Breaking Style

Taking the `export extern "C++"` style example, if we remove `extern "C++"` from `example.cppm`, we get the ABI-breaking style of module interface:

```C++
// example.cppm
export module example;
import std;
// and other third-party modules, if any
#define IN_MODULE_WRAPPER
#include "header.h"
```

After removing `extern "C++"`, the declarations in `header.h` now belong to the `example` module, which is fundamentally different from the previous wrapper approach. The declarations from `header.h` within the `example` module can no longer reuse the definitions from `src.cpp`; we must provide new definitions for them.

```C++
// src.cpp
#ifndef IN_MODULE_IMPL
#include "header.h"
#endif

std::size_t example::C::get() {
    return 43 + inline_get();
}
```

```C++
// src.module.cpp
module example;
#define IN_MODULE_IMPL
#include "src.cpp"
```

Now, if we link the object files for `src.cpp`, `src.module.cpp`, and `example.cppm` into `libexample.so`, the exported symbols will be:

```
$llvm-nm -ACD libexample.so
libexample.so:                  w _ITM_deregisterTMCloneTable
libexample.so:                  w _ITM_registerTMCloneTable
libexample.so: 0000000000001060 T initializer for module example
libexample.so: 0000000000001150 W example::C::inline_get()
libexample.so: 0000000000001130 T example::C::get()
libexample.so: 0000000000001180 T example::C@example::inline_get()
libexample.so: 0000000000001160 T example::C@example::get()
libexample.so:                  w __cxa_finalize@GLIBC_2.2.5
libexample.so:                  w __gmon_start__
```

We can see that `libexample.so` contains two sets of ABIs: `example::C::inline_get()` and `example::C@example::inline_get()`, as well as `example::C::get()` and `example::C@example::get()`. This can be called a "Dual ABI Mode" and is similar to the famous [ABI break in GCC 5's libstdc++ for C++11](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html).

Although your library remains compatible with both ABIs, for your library's users, choosing the module-based ABI will also break their ABI. For example, if your user's code contains:

```C++
#include "header.h"

namespace user {
    void user_def(example::C& c) {
        
    }
}
```

The exported symbols in your user's `libuser.so` might look like this:

```
$llvm-nm -ACD libuser.so
libuser.so:                  w _ITM_deregisterTMCloneTable
libuser.so:                  w _ITM_registerTMCloneTable
libuser.so: 0000000000001100 T user::user_def(example::C&)
libuser.so:                  w __cxa_finalize@GLIBC_2.2.5
libuser.so:                  w __gmon_start__
```

But when your user switches to your ABI-breaking style module:

```C++
import example;

namespace user {
    void user_def(example::C& c) {
        
    }
}
```

Their corresponding ABI changes to:

```
$llvm-nm -ACD libuser.so
libuser.so:                  w _ITM_deregisterTMCloneTable
libuser.so:                  w _ITM_registerTMCloneTable
libuser.so: 0000000000001100 T user::user_def(example::C@example&)
libuser.so:                  w __cxa_finalize@GLIBC_2.2.5
libuser.so:                  w __gmon_start__
```

As you can see, the symbol generated for `user_def` (after demangling) changes from `user::user_def(example::C&)` to `user::user_def(example::C@example&)`. This phenomenon is also identical to the ABI break in GCC 5's libstdc++ for C++11. However, since this ABI break is controlled by your user, you don't need to feel guilty about it.

Compared to the `export extern "C++"` style, the ABI-breaking style, in addition to the ABI change, also generates more efficient code for the compiler. For example, as mentioned earlier, in-class inline functions are no longer implicitly inline in named modules. In our example:

```C++
// header.h
#pragma once

#include <cstdint>

namespace example {
class C {
public:
    std::size_t inline_get() { return 42; }
    std::size_t get();
};
}
```

The symbols generated in the dual ABI mode are:
```
$llvm-nm -ACD libexample.so
...
libexample.so: 0000000000001150 W example::C::inline_get()
...
libexample.so: 0000000000001180 T example::C@example::inline_get()
...
```

You can see that `C::inline_get` is a weak symbol in the traditional ABI but becomes a strong symbol `T example::C@example::inline_get()` in the module ABI. The technique of controlling `inline` entities in headers with macros is also helpful here.

Another benefit of the ABI-breaking style is that it becomes harder for your users to unknowingly mix `#include` and `import` from your project. For example, if a user accidentally writes:

```C++
#include "header.h"
import example;

namespace user {
    void user_def(example::C& c) {
        
    }
}
```

The compiler will automatically issue an error:

```
$clang++ -std=c++23 -fPIC user.cpp -c -o user.o -fprebuilt-module-path=.
In file included from user.cpp:1:
./header.h:16:17: error: 'example::C' has different definitions in different modules; first difference is defined here found method 'inline_get' with body
   16 |     std::size_t inline_get() { return 42; }
      |     ~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~
./header.h:16:17: note: but in 'example' found method 'inline_get' with different body
   16 |     std::size_t inline_get() { return 42; }
      |     ~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~
1 error generated.
```

This also helps guide users toward better practices.

## Which Method to Choose for Providing a Module Interface for Your Header Library?

First, it depends on whether you anticipate any fundamental ABI-breaking changes for your library in the future, especially if you plan to introduce module-related ABI changes. If so, and if you are willing to provide corresponding module support for all implementation files, I believe the ABI-breaking style is the best choice. Of course, if you don't care about ABI at all, the ABI-breaking style is also the best option, provided you can support all implementation files with a module version.

Second, if you care about ABI or don't want to provide module versions for all your library's implementation files, you can choose between the `export using` style and the `export extern "C++"` style, depending on how you want to control symbol visibility.

Finally, regardless of the method, it's advisable to control all `#include` directives for third-party libraries (including the standard library) in all your headers with an `#ifdef` macro. This can save a lot of time.

# Choosing/Providing an ABI Owner for Your Module Wrapper

As mentioned earlier, every module unit generates at least one initializer in its object file, which means we need to decide where to place this object file. For libraries/projects that already distribute binaries, the simplest solution is to package the module unit's object file directly into the binary. If you don't distribute binaries, your build scripts must specify how to build this object file and which library to place it in. Users can then follow these build scripts for building and linking.

There has been some discussion about the form that header-only projects take after transitioning to C++20 Modules. Some have even introduced the concept of "interface-only" modules. However, I feel this might be overly complicated. A C++20 named module unit is essentially just an importable translation unit. At the binary level, there's no difference between a C++20 named module unit and a regular translation unit. This means if a project was originally header-only, after introducing C++20 named modules, the project itself should become the binary-level owner of the corresponding module units.

For example, [async_simple](https://github.com/alibaba/async_simple) is typically a header-only library. It also provides a C++20 module interface: [async_simple.cppm](https://github.com/alibaba/async_simple/blob/main/async_simple/async_simple.cppm). However, when users want to use the modules provided by `async_simple`, they need to enable the `ASYNC_SIMPLE_BUILD_MODULES` CMake option to build `libasync_simple` with C++20 modules from the source:

```cmake
message(STATUS "Fetching async_simple")
# Download and integrate async_simple
FetchContent_Declare(
  async_simple
  GIT_REPOSITORY https://github.com/alibaba/async_simple.git
  GIT_TAG f376f197e54d4921a7f0d8e40ad303e41018f7c2
)
set(ASYNC_SIMPLE_ENABLE_TESTS OFF CACHE INTERNAL "")
set(ASYNC_SIMPLE_DISABLE_AIO ON CACHE INTERNAL "")
set(ASYNC_SIMPLE_BUILD_DEMO_EXAMPLE OFF CACHE INTERNAL "")
set(ASYNC_SIMPLE_ENABLE_ASAN OFF CACHE INTERNAL "")
set(ASYNC_SIMPLE_BUILD_MODULES ON CACHE INTERNAL "")
FetchContent_MakeAvailable(async_simple)
```

(from https://github.com/ChuanqiXu9/socks_server/blob/main/CMakeLists.txt)

In other words, for a header-only library, you can use a CMake option to make modules opt-in for compatibility, ensuring the library remains header-only by default. But when a user needs C++20 Modules, they must be able to build the corresponding version of the library from the source.

# Modules Native Best Practices

Some background information can be found here: https://clang.llvm.org/docs/StandardCPlusPlusModules.html#background-and-terminology

## A Project Should Declare Only One Module; Use Module Partition Units for Multiple TUs

Consider a structure like this:

```
.
├── common.h
├── network.h
└── util.h
```

When converting this to modules, we should not create a `common` module, a `network` module, and a `util` module. This would introduce three modules, and such naming is prone to collisions.

Instead, we should introduce only one module for our project, let's call it `example`. Then, we should refactor `common.h`, `network.h`, and `util.h` into module partition units named `example:common`, `example:network`, and `example:util`.

This approach has two main benefits:
1. It directly avoids forward declaration issues.
2. It allows for better control over symbol visibility.

Regarding forward declaration issues, this is one of the most discussed language-level topics about modules online (aside from toolchain issues), as seen in questions like [C++ Modules and Circular References](https://stackoverflow.com/questions/73626743/c-modules-and-circular-class-reference). However, if we treat the entire project/library as a single module, this problem disappears. This is also logically sound, as your library is itself a cohesive module.

Regarding symbol visibility, before modules, when distributing binaries, it was common to use `-fvisibility=hidden -fvisibility-inlines-hidden` to hide all symbols by default and only mark declarations intended for export with `__attribute__((visibility("default")))`, for example:

```C++
void __attribute__((visibility("default"))) Exported()
{
    // ...
}
```

With the introduction of modules, we can connect these two concepts. For example, this Clang [Issue](https://github.com/llvm/llvm-project/issues/79910) requests a compiler option to not default `export`ed symbols to hidden. But even without such an option, providing a macro like this from a user's perspective is reasonable and natural:

```C++
#define EXPORT export __attribute__((visibility("default")))

EXPORT void Exported() {}
```

The premise for doing this is that we treat an entire library as a single module. In this context, `export` means "visible outside the library." If we were to declare a separate module for each header file, the meaning of `export` in those modules would become "visible to other files," and `export` would lose its meaning of library-level visibility.

## Use Module Implementation Partition Units, Not Module Implementation Units, to Implement Interfaces

When we use only one module in a library, we are likely to have a large number of module interface units. If we use module implementation units to implement their interfaces, for example:

```C++
// network.cpp
module example; // will implicitly import `example`
// define network interfaces...
```

```C++
// common.cpp
module example; // will implicitly import `example`
// define common interfaces...
```

```C++
// util.cpp
module example; // will implicitly import `example`
// define util interfaces...
```

We can see that all `*.cpp` files now depend on the primary interface of the `example` module. Typically, the primary module interface will depend on all of the module's interfaces, for example:

```C++
export module example;
export import :network;
export import :common;
export import :util;
```

The standard also mentions this:

> [module.unit]p3:
> All module partitions of a module that are module interface units shall be directly or indirectly exported by the primary module interface unit. No diagnostic is required for a violation of these rules.

The problem here is that when we modify an interface partition unit, such as `network.cppm`, all `*.cpp` files (including `common.cpp` and `util.cpp` in our example) will be recompiled due to the dependency chain. **This is unacceptable.** This problem becomes especially severe in practice as the number of interface and implementation files in a project grows.

The simplest way to solve this is to use module implementation partition units for the implementation files. For example:

```C++
// network.cpp
module example:network.impl;
// define network interfaces...
```

```C++
// common.cpp
module example:common.impl;
// define common interfaces...
```

```C++
// util.cpp
module example:util.impl;
// define util interfaces...
```

(We can't have duplicated partition name in the same module.)

This way, the file-level dependencies in the modules version are at least not a regression compared to the header file version.

However, in practice, there is a small issue for CMake users with this approach. Currently, CMake requires all module implementation partition units to be listed in `CXX_MODULES FILES`, which causes CMake to generate a BMI for each of them. This is a waste of time. In our example, `network.cpp`, `common.cpp`, and `util.cpp` are designed not to be imported by any other unit, a fact that the programmer can guarantee and intends. But under CMake, all such module implementation partition units still have a BMI generated, causing extra overhead. This issue is being discussed in [[C++20 Modules] We should allow implementation partition unit to not be in CXX_MODULES FILES](https://gitlab.kitware.com/cmake/cmake/-/issues/27048). If others have encountered a similar situation, you are welcome to comment on that issue.

## Use Module Implementation Partition Units to Write Unit Tests

This practice is a natural extension of the previous one (using module implementation partition units for implementation files).

The issue here is that during unit testing, we often need to test internal APIs of a project that are not necessarily `export`ed. We can solve this visibility problem by writing unit tests within a module implementation partition unit. This is because all module-level declarations are visible within the same module.

Another small point is that when writing a `main` function in a module implementation partition unit, you need to add `extern "C++"`. The ISO standard previously stated that `main` should not belong to any named module and forbade this usage. The committee later fixed this, and now you just need to add `extern "C++"`. This has little practical impact, as, to my knowledge, compilers' previous behavior was already as expected. The latest versions of compilers might issue a warning if `main` is in a named module but not inside an `extern "C++"` block.

## Use Module Implementation Partition Units to Replace Non-Exported Headers

One confusing aspect of module implementation partition units is that they are also importable. When I first encountered this, it was difficult for me to understand the difference between a module implementation partition unit and a module interface partition unit.

I initially thought that module implementation partition units corresponded to the `detail` namespace in traditional header files. But now I see that's incorrect. The `detail` namespace in traditional headers is actually the non-`export`ed part of a module interface partition unit.

```C++
// detail.h
namespace detail {
    ...
}
```

```C++
// detail.cppm
export module example:detail;
// No export here
namespace detail {
    // ...
}
```

A module implementation partition unit, aside from its use as an implementation file, plays a role more similar to a non-publicly exposed header file in a project when it is importable.

Let's take Clang as an example (Clang/LLVM is a library in itself, not just a compiler):

```
clang
├── ...
├── include
├── lib
└── ...
```

The `include` directory contains publicly visible header files, while `lib` primarily contains implementation files. However, `lib` also contains headers, for example:

```
clang/lib/Serialization/
├── ...
├── ASTReaderInternals.h
├── MultiOnDiskHashTable.h
├── TemplateArgumentHasher.h
└── ...
```

Headers like `ASTReaderInternals.h`, `MultiOnDiskHashTable.h`, and `TemplateArgumentHasher.h` here are only used within the `Serialization` library component. They are invisible to users of the Clang library. Such files are ideal candidates for being converted into module implementation partition units.

To put it another, perhaps more tautological, way: the principle for using module implementation partition units is that any file that is not part of the library's interface should be a module implementation partition unit. The library's interface should use module interface units (including the primary module interface unit and module interface partition units) and header files (if we still need to expose macros). Everything else should be a module implementation partition unit.

## Do Not `import` Module Implementation Partition Units in Module Interfaces

Do not `import` a module implementation partition unit within a module interface (which includes primary module interface units and module interface partition units). For example:

```C++
// impl.cppm
module example:impl;
```

```C++
// interface.cppm
export module example:interface;
import :impl;
```

Compiling this file will now produce a warning:

```
interface.cppm:2:1: warning: importing an implementation partition unit in a module interface is not recommended. Names from example:impl may not be reachable
      [-Wimport-implementation-partition-unit-in-interface-unit]
    2 | import :impl;
      | ^
1 warning generated.
```

There are two reasons for this practice:

1.  A module implementation partition unit that is not directly imported may not be "reachable."

    [module.reach]p2 states:
    > All translation units that are necessarily reachable are reachable. Additional translation units on which the point within the program has an interface dependency may be considered reachable, but it is unspecified which are and under what circumstances.

    "Necessarily reachable" is defined in [module.reach]p1:
    > A translation unit U is necessarily reachable from a point P if U is a module interface unit on which the translation unit containing P has an interface dependency, or the translation unit containing P imports U, in either case prior to P.

    In short, "necessarily reachable" refers to directly imported TUs or TUs on which there is an interface dependency. An "interface dependency" is defined in [module.import]p10:
    > A translation unit has an interface dependency on a translation unit U if it contains a declaration that imports U or if it has an interface dependency on a translation unit that has an interface dependency on U.

    This means an interface dependency refers to imported module interface units and recursively imported module interface units.

    In summary, a non-directly imported module implementation partition unit is not guaranteed to be reachable. In the standard's words, it "may be considered reachable, but it is unspecified which are and under what circumstances." This is very confusing in practice. There have been several Clang bug reports about this, but they were all closed as "invalid." To avoid this confusion, we recommend that users do not `import` module implementation partition units in module interfaces.

2.  The other reason to avoid importing implementation partitions in interfaces is readability. It makes the boundary between a library's interface and implementation very clear.

    Not all header files or importable module units are interfaces. The boundary of a library's interface should be carefully designed by the programmer. However, in large-scale projects, due to collaboration among many people, the interface boundary may not be well-maintained. Many headers might be mindlessly placed in an `include` folder and gradually become part of the de facto interface. With the introduction of modules, we have a new tool to help. When writing code, programmers can clearly see whether the current TU is a module interface unit or a module implementation partition unit, helping them determine if the file they are writing is part of the project's interface. This is a great help for readability.

## Module Implementation Partition Unit Summary

The principle for using module implementation partition units is: any file that is not part of the library's interface should be a module implementation partition unit. If this implementation partition unit can be imported by other module units, use a `.cppm` (or `.ccm`) suffix. Otherwise, use it as an implementation file with a `.cpp` or `.cc` suffix.

Do not `import` a module implementation partition unit in a module interface.

When you use a module implementation partition unit as an implementation file, CMake may still generate a BMI for it, leading to extra overhead. This can be discussed at https://gitlab.kitware.com/cmake/cmake/-/issues/27048.

## Module Implementation Unit

So, what about module implementation units? After discovering the use of module implementation partition units as implementation files, I do not recommend using module implementation units in any large module. I now feel that a module implementation unit is just a less-than-sweet piece of syntactic sugar. It saves one line of `import`, but the dependency it introduces is too coarse-grained.

## For Truly TU-Local Entities, Actively Use Anonymous Namespaces or `static`

For example:

```C++
export module a;
struct A {};
```

In this TU, module `a` exports nothing. Can we expect the compiler to optimize away the declaration of `A`? No, it cannot. Although this un-exported declaration is not visible externally, it is visible to other module units within the same module. Therefore, the compiler still needs to record the full information of `struct A` in the BMI.

In practice, programmers may forget this. Even with the `export` keyword introduced by modules, for entities that are only visible within the current TU, we should still actively use anonymous namespaces or the `static` specifier. This can reduce the number of symbols in the final binary and also reduce the BMI size.

(Regarding BMIs, a compiler could theoretically generate two sets of BMIs for a primary module interface: one for internal use within the module and one for external use. However, this requires collaboration between the compiler and the build system, which currently seems very difficult. Furthermore, this article focuses on the user's perspective, so I won't elaborate on this.)

## From Modules Wrapper to Modules Native

Although this may be a distant future, we can still imagine a time when a library providing a modules wrapper will want to develop natively in modules, rather than just providing a wrapper. At that point, there are a few options:

1.  Convert all header files to module interfaces. Then, either maintain the old header-based version of the library in a separate branch or freeze it in a separate folder.
2.  Keep the header files but announce that certain new features will only be available in the modules version.

The first option, whether done manually or with tools (like [clang-modules-converter](https://github.com/ChuanqiXu9/clang-modules-converter)), is relatively straightforward once you understand how to write code in a Modules Native way.

For the second option, we can first rename the original modules wrapper to a partition of the current module, for example:

```C++
export module example:header_interfaces;
import std;
// Other third-party modules, if any
#define IN_MODULE_WRAPPER
extern "C++" {
    #include "header.h"
}
```

Afterward, we can write normal modules code in other partitions. When we need to use an interface from the old headers, we can just `import :header_interfaces`. Finally, we export `header_interfaces` from the primary module interface:

```C++
export module example;
export import :header_interfaces;
export import :...; // other partitions
```

This way, we can develop with C++20 Modules while still maintaining the original header files.

# Runtime Issues Encountered During Module Conversion

From late 2024 to early 2025, I spent over two months converting a large-scale C++ project of 7 million lines of code (7M LoC) to be Modules Native. Before starting, I expected to spend most of my time fixing compiler bugs. In reality, I only spent about two weeks on compiler issues; the majority of my time was actually spent investigating runtime problems after the conversion. This was not what I expected. I had thought that most problems with a modules conversion would occur at compile time—if it compiled, it should work. But that wasn't the case. In large-scale C++ projects, ODR violations are common, and often, things "just work." A modules conversion can trigger many of these latent issues. The root causes of these problems are actually simple, but the process of debugging them is a headache. Looking at it from another angle, however, the process of converting to modules genuinely helps improve the stability of a project, as it uncovers a lot of technical debt.

# Performance

Many have mentioned that the current practice of named modules not exporting non-`inline` function definitions is detrimental to performance optimization. This behavior is now recommended by the C++ standards committee, primarily to ensure ABI stability.

Setting aside ABI stability, we have found in practice that this approach, combined with ThinLTO, does not cause any observable performance degradation (in fact, we observed slight performance improvements in several projects). Instead, it leads to faster compilation speeds. If a compiler were to trivially import every importable function during optimization, the optimization complexity for each TU would increase from O(N) to O(N^2) (where N is the average number of functions per TU), which would be unacceptable.

# Conclusion

Overall, C++20 Modules are quite simple from a language feature perspective compared to other major C++ features. Looking back at this article, most of the content is about how to provide C++20 Modules support while maintaining compatibility with header files, and about ABI-related topics. If you don't care about ABI or are aggressive enough to write a Modules Native project from scratch, you should encounter relatively few language-level problems.

Below is my comment on Reddit, which I think serves as a fitting summary here:

(1) In both design and practice, modules can deliver significantly greater compile-time speedups compared to precompiled headers (PCH). Additionally, named modules can reduce the size of build artifacts—something PCH simply cannot do.

(2) The encapsulation provided by modules enables finer-grained dependency and recompilation analysis. Some of this work has already been open-sourced—for example: https://clang.llvm.org/docs/StandardCPlusPlusModules.html#experimental-non-cascading-changes. We have even more such efforts internally, which we plan to gradually open-source in the future.

(3) Moreover, the ability of named modules to detect ODR (One Definition Rule) violations genuinely impressed me. I was already aware of, understood, and had personally encountered various ODR violation issues before. However, during our migration, discovering so many previously hidden ODR violations was still quite shocking.

(4) Regarding complexity, I suspect the perception that C++20 modules are overly complicated stems largely from their long implementation journey and the abundance of (sometimes conflicting) articles written about them. But if we set aside build system integration for a moment, the language-level features of C++20 modules are actually quite straightforward—even simpler than header files once you get used to them. I believe I’m well-positioned to say this: we completed our native C++20 modules migration back in early 2025. Most developers adapted quickly; after an initial ramp-up period where they occasionally came to me with questions, I now rarely receive any module-related inquiries. In my experience, for a large project, you only need one or two people to handle the build system and framework setup—the rest of the team can simply follow established best practices. From this standpoint, C++20 modules haven’t introduced meaningful additional burden to C++ development.

(5) As for toolchains, it’s true that C++20 modules have had a massive impact, and their implementation and practical adoption have indeed taken a very long time. I fully understand why users—especially those who’ve been closely following C++20 modules’ progress—might feel fatigued. But while we may not be moving fast, we’ve never stopped moving forward. One motivation behind writing this blog post was precisely to create content with longer-lasting relevance. I noticed that many articles commenting on the state of toolchains from just a year ago are already outdated. That’s why I wrote this piece—from a user’s perspective on modules.

As for migration cost, of course, opinions will vary. But based on my experience, it’s a one-shot effort: once you go through it once, you’re essentially done.