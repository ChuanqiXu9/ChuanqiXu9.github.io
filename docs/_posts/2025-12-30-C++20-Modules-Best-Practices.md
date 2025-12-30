---
layout: post
title: "C++20 Modules 用户视角下的最佳实践"
date: 2025-12-30 +0800
categories: C++
toc: true
---

* TOC
{:toc}

在 2025 年的现在，关于 C++20 Modules 的文章、演讲和讨论比比皆是。只是他们大多是关于工具链的讨论或抱怨，在用户层面的探讨似乎较少。一方面工具链确实非常重要，是大范围使用 C++20 Modules 的基础。另一方面在语言特性角度 C++20 Modules 和 Coroutines、Concepts、反射以及 Contracts 等特性相比，说一句非常简单并不过分。但即便 C++20 Modules 在语言层面已经非常简单了，一些使用经验的分享应该依然是有价值的。

标题中的 “用户视角”，指的是不关心选择什么编译器、什么构建系统、编译器怎么实现 Modules、编译器如何与构建系统交互、不同编译器的不同行为等等事项，而是作为一个 C++ 库的维护者或一个 C++ 项目的最佳实践管理者的角色，在语言层面该如何使用 C++20 Modules。这里强调 “用户视角” 的原因是我想总结一些我觉得很有价值但还没被说的东西，并不是指现在工具链一切都 ready 了。（虽然我依然觉得 linux + clang 下 C++20 Modules 已经可用了）。

这里的讨论可以分为两部分：如何为现存使用头文件的项目提供 C++20 Modules Wraper（但依然使用头文件开发），以及怎么在 Modules Natively 地组织代码。Modules Natively 指像 Modules 从第一天就存在于 C++ 里一样地去编写代码。

我尝试让这篇博客的各个节保持独立，所以兴趣不同的读者可以跳过不感兴趣的内容。例如如果你想开始在一个全新的项目中使用 C++20 Modules，你可以只看 Modules Native 相关的节，这其实是最简单的部分。或者你不关心 ABI，你和你的用户也可以跳过 ABI 相关的部分。

# C++20 Modules 的好处

在介绍实践方式之前，我们先介绍下 C++20 Modules 的好处有哪些，为之后介绍不同的实践方式的原因做铺垫。C++20 Modules 的设计目的主要有：

- 更快的编译速度
- 避免 ODR Violation
- 控制 API 可见性
- 避免宏污染

其中更快的编译速度和避免 ODR Violation 两个目的都是通过 C++20 Modules 可以为每一个声明提供唯一一个归属的 TU 来达到的。

## 更快的编译速度 (和更小的代码体积)

之前有人认为 C++20 Modules 不过是标准化的 PCH 或者标准化的 Clang Header Modules。这都不对。PCH 或 Clang Header Modules 通过避免不同 TU 重复的预处理/语法分析以减少编译时间。

而 C++20 Modules 在此之上，还可以避免相同声明在编译器中后端的重复优化与编译。而对于很多项目而言，编译器中后端的优化和编译才是耗时的主要来源。

例如

```C++
// a.h
inline void func_a() {
    ...
}
```

这个写法会让每一个包含 `a.h` 且引用到了 `func_a()` 的 TU 都对 `func_a()` 做优化以及代码生成。

而使用 Modules 的写法

```C++
export module a;
export int func_a() {
    ...
}
```

无论有多少 TU 引用了 `func_a()`，这些 TU 被编译时都不会再对 `func_a()` 做重复的优化和代码生成。这是 C++20 Modules 相比于 PCH 或 Clang Header Modules 能提升更多编译速度的一个点。

比起全局函数，更常见的是 `in class inline function`，即：

```C++
class A {
public:
    void a() { ... }
};
```

C++20 标准规定，位于 Named Modules 中的 in class inline function 不再是 `implicitly inline`。即当 `A::a()` 位于 Named Modules 中时， `A::a()` 的定义只应该被放到 Named Modules 对应的 Object File 中，而不会被不同的 Consumer 重复优化/编译。

而除了这样显式的函数定义之外，诸如虚表和 debug info 等信息，都应该遵循相同的原则，即此类信息应该只在相关定义对应的 Named Modules 中生成，避免在各个 Consumer 中都生成一遍，即浪费时间还浪费空间。是的，我们在实践中发现，应用 C++20 Modules 不但可以减少编译时间，对于减少构建产物的体积也有显著帮助。

## 避免 ODR Violation

ODR（One Definition Rule）指的是一个程序中每个实体都应该只有一个相同的定义。当一个实体有多个不同的定义时，这个程序是就违反了 ODR，称为 ODR Violation，此时程序是 ill-formed。

实践中，若一个实体的多个定义是强符号，则会在链接时报错并提示 `multiple definition`。而如果一个实体的多个定义全是弱符号，则会在链接时挑选任意一个定义，实践上链接器一般会选择遇到的第一个定义。（忽略一个强符号多种弱符号的情况，这种情况一般是特意设计的）。两种情况相比，在链接时报错比在运行时报错要强很多，安全很多。

头文件机制因为其自身不是 TU 却要被许多 TU 共享的特征，天然地会将头文件内的几乎所有符号都设计为弱符号，为 ODR 安全埋下了很大的隐患。当一个大项目因为各种原因引入了同一个三方库的不同版本时，可能就陷入了 ODR Violation 的潜在危机中。

而 C++20 Modules 基于每一个实体都有唯一的 Owning TU 的原则，会为每一个实体都提供强符号，天然地可以避免这类 ODR Violation。

此外 C++20 Modules 还引入了独特的 Mangling 机制，为 Named Modules 中的每个实体添加和 Module 名强相关的后缀，可以避免不同库开发人员之间不经意的重名冲突。例如

```C++
export module M;
namespace NS {
  export int foo();
}
```

`NS::foo()` 的链接名在 Demangle 后为显示为 `NS::foo@M()`。进一步降低和其他 Module 中的 `foo()` 函数重名的概率。

至于 Module 的重名，C++20 Modules 要求每个 Module Unit 都生成一个 Module Initializer 用于初始化其内部状态（哪怕这个 Module 内部实际上不需要初始化任何东西），这个 Module Initializer 是一个强符号。从这个角度我们可以避免一个程序中出现重名的 Module Unit。

# Module Interface 后缀名

Clang 推荐 Module Interface 文件以 `.cppm` 作为后缀名。MSVC 推荐 `.ixx`。GCC 则没有特殊偏好。

大多数用户一般通过构建系统和编译器交互，而构建系统实际上清楚那些文件是 Module Interface 而那些不是。所以不少用户觉得这个问题并不重要，随意就行。

但我还是推荐大家使用 `.cppm` 或者 `.ccm` 这样的后缀作为 Module Interface 文件的后缀。原因一个是对于工具友好，另一个是可读性也更好。

对于工具，例如代码行统计这样的工具，通过后缀名进行统计可以给出更直观的结果。哪怕是对于 clangd 这样更复杂的工具，如果 clangd 可以假设所有 Module Interface 都以 `.cppm` 结尾，clangd 的处理速度就可以得到提升。例如 [Clion](https://www.jetbrains.com/help/clion/support-for-c-20-modules.html) 就假设了只有 `.cppm` 结尾的文件才是 Module Interface 文件。另外例如我现在维护 [Are We Modules Yet](https://arewemodulesyet.org/)，对我来说能假设所有 Module Interface 文件都以 `.cppm` 结尾会简单很多。

同时在代码可读性方面，`.cppm` 这样的特殊后缀对可读性也有帮助，例如：

```
.
├── network.cpp
└── network.cppm
```

当我们看到这样的文件结构时，我们可以很简单的认识到 `network.cppm` 声明了接口而 `network.cpp` 声明相关实现。此外对于 clangd 这样的工具，还有 `Switch Between Source/Header`（这个名字应该修改为 Impl/Interface）这样的功能可以让我们在 `network.cppm` 和 `network.cpp` 间快速挑战。

而对于 `.ixx`，我觉得他读上去像预处理后的文件，没有 `.cppm` 好看。

所以我推荐大家使用 `.cppm` 或者 `.ccm` 作为Module Interface 文件的后缀。

# 为基于头文件的项目提供 C++20 Modules Interface

（更简洁的版本位于 https://clang.llvm.org/docs/StandardCPlusPlusModules.html#transitioning-to-modules）

我们使用一个简单项目作为例子：

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

为了 ABI 兼容性，我们假设其会分发一个 libexample.so，其导出的符号为：

```
$nm -ACD libexample.so
libexample.so:                 w __cxa_finalize
libexample.so:                 w __gmon_start__
libexample.so:                 w _ITM_deregisterTMCloneTable
libexample.so:                 w _ITM_registerTMCloneTable
libexample.so:0000000000001130 W example::C::inline_get()
libexample.so:0000000000001110 T example::C::get()
```
（W 表示 Weak，指 `example::C::inline_get()` 为弱符号。T 表示 `example::C::get()` 为强符号）

对于 Header Only 库作者和只分发源码不分发二进制的库作者而言，这个 case 可能依然复杂了些，但只要理解了这个简单 case，相信为其他更简单的 case 封装 Module Wrappers 也不成问题。

## export using style

export using style 是为头文件提供 C++20 Module Interface 最简单的办法，包括 libc++、libstdc++ 和 MSSTL 使用的都是这种办法。目前看到的大部分支持 C++20 Modules 的库使用的也是这种办法。

这种办法类似：


```C++
// example.cppm
module;
#include "header.h"
export module example;
namespace example {
    export using example::C;
}
```

即在 Global Module Fragment 中插入此项目的所有 Headers ，然后在 Module Purview 中通过 `export using` 语句导出对外可见的声明。

这种方式最大的优点是简单以及对其他代码没有侵入性。

因为没有侵入性，我们在应用中发现其他三方库不支持 Modules 但我们又希望 import 这些三方库时，我们可以在自己的项目中为这些三方库添加 wrapper。

但这个方式的缺点主要是 Module Wrapper 与原先头文件中的实现位于不同的文件中，维护者可能在维护头文件的过程中新增/删除/修改了原先导出的接口但忘记更新 `example.cppm` 导致 break。这个问题可以靠脚本或测试来解决/缓解。例如：https://github.com/llvm/llvm-project/blob/main/libcxx/utils/generate_libcxx_cppm_in.py

### 对支持 C++20 Modules 的三方库使用 import

若你项目使用的三方库使用了 C++20 Modules，我们应该在 Module Interface 中应使用 import 引入该三方库而非 #include。这对于提升用户的编译速度有帮助。由于 Clang 编译器目前的实现限制，在存在 import 时避免使用 #include 可以带来更大的编译加速。

而我们可以将基础库看作最普遍最普遍的三方库，所有对于上述例子，我们可以改写 `header.h` 为：

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

然后将 `example.cppm` 改为：

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

这样可以让你的用户获得更大的编译性能提升。

### Export Using Style 的 ABI

需要注意，如果你的项目会分发二进制，你需要将 `example.cppm` 编译到你分发的二进制中，此时 libexample.so 中导出的符号应为：

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
（使用 `llvm-nm` 而非 `nm` 因为低版本 `nm` 不能 demangle C++20 Modules 相关 Mangling 规则）

与之前的版本相比，这里导出的符号多了 `initializer for module example`。

类似地，哪怕你的项目不分发二进制，但如果你的项目中存在源文件，你的构建脚本中应该将 `example.cppm` 这样的 Module Interface 和你的源文件编译同一个库文件中。因为这些文件逻辑上都属于你项目的头文件。

对于 Header Only 的库而言，如果你愿意将 `example.cppm` 这样的 Module Interface 添加到一个库（哪怕你不真的分发二进制）中对于用户来说是最方便的。

但如果你像以前一样单纯地只分发 `example.cppm` 的源码，那用户需要自己处理 `example.cppm` 对应的 Object File。如果此用户是终端用户，没有更下游的代码用户，那问题还算简单，只需要编译 `example.cppm`  到 Object File 然后链接到一起即可。而如果此用户依然是一个库用户，其存在更下游的代码用户，那可能最好的办法是不要将 `example.cppm` 编译到 Object File，将这个任务推迟到最后的二进制用户。这里的关键在于，如果 `example.cppm` 在一开始的库中没有被指派到某个库中，那这个源文件在二进制层面缺乏了真正的 Owner，我们只能希望最终的可执行文件的用户去处理它。

## export extern "C++" style

我们对所有头文件中的 #include 通过一个宏进行控制，例如：

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

然后我们通过以下形式在 example.cppm 对其封装 Modules Wrapper。

```C++
// example.cppm
module;
#include <cstdint>
// 在通用形式下，即在 global module fragment 中编写所有三方库头文件
export module example;
#define IN_MODULE_WRAPPER
extern "C++" {
    #include "header.h"
}
```

而如果我们的所有三方库都提供了 Modules 的话，我们可以进一步：

```C++
// example.cppm
export module example;
import std;
// 以及其他三方库的 Module, if any
#define IN_MODULE_WRAPPER
extern "C++" {
    #include "header.h"
}
```

这种形式下，在完成前期的 set up 后，后续在 example.cppm 中只需要在文件级别进行维护即可，不同声明的可见性通过 `EXPORT` 宏在声明处进行控制，相比于 export using style，可维护性大大提高了。

`example.cppm` 中的 `extern "C++"` 很关键，它用于保护库的 ABI 一致。将 `extern "C++"` 去掉则成为更近一步的 ABI Breaking Style。所以 export extern "C++" style 也可看作为后续更激进的改造做准备。

此外相比于 export using style，export extern "C++" style  在头文件存在 `inline` 全局函数和 inline 全局变量，特别是该变量存在动态初始化时，我们可以通过选择性 inline 令其应用更高的编译性能。

例如假设我们之前例子中的头文件为：

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

我们可以将其改造为：

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

（`example.cppm` 实现不变，这可以作为 export extern "C++" style 更具维护性的一个侧面例子）

在这种情况下，example.cppm 的 Consumer 不会重新编译 func。

注意这个改造其实更改了 ABI，让原先的弱符号变成了现在的强符号。这在 well defined 的项目，即不存在 ODR Violation 的项目中没有关系。但如果本来就有关于这几个符号的 ODR Violation，那这样的改造可能会改变现有的行为。如果我们担心这样的情况，我们可以修改 header.h 的实现为：

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

此时哪怕在 Modules 中，`example::func`，`example::init` 和 `example::var` 依然会是一个弱符号。这降低了触发本来 ODR Violation 的可能性，但需要再次强调，若项目在这几个符号上本来就处于 ODR Violation 状态，这样修改也只是降低了触发的可能性，依然有可能会改变程序现有的行为。

### 编译器忽略 Modue Units 中的所有 inline linkage

说到这里，还是想谈一下编译器中的处理。在我的实现过程中，有多人向我建议，编译器应该忽略 Module Units 的 inline 标识符，直接生成强符号或者对应的弱符号，而不是现在的 inline linkage。提到这样可以在 Modules 中避免 C++ 早期的一些设计问题（按我理解，现在很多 ODR 问题的原因之一便是当年头文件的设计）。但我还是觉得兼容性非常重要，应该尽可能把选择权留给用户。

### export extern "C++" style 的 ABI

在 ABI 上，如果不做上述的 inline 改写，export extern "C++" style 的 ABI 应与 export using style 的 ABI 完全一致。

## ABI Breaking Style

对以上例子的 export extern "C++" style 为例，在 example.cppm 中我们去掉 `extern "C++"` 就得到了 ABI Breaking Style 的 Module Interface：

```C++
// example.cppm
export module example;
import std;
// 以及其他三方库的 Module, if any
#define IN_MODULE_WRAPPER
#include "header.h"
```

去掉 `extern "C++"` 后，`header.h` 中的声明就隶属于 `example module`  了，和之前的 wrapper 有本质区别。此时 `example module` 中的 header.h 的声明不能复用之前 src.cpp 中的定义，我们需要为其提供新定义。

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

此时我们将 `src.cpp`, `src.module.cpp` 以及 `example.cppm`  对应的 Object File 链接为 `libexample.so`，然后查看其导出的符号：

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

我们可以看到 `libexample.so` 中同时存在 `example::C::inline_get()` 和 `example::C@example::inline_get()` 以及 `example::C::get()` 和 `example::C@example::get()` 两套 ABI。所以这也可以被称为 Dual ABI Mode。 这类似 [GCC5 libstdc++ 在 C++11 上那次有名的 ABI Break](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html) 。

虽然你的库本身依然兼容两套 ABI，但对于你的库用户来说，当其选择了使用基于 Modules 的 ABI 后，他的 ABI 也会 Break。例如你的用户代码中存在：

```C++
#include "header.h"

namespace user {
    void user_def(example::C& c) {
        
    }
}
```

然后你的用户的 libuser.so 暴露的符号可能是这样子的：

```
$llvm-nm -ACD libuser.so
libuser.so:                  w _ITM_deregisterTMCloneTable
libuser.so:                  w _ITM_registerTMCloneTable
libuser.so: 0000000000001100 T user::user_def(example::C&)
libuser.so:                  w __cxa_finalize@GLIBC_2.2.5
libuser.so:                  w __gmon_start__
```

而当你的用户选择使用你提供的，ABI Breaking Style 的 Module 后：

```C++
import example;

namespace user {
    void user_def(example::C& c) {
        
    }
}
```

其对应的 ABI 变成了

```
$llvm-nm -ACD libuser.so
libuser.so:                  w _ITM_deregisterTMCloneTable
libuser.so:                  w _ITM_registerTMCloneTable
libuser.so: 0000000000001100 T user::user_def(example::C@example&)
libuser.so:                  w __cxa_finalize@GLIBC_2.2.5
libuser.so:                  w __gmon_start__
```

可以看到用户代码中 `user_def` 生成的符号（demangle 后）从 `user::user_def(example::C&)` 变为了 `user::user_def(example::C@example&)`。这个现象和 [GCC5 libstdc++ 在 C++11 上的 ABI Break](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html)  也是一样的。不过这里的 ABI Break 是由你的用户自己控制的，所以不必内疚。

ABI Breaking Style 相比于 export extern "C++" style，除了 ABI 变化外，其生成的代码对于编译器而言也更有效率。例如之前提到的，in class inline function 在 named module 中不再是 implicitly inline 的。例如对于我们的例子：

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

现在 Dual ABI 中生成的符号是

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

可以看到 `C::inline_get` 在传统 ABI 中是一个弱符号，而在 Module 里就变成了强符号 `T example::C@example::inline_get()`。此外像上述的通过宏控制头文件中的 `inline` 实体的技巧在这里也有帮助。

ABI Breaking Style 的另一个好处是，现在你的用户在不知情的情况下想要混用你项目的 #include 和 import 更困难了。例如，如果你的用户无意间写出了这样的代码：

```C++
#include "header.h"
import example;

namespace user {
    void user_def(example::C& c) {
        
    }
}
```

编译器会自动提示：

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

这对帮助用户使用更佳的实践也有帮助。

## 选择什么方式来为你的头文件库提供 Module Interface

首先取决于你预计你的库之后是否会出现 fundamental 的 ABI breaking change，特别是你是否准备为你的库引入 Modules 相关的 ABI Breaking Change。如果是的话，在你能接受为所有实现文件都提供对应的 Modules 支持的情况下，我觉得 ABI Breaking Change 是最好的。当然你完全不关心 ABI，那 ABI Breaking Change 在你能接受对所有实现文件都提供对应 Modules 版本的情况下也是最好的。

其次，如果你关心 ABI，或者不想为你库的所有实现文件都提供的 Modules 版本，那你可以选择 export using style 或者 export extern "C++" style，取决于你想以什么方式控制符号的可见性。

最后无论何种方法，都建议在所有头文件中将所有的三方库 （包括标准库）的 `#include` 通过宏 #ifdef 控制，这样可以节约很多时间。

# 为你的 Module Wrapper 选择/提供一个 ABI Owner

前文提到，所有 Module Unit 都会在 Object File 中生成至少一个 initializer，此时我们就需要考虑该把这个包含 initializer 的 Object File 放于何处。对于本身就会分发二进制的库/项目而言，把 Module Unit 的 Object File 直接封装在二进制中是最简单的。如果不分发二进制，就需要在构建脚本中说明该如何构建这个 Object File 并放在什么库中。然后用户可以根据这个构建脚本进行构建和链接。

之前有一些关于 Header Only 项目转换到 C++20 Modules 后形式的讨论。还有人引入了 interface only 的概念。但我感觉这似乎都有点过于复杂了。C++20 Named Module Units 本质只是可 import 的 translation unit 而言。在二进制层面，C++20 Named Module Units 和一般的 Translation Unit 是没有区别的。即一个项目原先是 header only 的，引入了 C++20 Named Module 后，这个项目本身在二进制层面就应该是相应 Module Units 的 Owner。

例如 [async_simple](https://github.com/alibaba/async_simple) 一般情况下是一个 header only 的库。async_simple 也提供了 C++20 Modules Interface：
[async_simple.cppm](https://github.com/alibaba/async_simple/blob/main/async_simple/async_simple.cppm)。但用户希望使用 async_simple 提供的 modules 时，需要通过额外的 CMake 选项 ASYNC_SIMPLE_BUILD_MODULES，从头构建包含 C++20 Modules 的 libasync_simple：

```
message(STATUS "Fetching async_simple")
# 下载并集成 async_simple
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

(https://github.com/ChuanqiXu9/socks_server/blob/main/CMakeLists.txt)

即对于 header only 库，为了兼容性考虑，可以通过 CMake 选项 opt-in 的开启 modules 能力，保证此库在默认情况下依然是 header only 的库。但当用户需要 C++20 Modules 时，用户需要有能力从源码构建 C++20 Modules 所对应的版本的库。

# Modules Native Best Practice

一些背景知识可以在这里查看：https://clang.llvm.org/docs/StandardCPlusPlusModules.html#background-and-terminology

## 一个项目只声明一个 Module，需要有多个 TU 时使用 Module Partition Unit

例如这样的结构

```
.
├── common.h
├── network.h
└── util.h
```

当我们想将其改造为 Module 时，我们不应该将其改造为 `common module`，`network module` 以及 `util module`。这样引入了三个 module， 而且这样的命名方式还很容易重名。

我们应该只为我们的项目引入一个 module，暂且称其为 `example module`。然后将 `common.h`，`network.h` 和 `util.h` 改造后声明 `example:common`、`example:network` 和 `example:util` 三个 module unit。

这样的改造方式有两个好处：
1. 直接避免 forward declaration 问题。
2. 可以更好的控制符号可见性。

关于 forward declaration 问题，目前网上关于 Modules 的讨论中，抛开工具链相关的事情，在语言层面讨论最多的就是 forward declaration 问题了。例如 [C++ Modules and Circular References](https://stackoverflow.com/questions/73626743/c-modules-and-circular-class-reference)。但如果我们把当前这个项目/库看作是一个 module 的话就不会有这个问题了。这个在逻辑上也非常合理，你的库本身就是一个内聚的模块。

关于符号可见性。在 Modules 之前，分发二进制时，使用 `-fvisibility=hidden -fvisibility-inlines-hidden` 将所有符号都标记隐藏，只对希望对外可见的声明标记 `__attribute__((visibility("default")))`，例如：

```C++
void __attribute__((visibility("default"))) Exported()
{
    // ...
}
```

在引入 Modules 后，我们会将这两者联系在一起。例如 Clang 中的这个 [Issue](https://github.com/llvm/llvm-project/issues/79910)。即要求编译器提供一个选项，将 `export` 的符号不要默认为 hidden。但即使编译器没有这个选项，在用户视角下提供这样的宏也是很合理、自然的：

```C++
#define EXPORT export __attribute__((visibility("default")))

EXPORT void Exported() {}
```

而能这样做的前提则是我们将一个库作为一个模块。此时的 `export` 含义即是对库外可见。而当我们为每一个头文件都声明一个模块时，这些模块中 `export` 的含义变为了对其他文件可见。这样`export` 就失去了库层面的可见性的含义。

## 使用 Module Implementation Partition Unit 而不是 Module Implementation Unit 来实现接口

当我们在一个库中只使用一个模块时，我们很可能会引入大量的 Module Interface Units。而此时如果我们使用 Module Implementation Unit 来实现接口，例如

```C++
// network.cpp
module example; // will import example implicitly
// define network interfaces...
```

```C++
// common.cpp
module example; // will import example implicitly
// define common interfaces...
```

```C++
// util.cpp
module example; // will import example implicitly
// define util interfaces...
```

此时我们可以发现所有的 `*.cpp` 文件都依赖了 `example module` 的 Primary Interface。而一般来说，Module Primary Interface 会依赖这个 Module 的所有 Interfaces，例如：

```C++
export module example;
export import :network;
export import :common;
export import :util;
```

标准中也有相关说明：

> [module.unit]p3:
>    All module partitions of a module that are module interface units shall be directly or indirectly exported by the primary module interface unit. No diagnostic is required for a violation of these rules.

这里的问题在于，当我们修改某个 interface partition unit 时，例如 network.cppm，由于依赖传导，所有的 `*.cpp` 文件，在例子中包括 `common.cpp` 和 `util.cpp` 都会被重编译。 **这是不可接受的**。尤其当我们项目中 interfaces 和 implementations 文件的数量上升时，**这个问题在实践中是不可接受的**。

解决这个问题最简单的办法即是使用 Module Implementation Partition Unit 来实现接口。例如：

```C++
// network.cpp
module example:network;
// define network interfaces...
```

```C++
// common.cpp
module example:common;
// define common interfaces...
```

```C++
// util.cpp
module example:util;
// define util interfaces...
```

通过这种方式，起码 Modules 下的文件级依赖起码比起头文件版本，不算是 regression 了。

不过在实践中，对 CMake 的用户而言这个做法有个小问题，因为现在 CMake 要求所有 module implementation partition unit 都必须位于 CXX_MODULES FILES 中，这导致 CMake 会为所有 module implementation partition unit 生成 BMI。但这只是在浪费时间。例如我们上面这个例子，`network.cpp` `common.cpp` 和 `util.cpp` 在设计上不会任何其他 Unit import 他们，这一点是程序员自己可以保证的，这也是程序员的意图。但在 CMake 下，所有这样的 module implementation partition unit 都需要额外生成 BMI，有额外开销。这个问题在 [[C++20 Modules] We should allow implementation partition unit to not be in CXX_MODULES FILES](https://gitlab.kitware.com/cmake/cmake/-/issues/27048)。如果其他人发现了类似的情况下，很欢迎在该 issue 中 comment。

## 使用 module implementation partition unit 编写单元测试

这一条是上一条（使用  Module Implementation Partition Unit 作为实现文件）的自然延伸。

这里的问题是我们在单元测试时，常常需要测试项目内部的 API，但这些 API 不一定是对外 export 的。我们可以通过在 module implementation partition unit 中编写单元测试来避免这个 visibility 的问题。因为在一个 module 中所有 module level 的声明都是可见的。

另外一个小的点是，在 module implementation partition unit 中编写 main 函数时需要加上 `extern "C++"`。之前的 ISO 标准认为 main 函数不应该隶属于任何 named module，而禁止了这种用法，后来委员会修复了这个问题，只需要加上 `extern "C++"` 即可。不过这个对实践的影响不大，因为据我所知编译器之前的行为也是符合预期的，最新版本的编译器可能会对 Named Modules 中不在  `extern "C++"` 提示 warning。

## 使用 module implementation partition unit 改写不对外暴露的头文件

Module Implementation Partition Unit 令人困惑的一点在于，它也是可被 import 的。我在初次接触这块内容时，这让我很难理解 Module Implementation Partition Unit 和 Module Interface Partition Unit 的区别。

我一开始以为 Module Implementation Partition Unit 对应传统头文件中的 detail namespace。但现在看这不对，传统头文件中的 detail namespace 实际上就是 Module Interface Partition Unit 中没有被 export 的部分。

```
// detail.h
namespace detail {
    ...
}
```

```
// detail.cppm
export module example:detail;
// No export here
namespace detail {
    // ...
}
```

而 Module Implementation Partition Unit 除了上述作为实现文件的用处之外，在可被 import 时，其扮演的角色更类似于现在项目中不对外暴露的头文件。

以 Clang 为例 （Clang/LLVM 除编译器之外本身也是个库），

```
clang
├── AreaTeamMembers.txt
├── bindings
├── cmake
├── CMakeLists.txt
├── docs
├── examples
├── include
├── INSTALL.txt
├── lib
├── LICENSE.TXT
├── Maintainers.rst
├── NOTES.txt
├── README.md
├── runtime
├── test
├── tools
├── unittests
├── utils
└── www
```

其中 include 文件存放对外可见的头文件，而 lib 中以实现文件为主，但在 lib 中也存在头文件，例如

```
clang/lib/Serialization/
├── ASTCommon.cpp
├── ASTCommon.h
├── ASTReader.cpp
├── ASTReaderDecl.cpp
├── ASTReaderInternals.h
├── ASTReaderStmt.cpp
├── ASTWriter.cpp
├── ASTWriterDecl.cpp
├── ASTWriterStmt.cpp
├── CMakeLists.txt
├── GeneratePCH.cpp
├── GlobalModuleIndex.cpp
├── InMemoryModuleCache.cpp
├── ModuleCache.cpp
├── ModuleFile.cpp
├── ModuleFileExtension.cpp
├── ModuleManager.cpp
├── MultiOnDiskHashTable.h
├── ObjectFilePCHContainerReader.cpp
├── PCHContainerOperations.cpp
├── TemplateArgumentHasher.cpp
└── TemplateArgumentHasher.h
```

像这里的 `ASTReaderInternals.h`， `MultiOnDiskHashTable.h` 和 `TemplateArgumentHasher.h` 都是只会在 Serialization 内使用的头文件。这些头文件对于 Clang 的库用户而言都是不可见的。例如这样的文件，就适合改造为 Module Implementation Partition Unit。

或者换一个略显废话的角度来阐述使用 Module Implementation Partition Unit 的原则，任何不属于库的 interface 的文件，都使用 Module Implementation Partition Unit。
即库的 interface 使用 module interface units （包含 module primary interface units 和 module interface partition unit）和头文件（如果我们依然需要暴露宏），除此之外都使用 Module Implementation Partition Unit。

## 不要在 module interface 中 import module implementation partition unit

在 module interface 包含 module primary interface units 和 module interface partition unit）中不要 import module implementation partition unit。例如

```C++
module example:impl;
```

```C++
// interface.cppm
export module example:interface;
import :impl;
```

现在编译这个文件会提示：

```
interface.cppm:2:1: warning: importing an implementation partition unit in a module interface is not recommended. Names from example:impl may not be reachable
      [-Wimport-implementation-partition-unit-in-interface-unit]
    2 | import :impl;
      | ^
1 warning generated.
```

这个 practice 有两个原因：

1. 非直接 import 的  module implementation partition unit 不一定是 reachable 的。

[module.reach]p2 提到：

> All translation units that are necessarily reachable are reachable. Additional translation units on which the point within the program has an interface dependency may be considered reachable, but it is unspecified which are and under what circumstances.

necessarily reachable 的定义在 [module.reach]p1:

> A translation unit U is necessarily reachable from a point P if U is a module interface unit on which the translation unit containing P has an interface dependency, or the translation unit containing P imports U, in either case prior to P

即 necessarily reachable 指的是直接 import 的 TU 或者有 interface dependency 的 TU。interface dependency 的定义在 [module.import]p10:

> A translation unit has an interface dependency on a translation unit U if it contains a declaration that imports U or if it has an interface dependency on a translation unit that has an interface dependency on U.

即 interface dependency 指的是 import 的 module interface unit 以及递归 import 的 module interface unit。

总而言之，非直接 import 的  module implementation partition unit 不一定是 reachable 的，用原文说，这是 “may be considered reachable, but it is unspecified which are and under what circumstances.”。这一点在实践中很迷惑人。有多个 Clang 关于这个的 bug report，但最后都被以 invalid 的名义关掉了。所以为了进一步避免这种迷惑，我们建议用户不要在 module interface 中 import module implementation partition unit。

2. 建议不要在 module interface 中 import module implementation partition unit 的另一个原因是，可读性，这使得一个库的和 interface 和 implementation 的边界非常清楚。

不一定所有头文件或 importable module unit 都是 interface。库的 interface 的边界应该是由程序员精心设计的。但对于大规模项目而言 interface 边界，由于多人协同的关系，可能并没有被很好的维护起来，可能很多头文件都被无脑的放入 `include` 文件夹下最后渐渐变成了事实 interface 的一部分。而在引入 Modules 之后，我们有了新的工具进行辅助，程序员写代码时可以很清晰的看到当前 TU 是 module interface unit 或者是 module partition implementation unit，从而判断当前编写的文件是否属于项目的 interface。这对于可读性帮助是很大的。

## module implementation partition unit 小结

使用 module implementation partition unit 的原则是，任何不属于库的 interface 的文件，都使用 Module Implementation Partition Unit。如果这个 Module Implementation Partition Unit 可被其他 Module Unit import 的话，则使用 `.cppm` （`.ccm`） 后缀。否则，则将 Module Implementation Partition Unit 作为实现文件使用，以 `.cpp` 或 `.cc` 作为后缀。

不要在 module interface 中 import module implementation partition unit。

当你使用 module implementation partition unit 作为实现文件时，CMake 现在可能依然会编写 BMI，这会导致额外的开销。可以在 https://gitlab.kitware.com/cmake/cmake/-/issues/27048 中讨论。

## Module Implementation Unit

那 Module Implementation Unit 呢？在发现了 module implementation partition unit 作为实现文件的用法后，我不建议在任何大型 module 中使用 Module Implementation Unit。我现在感觉 Module Implementation Unit 只是一个不太甜的语法糖，可以帮助我们节约一行 `import` 的空间，但引入的依赖过于粗了。

## 对于实际上 TU Local 的 Entity，积极使用匿名空间或 static

例如

```
export module a;
struct A {};
```

在这个 TU 中，a 模块没有导出任何东西，我们可以期望编译器在编译时优化掉 `A` 的声明吗？编译器不可以这样做，因为虽然这里未导出的声明对外不可见，但其对于同一 Module 中的其他 module unit 是可见的。所以编译器依然需要中 BMI 中完整地记录 `struct A` 的所有信息。

在实践中，程序员可能会忘记这一点。哪怕 modules 引入了 export 关键字，对于只在当前 TU 可见的实体，我们还是应该积极的使用使用匿名空间或 static 标识符。这既可以减少最终生成的二进制符号，也可以减少 BMI 体积。

（关于 BMI，编译器理论上可以为 Primary Module Interface 生成两套 BMI，一套在 Module 内部用，一套在 Module 外部用。但这一方面需要编译器和构建系统协作，目前看编译器和构建系统的协作还是很困难。另一方面本文还是更关注于用户视角。所以不多展开。）

## 从 Modules Wrapper 到 Modules Native

虽然这比较遥远，但我们依然可以想象，未来某个时候，会有一个提供 Modules Wrapper 的库希望真正的在 Modules 中开发，而不是只提供 Wrapper。此时有几个选项：

1. 将头文件全部转换为 Module Interfaces。然后将原先的头文件版本的库放在单独的分支维护或放在单独文件夹内 freeze。
2. 依然保留头文件，但宣布部分新特性只能在 Modules 版本中使用。

其中第一个选项无论是手动修改还是依靠工具（例如 [clang-modules-converter](https://github.com/ChuanqiXu9/clang-modules-converter)）修改，在理解了该如何 Modules Natively 地编写代码后都比较直观。

关于第 2 个选项，我们可以先将原先的 modules wrapper 重命名到当前 module 一个 partition，例如：

```C++
export module example:header_interfaces;
import std;
// Other thirdparty modules, if any
#define IN_MODULE_WRAPPER
extern "C++" {
    #include "header.h"
}
```

之后我们在其他 partition 内正常编写 Modules 代码即可，需要使用到原先头文件中的接口时 `import :header_interfaces` 即可。最后在 primary module interface 中导出  `header_interfaces`：

```C++
export module example:
export import :header_interfaces;
export import :...; // other partitions
```

通过这种方式，我们可以做到保持原先头文件的前提下，使用 C++20 Modules 开发。

# Modules 改造过程中发生的 Runtime 问题

24年底到25年初，我曾经花了两个多月将一个 7M LoC 的大规模 C++ 项目修改为 Modules Native 项目。在改造开始前，我预计大部分时间可能会花在修复编译器 bug 上。但实际上真正花在编译器上的时间只有两个多星期，其余的大部分时间实际上是在查询改造后的运行时问题。这和我的预期不符。我之前认为 Modules 改造的大部分问题会发生在编译时，有问题应该就编不过，遍过了就不应该有问题。但实际还是不是这样的。在大规模 C++ 项目中 ODR Violation 是普遍存在的，很多时候他只是 just works。Modules 改造后就可能触发不少。这些问题的根因其实都很简单，但排查的过程很头疼。但换个角度想，Modules 改造的过程对于提升项目的稳定性，也是确实有帮助的，发现了很多技术债。

# 性能

多人提到过，现在 Named Modules 不导出非 inline 函数定义的方式对于性能优化有损害的。这个行为现在算标准委员会推荐的行为。主要动机是保证 ABI 的稳定性。

抛开 ABI 稳定性不谈，我们在实践中发现，这样的做法结合 thinLTO 其实并没有造成任何可观测的性能损失（我们在多个项目中反而发现了略微的性能提升），反而带来了更快的编译速度。如果编译器 trivial 地在优化时导入一切可导入的函数的话，那每个 TU 的优化复杂度会从 O(N) 增长到 O(N^2) （N 指每个 TU 中函数的平均数量），这反而不太能接受。

# 总结

整体上 C++20 Modules 相比于其他 C++ 的大特性而言，在语言特性角度还是很简单的。回顾这篇文章，其实大部分内容在介绍如何兼容头文件的情况下提供 C++20 Modules 以及 ABI 相关内容。如果不关心 ABI 或者非常激进，愿意直接编写 Modules Natively 项目的话，遇到的语言角度的问题应该还是比较少的。
