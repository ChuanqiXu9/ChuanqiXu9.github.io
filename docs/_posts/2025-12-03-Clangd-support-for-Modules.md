---
layout: post
title: "Clangd 中的 C++20 Modules 支持"
date: 2025-12-3 +0800
categories: C++
toc: true
---

* TOC
{:toc}

这篇博客的目的是鼓励更多的人来贡献 Clangd 中的 C++20 Modules 支持。
写这篇博客的动机是，我发现，抱怨 C++20 Modules 的智能提示的人数，与 Clangd 中 C++20 Modules 支持的贡献者人数，以及 Clangd 中 C++20 Modules 支持的复杂度，这三者并不成比例。所以我推测大多数人可能没有理解 Clangd 中 C++20 Modules 的支持复杂度实际上非常简单。所以我写了这篇博客，希望和更多的人说明，与编译器中 Modules 支持的复杂度不同，在 clangd（以及类似的工具、例如 clang-tidy 或其他 language server）支持 C++20 Modules 的复杂度并不高。

此外也在这里推荐下 [Clice](https://github.com/clice-io/clice)，虽然目前还没有支持 Modules，但看上去前景不错。

# C++20 Modules 前置背景

在 C++20 Modules 之前，所有的编译单元都可以被直接编译。几乎所有 C++ 工具（构建系统、静态分析工具等等）都依赖这一前提。然而在引入 C++20 Modules 之后，这个前提不正确了。这是 C++20 Modules 会对工具链造成巨大影响的原因，也是目前 C++20 Modules 应用进度相对较慢的原因之一。

对于一个最简的 C++20 Modules 例子：

```C++
// a.cppm
export module a;
```

```C++
// b.cppm
export module b;
import a;
```

为了编译 `b.cppm`，我们现在 “需要” 使用两条编译命令：

```
$ clang++ -std=c++20 a.cppm --precompile -o a.pcm
$ clang++ -std=c++20 b.cppm -fmodule-file=a=a.pcm -fsyntax-only
```

（对于这种极简单的 case，编译器可能有能力通过一条语句进行处理，但那不具备可扩展性，除非我们把构建系统塞到编译器里，但这里不讨论那个方向了）

这里的 `a.pcm` 叫做 BMI *，是编译器对 `a.cppm` 需要对外暴露信息序列化后的结果。所有直接或间接 `import a;` 的编译单元，都需要 `a.cppm` 的 BMI 才可以被编译。

* BMI 指 Binary Module Interface 或 Built Module Interface，其全称有争议，但 BMI 这个简写的使用没有争议。

# 工具链支持 Modules 所需步骤

除了编译器之外的工具链，为了支持 Modules 需要做到：
- 对于任意含有 import 的编译单元，我们需要计算出其直接和间接 import 的 module unit，这里我们称这些 module units 为 `Required Module Units`。并根据此信息计算出编译单元的有向无环图依赖图。
- 获取这些 `Required Module Units` 对应的 BMI，当不存在这样的 BMI 时，工具需要根据有向无环依赖图调用编译器以生成所需的 BMI。
- 对于一个编译单元，当其存在 `import` 时，需根据以上两条信息，拼接出编译此编译单元所需的编译命令。

特别地，当我们构建了一个项目，并生成了 [Compilation Database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) 之后，对于其中的每一个编译单元，我们实际上既有编译此单元所需的编译命令，又拥有编译此单元所需的全部 BMI，所以在这种情况下很多工具其实是可用的。这也是为什么我们可以在不显式支持 clang-tidy 的情况下，集成 clang-tidy 到使用 Modules 的项目中的原因。此外，因为这个原因，我们建议构建系统出一个特别的模式，“build BMIs only”。这样可以让用户在只需要 clang-tidy 这样的场景里只构建项目里所有的 BMI 而不需要编译整个项目。

# Clangd 支持 Modules 所需步骤

与上一条类似，当构建一个项目并生成 [Compilation Database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) 后，将该 Compilation Database 导入 Clangd，我们可能会发现此时的 Clangd 突然能在使用 Modules 的项目中工作了。

但这并不理想，因为：
- 这要求 Clangd 的版本和构建时使用的 Clang 必须是完全一致的。这暗示了我们不能使用 GCC 或 MSVC 作为构建工具。这和 Clangd 的设计理念不符。这一点对所有工具都适用。
- Clangd 作为 Language Server 需要即时反馈。但 clangd 不显式支持 Modules 的话，当我们修改了某个 Module Interface 后，对应的 BMI 是不会自动呢更新的，我们也无法看到对应的修改的，除非我们重新构建一遍我们的项目。这样的用户体验不好。

这里的第二点是 clangd 显式支持 C++20 Modules 更迫切的主要原因。其他工具诸如 clang-tidy 是可以接受先构建后分析的 workflow 的，但对于 clangd 而言，不能即时看到更改的用户体验差异会非常大。

# Clangd 是如何工作的

High Level 地来说，Clangd 的核心逻辑位于 `clang-tools-extra/clangd/TUScheduler.cpp` 中，其中有详细的注释，大家感兴趣的可以阅读。其中 Clangd 会为我们打开的每一个 Tab 页面创建一个 Worker，这个 Worker 位于一个单独的线程中。每个 Worker 会调用 ClangAPI 对其打开的页面进行编译，获取 AST。当后续事件来临时（例如鼠标悬浮，要求跳转等等），Worker 会先检查之前的 AST 是否依然可用，如果已经过时了 Worker 会重新进行编译，然后根据 AST 响应各种事件的请求。

从这个角度来说，Clangd 支持 C++20 Modules 所做的工作主要集中在编译部分和检测编译结果是否过时两部分。

对于利用 AST 的部分，在不要求新功能，只要求 clangd 可以继续工作的情况下暂时可以不用关心。新功能的例子之一即编写 import A 时 Clangd 可以自动提示后面可能的 Module 名，这部分工作还没有支持。但从个人体验上看，这方面目前的 AI 做的已经很不错了，所以我个人对这部分没什么需求。

## 编译

正如前文所述，在引入 C++20 Modules 之前，每个编译单元都是独立的，只需要对应的编译命令（当然，以及对应的文件）就可以进行编译。Clangd 通过 Compilation Database 获取每个编译单元的编译命令，然后编译当前页面以获取 AST。这部分逻辑也主要位于 `TUScheduler.cpp` 中。

特别地，为了加速编译，Clangd Worker 为其打开的 Tab 页面使用 PCH （预编译头技术）将头文件提前编译并将编译结果保存起来，称为 Preamble。这样做的好处是当我们打开一个 Tab 页面，并修改这个页面的内容时，（如果我们没有修改头文件区域），我们重新编译当前 Tab 的内容时可以避免重新编译头文件区域的内容，这是一个合理且有效的优化手段，大大加快了 Clangd 的处理速度。当然这样的做法也有坏处，因为每个 Preamble 属于不同的 AST Worker，所以我们打开新的 Tab 页面时，Clangd 依旧会花费额外的时间编译该 Tab 页面的 Preamble，这可能是会浪费时间。此外由于每个 Tab 页面都拥有自己的 Preamble，可以预见当我们打开大量 Tab 页面时 Clangd 会花费更多内存。

从这个角度，我们可以合理地想象，对于充分使用 C++20 Modules 的项目，在完成 warmup 之后，Clangd 消耗的内存将大大减少。

## 检测编译结果是否过时

Preamble 设计的另一个意义是，它将一个 Tab 页面的内容划分为了两部分，一个是当前页面的依赖，另一个是这个页面的内容本身。此时检测编译结果是否过时则自然变成了检测当前页面内容是否过时以及依赖的内容是否过时两部分。检测当前页面是否过时很简单。我们只需要检测依赖的内容，及 Preamble 是否过时即可。

这部分在 [isPreambleCompatible](https://github.com/llvm/llvm-project/blob/822fc449985553c609e44915374f935672c0db50/clang-tools-extra/clangd/Preamble.cpp#L745-L757) 中实现。

# Clangd 中 C++20 Modules 的支持

## High Level 现状

目前 Clangd 中已有对 C++20 Modules 的基本支持，只需要传入 `--experimental-modules-support` 选项即可开启。我们本地用着功能算符合预期，但在社区里还是有很多抱怨，很多 Issue 也不好复现。这也是写这篇blog的主要动机，大家可以在自己的环境里简单检查下，发现 clangd 的 bug 或者顺手开发的新 feature 也可以贡献到社区。

如前文所述，使用 C++20 Modules 需要传入 Compilation Database。Compilation Database 中需要包含当前项目中所有 TU 的编译选项。社区常见的几个错误报告要么是没有传入 Compilation Database，要么是 Compilation Database 没有包含必须的编译单元及其编译命令。例如如果你的项目使用了 `import std;` 那么我们需要在传入的 Compilation Database 中看到提供 `std module` 的编译单元。

## 测试与调试

由于 Clangd 可以通过 Compilation Database 复用构建系统构建的 BMI 文件，大家测试 Clangd 的 C++20 Modules 能力时可以在生成 Compilation Database 后删除所有 BMI，避免出现 false negative 现象的出现。

当大家目前使用 clangd 的 C++20 Modules feature 时，如果有问题，可以先使用 `--log=verbose` 观察 Clangd 的输出，可能会有发现。

若发现 Clangd 编译 Module Units 报错时，需要检查你的项目是否可被同版本的 Clang 编译器构建。如果你的项目可以被同版本的 Clang 编译器构建，但是 Clangd 编译 Module Units 时依然报错甚至 Crash，你可以尝试使用 `--debug-modules-builder` 选项。不开启 `--debug-modules-builder` 选项时，Clangd 会在退出时删除所有其构建的 BMI 文件，这不利于我们复现 Clangd 的报错或奔溃。在开启 `--debug-modules-builder` 后，Clangd 不会删除其构建的 BMI 文件。结合 `--log=verbose` 中显示的 clangd 构建 Compilation Database 所使用的编译命令，我们就可以在终端中复现 Clangd 的报错了。

另外根据推测，在 `#include` 与 `import` 大量混用的项目中，Clangd 编译结果和 Clang 编译结果不一致的原因可能是 Clangd 自带的 PCH 和 C++20 Modules 之间的兼容性问题。PCH、Clang Header Modules 和 C++20 Modules 之间理论上是兼容的，但缺乏足够的测试和实践，在我早期开发 Clangd 中 C++20 Modules 支持的时候，遇到的不少问题就是 Clangd 自带的 PCH 和 C++20 Modules 二者交互时产生的问题。我目前在 Clangd 中没有看到可以关闭 Clangd PCH 功能的选项，之后可能可以增加一个（避免在使用 `import` 的TU中构建 PCH）。

## 支持思路

如前文所述，在 clangd 中支持 C++20 Modules，需要打开任意一个 Tab 页面时，找到其直接和间接 import 的所有 module units `Required Module Units`，然后找到或构建这些 `Required Module Units` 所对应的 BMI，然后拼接编译选项即可。

第一个问题是在源代码当中，我们只能看到 `import a;` `import b;` `import c;` 这样的语句，我们并不能确定当前 TU 中 `import a;` 所指的 `a module` 是由哪一个 module unit 提供的。为了解决这个问题，**对于每个 Tab Page**， 我们会在看到 `import` 时扫描整个项目以获取从 Module Name 到 Module Unit 的映射。我们会全局维护一个 `ModuleName` 到 `Module Unit` 的映射作为 Cache。

然后我们可以递归地得到当前 TU 直接依赖和间接依赖的 Module Units  `Required Module Units`。然后我们可以 Bottom Up 地对  `Required Module Units` 中的 Module Units 执行 `GetOrBuildBMI` 逻辑。我们维护一个全局的 `ModuleName` 到 `BMI` 文件的映射作为 Cache。

同时因为 import 和 `#include` 都可以认为是为当前 TU 引入依赖，import 也一般都会被放在 TU 的最前面，所以当前 TU 所需的 BMI 自然成为了当前 TU 的 Preamble。（与 PCH 不同，每个 Tab Page 只存一份到全局 BMI 文件的引用，避免了相同 BMI 文件的反复构建和冗余存储）。所以我们也需要在 `isPreambleCompatible` 中管理当前 TU 所 import 的 module 是否依然可复用。我们通过编译器提供的能力来检测 BMI 是否过时。

以上，即是 Clangd 中支持 C++20 Modules 的全部 high level 逻辑了。其中真正最复杂的两个逻辑 “如何构建 BMI” 和 “如何检测 BMI 是否过时” 实际上都是通过 Clang 完成的，Clangd 中只需要管理如何调用 BMI 以及如何管理 BMI 的生命周期而已了。

## 实现

本节说明 Clangd 对 C++20 Modules 的实现细节，对实现细节暂不感兴趣的读者可以跳过。

我们可以通过以下三个抽象作为理解 Clangd 中 C++20 Modules 的支持的入口
1. **ProjectModules** - 查询项目中的 Modules 信息
3. **PrerequisiteModules** - 一个文件所需的 Modules 信息，也是 Preamble 中所需的 Modules 信息
2. **ModulesBuilder** - 构建 Modules 文件（.pcm）

### ProjectModules

```cpp
class ProjectModules {
public:
  using CommandMangler =
      llvm::unique_function<void(tooling::CompileCommand &, PathRef) const>;

  virtual std::vector<std::string> getRequiredModules(PathRef File) = 0;
  virtual std::string getModuleNameForSource(PathRef File) = 0;
  virtual std::string getSourceForModuleName(llvm::StringRef ModuleName,
                                             PathRef RequiredSrcFile) = 0;

  virtual void setCommandMangler(CommandMangler Mangler) {}

  virtual ~ProjectModules() = default;
};
```

ProjectModules 由于查询一个项目中 Modules 的信息。
其中 `setCommandMangler(CommandMangler Mangler)` 用于处理 clang 处理不了的命令，一般用于使用 GCC 编译的场景下。

其他接口为查看一个文件直接 `import` 的 Modules，查看一个文件的 `Module Name` 以及查看一个 `Module Name` 对应的文件。

因为其生态位类似 Compilation Database，即为 clangd 提供项目的信息，所以接口上设计我们需要从 GlobalCompilationDatabase 获取 ProjectModules。

### ScanningAllProjectModules

```C++
class ScanningAllProjectModules : public ProjectModules {
public:
  ScanningAllProjectModules(
      std::shared_ptr<const clang::tooling::CompilationDatabase> CDB,
      const ThreadsafeFS &TFS)
      : Scanner(CDB, TFS) {}

  ~ScanningAllProjectModules() override = default;

  std::vector<std::string> getRequiredModules(PathRef File) override;

  void setCommandMangler(CommandMangler Mangler) override;

  /// RequiredSourceFile is not used intentionally. See the comments of
  /// ModuleDependencyScanner for detail.
  std::string getSourceForModuleName(llvm::StringRef ModuleName,
                                     PathRef RequiredSourceFile) override;

  std::string getModuleNameForSource(PathRef File) override;

private:
  ModuleDependencyScanner Scanner;
  CommandMangler Mangler;
};

std::unique_ptr<ProjectModules> scanningProjectModules(
    std::shared_ptr<const clang::tooling::CompilationDatabase> CDB,
    const ThreadsafeFS &TFS) {
  return std::make_unique<ScanningAllProjectModules>(CDB, TFS);
}
```

ScanningAllProjectModules 则实现了前面所述的扫描 Compilation Database 中所有文件的逻辑。实现底层调用了与 clang-scan-deps 中扫描 Modules 相同的 API。

### CachingProjectModules

```C++
class CachingProjectModules : public ProjectModules {
public:
  CachingProjectModules(std::unique_ptr<ProjectModules> MDB,
                        ModuleNameToSourceCache &Cache);

  std::vector<std::string> getRequiredModules(PathRef File) override;

  std::string getModuleNameForSource(PathRef File) override;

  std::string getSourceForModuleName(llvm::StringRef ModuleName,
                                     PathRef RequiredSrcFile) override;

private:
  std::unique_ptr<ProjectModules> MDB;
  ModuleNameToSourceCache &Cache;
};
```

CachingProjectModules 用于全局缓存 Module Name 到 Module Unit 的映射。避免反复扫描所有文件造成浪费。

### PrerequisiteModules

```C++
class PrerequisiteModules {
public:
  /// Change commands to load the module files recorded in this
  /// PrerequisiteModules first.
  virtual void
  adjustHeaderSearchOptions(HeaderSearchOptions &Options) const = 0;

  /// Whether or not the built module files are up to date.
  /// Note that this can only be used after building the module files.
  virtual bool
  canReuse(const CompilerInvocation &CI,
           llvm::IntrusiveRefCntPtr<llvm::vfs::FileSystem>) const = 0;

  virtual ~PrerequisiteModules() = default;
};
```

PrerequisiteModules 用于表示一个 TU 所需要的 Modules 信息。这里我们只输出两个接口，`canReuse()`
用于查询这些 BMI 是否已过时，`void adjustHeaderSearchOptions(HeaderSearchOptions &Options)`
则用于将 BMI 信息直接拼接到编译命令中。这里我们特意避免了泄露 BMI 信息。

如前文所述，PrerequisiteModules 的语义天然地属于一个 TU 的前置信息，所以它被放置在 PreambleData 中。

```C++
struct PreambleData {
  PreambleData(PrecompiledPreamble Preamble) : Preamble(std::move(Preamble)) {}

  // Version of the ParseInputs this preamble was built from.
  std::string Version;
  tooling::CompileCommand CompileCommand;
  // Target options used when building the preamble. Changes in target can cause
  // crashes when deserializing preamble, this enables consumers to use the
  // same target (without reparsing CompileCommand).
  std::unique_ptr<TargetOptions> TargetOpts = nullptr;
  PrecompiledPreamble Preamble;
  std::vector<Diag> Diags;
  // Processes like code completions and go-to-definitions will need #include
  // information, and their compile action skips preamble range.
  IncludeStructure Includes;
  // Captures #include-mapping information in #included headers.
  std::shared_ptr<const include_cleaner::PragmaIncludes> Pragmas;
+ // Information about required module files for this preamble.
+ std::unique_ptr<PrerequisiteModules> RequiredModules;
  // Macros defined in the preamble section of the main file.
  // Users care about headers vs main-file, not preamble vs non-preamble.
  // These should be treated as main-file entities e.g. for code completion.
  MainFileMacros Macros;
  // Pragma marks defined in the preamble section of the main file.
  std::vector<PragmaMark> Marks;
  // Cache of FS operations performed when building the preamble.
  // When reusing a preamble, this cache can be consumed to save IO.
  std::shared_ptr<PreambleFileStatusCache> StatCache;
  // Whether there was a (possibly-incomplete) include-guard on the main file.
  // We need to propagate this information "by hand" to subsequent parses.
  bool MainIsIncludeGuarded = false;
};
```

### ModulesBuilder

```C++
class ModulesBuilder {
public:
  ModulesBuilder(const GlobalCompilationDatabase &CDB);
  ~ModulesBuilder();

  ModulesBuilder(const ModulesBuilder &) = delete;
  ModulesBuilder(ModulesBuilder &&) = delete;

  ModulesBuilder &operator=(const ModulesBuilder &) = delete;
  ModulesBuilder &operator=(ModulesBuilder &&) = delete;

  std::unique_ptr<PrerequisiteModules>
  buildPrerequisiteModulesFor(PathRef File, const ThreadsafeFS &TFS);

private:
  class ModulesBuilderImpl;
  std::unique_ptr<ModulesBuilderImpl> Impl;
};
```

ModulesBuilder 是 clangd 实现 C++20 Modules 的核心逻辑。主要逻辑都在 `ModulesBuilder.cpp` 中。

```C++
std::unique_ptr<PrerequisiteModules>
ModulesBuilder::buildPrerequisiteModulesFor(PathRef File,
                                            const ThreadsafeFS &TFS) {
  std::unique_ptr<ProjectModules> MDB = Impl->getCDB().getProjectModules(File);
  if (!MDB) {
    elog("Failed to get Project Modules information for {0}", File);
    return std::make_unique<FailedPrerequisiteModules>();
  }
  CachingProjectModules CachedMDB(std::move(MDB),
                                  Impl->getProjectModulesCache());

  std::vector<std::string> RequiredModuleNames =
      CachedMDB.getRequiredModules(File);
  if (RequiredModuleNames.empty())
    return std::make_unique<ReusablePrerequisiteModules>();

  auto RequiredModules = std::make_unique<ReusablePrerequisiteModules>();
  for (llvm::StringRef RequiredModuleName : RequiredModuleNames) {
    // Return early if there is any error.
    if (llvm::Error Err = Impl->getOrBuildModuleFile(
            File, RequiredModuleName, TFS, CachedMDB, *RequiredModules.get())) {
      elog("Failed to build module {0}; due to {1}", RequiredModuleName,
           toString(std::move(Err)));
      return std::make_unique<FailedPrerequisiteModules>();
    }
  }

  return std::move(RequiredModules);
}
```

`ModulesBuilder::buildPrerequisiteModulesFor` 会去调用 `ModulesBuilderImpl::getOrBuildModuleFile`。


```C++
llvm::Error ModulesBuilder::ModulesBuilderImpl::getOrBuildModuleFile(
    PathRef RequiredSource, StringRef ModuleName, const ThreadsafeFS &TFS,
    CachingProjectModules &MDB, ReusablePrerequisiteModules &BuiltModuleFiles) {
  if (BuiltModuleFiles.isModuleUnitBuilt(ModuleName))
    return llvm::Error::success();

  std::string ModuleUnitFileName =
      MDB.getSourceForModuleName(ModuleName, RequiredSource);
  /// It is possible that we're meeting third party modules (modules whose
  /// source are not in the project. e.g, the std module may be a third-party
  /// module for most project) or something wrong with the implementation of
  /// ProjectModules.
  /// FIXME: How should we treat third party modules here? If we want to ignore
  /// third party modules, we should return true instead of false here.
  /// Currently we simply bail out.
  if (ModuleUnitFileName.empty())
    return llvm::createStringError(
        llvm::formatv("Don't get the module unit for module {0}", ModuleName));

  /// Try to get prebuilt module files from the compilation database first. This
  /// helps to avoid building the module files that are already built by the
  /// compiler.
  getPrebuiltModuleFile(ModuleName, ModuleUnitFileName, TFS, BuiltModuleFiles);

  // Get Required modules in topological order.
  auto ReqModuleNames = getAllRequiredModules(RequiredSource, MDB, ModuleName);
  for (llvm::StringRef ReqModuleName : ReqModuleNames) {
    if (BuiltModuleFiles.isModuleUnitBuilt(ReqModuleName))
      continue;

    if (auto Cached = Cache.getModule(ReqModuleName)) {
      if (IsModuleFileUpToDate(Cached->getModuleFilePath(), BuiltModuleFiles,
                               TFS.view(std::nullopt))) {
        log("Reusing module {0} from {1}", ReqModuleName,
            Cached->getModuleFilePath());
        BuiltModuleFiles.addModuleFile(std::move(Cached));
        continue;
      }
      Cache.remove(ReqModuleName);
    }

    std::string ReqFileName =
        MDB.getSourceForModuleName(ReqModuleName, RequiredSource);
    llvm::Expected<std::shared_ptr<BuiltModuleFile>> MF = buildModuleFile(
        ReqModuleName, ReqFileName, getCDB(), TFS, BuiltModuleFiles);
    if (llvm::Error Err = MF.takeError())
      return Err;

    log("Built module {0} to {1}", ReqModuleName, (*MF)->getModuleFilePath());
    Cache.add(ReqModuleName, *MF);
    BuiltModuleFiles.addModuleFile(std::move(*MF));
  }

  return llvm::Error::success();
}
```

剔除 Modules 已被构建的情况，构建 Modules 的逻辑一方面是先从构建命令中获取之前编译好的 BMI（即前文所述的由构建系统构建的 BMI），然后递归的扫描所有依赖的文件以获取所有直接或间接 import 的 module unit。
之后若无法在 cache 中查找到对应的 BMI，则通过 `buildModuleFile` 接口进行构建。

```C++
/// Build a module file for module with `ModuleName`. The information of built
/// module file are stored in \param BuiltModuleFiles.
llvm::Expected<std::shared_ptr<BuiltModuleFile>>
buildModuleFile(llvm::StringRef ModuleName, PathRef ModuleUnitFileName,
                const GlobalCompilationDatabase &CDB, const ThreadsafeFS &TFS,
                const ReusablePrerequisiteModules &BuiltModuleFiles) {
  // Try cheap operation earlier to boil-out cheaply if there are problems.
  auto Cmd = CDB.getCompileCommand(ModuleUnitFileName);
  if (!Cmd)
    return llvm::createStringError(
        llvm::formatv("No compile command for {0}", ModuleUnitFileName));

  llvm::SmallString<256> ModuleFilesPrefix =
      getUniqueModuleFilesPath(ModuleUnitFileName);

  Cmd->Output = getModuleFilePath(ModuleName, ModuleFilesPrefix);

  ParseInputs Inputs;
  Inputs.TFS = &TFS;
  Inputs.CompileCommand = std::move(*Cmd);

  IgnoreDiagnostics IgnoreDiags;
  auto CI = buildCompilerInvocation(Inputs, IgnoreDiags);
  if (!CI)
    return llvm::createStringError("Failed to build compiler invocation");

  auto FS = Inputs.TFS->view(Inputs.CompileCommand.Directory);
  auto Buf = FS->getBufferForFile(Inputs.CompileCommand.Filename);
  if (!Buf)
    return llvm::createStringError("Failed to create buffer");

  // In clang's driver, we will suppress the check for ODR violation in GMF.
  // See the implementation of RenderModulesOptions in Clang.cpp.
  CI->getLangOpts().SkipODRCheckInGMF = true;

  // Hash the contents of input files and store the hash value to the BMI files.
  // So that we can check if the files are still valid when we want to reuse the
  // BMI files.
  CI->getHeaderSearchOpts().ValidateASTInputFilesContent = true;

  BuiltModuleFiles.adjustHeaderSearchOptions(CI->getHeaderSearchOpts());

  CI->getFrontendOpts().OutputFile = Inputs.CompileCommand.Output;
  auto Clang =
      prepareCompilerInstance(std::move(CI), /*Preamble=*/nullptr,
                              std::move(*Buf), std::move(FS), IgnoreDiags);
  if (!Clang)
    return llvm::createStringError("Failed to prepare compiler instance");

  GenerateReducedModuleInterfaceAction Action;
  Clang->ExecuteAction(Action);

  if (Clang->getDiagnostics().hasErrorOccurred()) {
    std::string Cmds;
    for (const auto &Arg : Inputs.CompileCommand.CommandLine) {
      if (!Cmds.empty())
        Cmds += " ";
      Cmds += Arg;
    }

    clangd::vlog("Failed to compile {0} with command: {1}", ModuleUnitFileName,
                 Cmds);

    std::string BuiltModuleFilesStr = BuiltModuleFiles.getAsString();
    if (!BuiltModuleFilesStr.empty())
      clangd::vlog("The actual used module files built by clangd is {0}",
                   BuiltModuleFilesStr);

    return llvm::createStringError(
        llvm::formatv("Failed to compile {0}. Use '--log=verbose' to view "
                      "detailed failure reasons. It is helpful to use "
                      "'--debug-modules-builder' flag to keep the clangd's "
                      "built module files to reproduce the failure for "
                      "debugging. Remember to remove them after debugging.",
                      ModuleUnitFileName));
  }

  return BuiltModuleFile::make(ModuleName, Inputs.CompileCommand.Output);
}
```

这段代码的核心逻辑即是从 Compilation Database 中获取编译选项，然后调用 Clang API 进行编译。

其中

```C++
BuiltModuleFiles.adjustHeaderSearchOptions(CI->getHeaderSearchOpts());
```

这使用 clangd 构建的 BMI 替换 Compilation Database 中提供的 BMI。用户可以理解为替换/增加编译命令中的 `-fmodule=<module-name>=<BMI-path>` 选项。


```C++
CI->getHeaderSearchOpts().ValidateASTInputFilesContent = true;
```

这是启用编译器中的一致性检查功能。编译器在写 BMI 时会记录所有的输入文件（Module Unit 即被 include 的文件）。
然后当 `ValidateASTInputFilesContent` 选项为真时，当此 BMI 被 import 时，编译器会检查此前纪录的所有文件的哈希值是否依然相等。

### Reuse

接上文，通过编译器的能力，我们可以通过 try-import 的方式测试 BMI 是否依然可复用。

```C++
bool ReusablePrerequisiteModules::canReuse(
    const CompilerInvocation &CI,
    llvm::IntrusiveRefCntPtr<llvm::vfs::FileSystem> VFS) const {
  if (RequiredModules.empty())
    return true;

  llvm::SmallVector<llvm::StringRef> BMIPaths;
  for (auto &MF : RequiredModules)
    BMIPaths.push_back(MF->getModuleFilePath());
  return IsModuleFilesUpToDate(BMIPaths, *this, VFS);
}
```

```C++
bool IsModuleFileUpToDate(PathRef ModuleFilePath,
                          const PrerequisiteModules &RequisiteModules,
                          llvm::IntrusiveRefCntPtr<llvm::vfs::FileSystem> VFS) {
  HeaderSearchOptions HSOpts;
  RequisiteModules.adjustHeaderSearchOptions(HSOpts);
  HSOpts.ForceCheckCXX20ModulesInputFiles = true;
  HSOpts.ValidateASTInputFilesContent = true;

  clang::clangd::IgnoreDiagnostics IgnoreDiags;
  DiagnosticOptions DiagOpts;
  IntrusiveRefCntPtr<DiagnosticsEngine> Diags =
      CompilerInstance::createDiagnostics(*VFS, DiagOpts, &IgnoreDiags,
                                          /*ShouldOwnClient=*/false);

  LangOptions LangOpts;
  LangOpts.SkipODRCheckInGMF = true;

  FileManager FileMgr(FileSystemOptions(), VFS);

  SourceManager SourceMgr(*Diags, FileMgr);

  HeaderSearch HeaderInfo(HSOpts, SourceMgr, *Diags, LangOpts,
                          /*Target=*/nullptr);

  PreprocessorOptions PPOpts;
  TrivialModuleLoader ModuleLoader;
  Preprocessor PP(PPOpts, *Diags, LangOpts, SourceMgr, HeaderInfo,
                  ModuleLoader);

  IntrusiveRefCntPtr<ModuleCache> ModCache = createCrossProcessModuleCache();
  PCHContainerOperations PCHOperations;
  CodeGenOptions CodeGenOpts;
  ASTReader Reader(PP, *ModCache, /*ASTContext=*/nullptr,
                   PCHOperations.getRawReader(), CodeGenOpts, {});

  // We don't need any listener here. By default it will use a validator
  // listener.
  Reader.setListener(nullptr);

  if (Reader.ReadAST(ModuleFilePath, serialization::MK_MainFile,
                     SourceLocation(),
                     ASTReader::ARR_None) != ASTReader::Success)
    return false;

  bool UpToDate = true;
  Reader.getModuleManager().visit([&](serialization::ModuleFile &MF) -> bool {
    Reader.visitInputFiles(
        MF, /*IncludeSystem=*/false, /*Complain=*/false,
        [&](const serialization::InputFile &IF, bool isSystem) {
          if (!IF.getFile() || IF.isOutOfDate())
            UpToDate = false;
        });
    return !UpToDate;
  });
  return UpToDate;
}

bool IsModuleFilesUpToDate(
    llvm::SmallVector<PathRef> ModuleFilePaths,
    const PrerequisiteModules &RequisiteModules,
    llvm::IntrusiveRefCntPtr<llvm::vfs::FileSystem> VFS) {
  return llvm::all_of(
      ModuleFilePaths, [&RequisiteModules, VFS](auto ModuleFilePath) {
        return IsModuleFileUpToDate(ModuleFilePath, RequisiteModules, VFS);
      });
}
```

这种做法最大的缺点是加载 BMI 的成本过高。这里应该有充分的优化空间。

# Future TODOs

至此，我们已介绍完了目前 Clangd 中 C++20 Modules 的支持原理和情况。大家在遇到问题时可以先根据自己的情况尝试修复。除了 bug fix 之外，其他能加入的优化和 feature：

## Reuse 优化

如上文提到的，目前的 Modules BMI Reuse 的实现依赖编译器加载 BMI 后给出结果。这可能太慢了。我们可能可以在 Clangd 中每个 BMI 涉及到的文件进行记录，然后 Reuse 时直接比较这些文件。

## 扫描优化

现在虽然有扫描 Cache，但对于超大规模项目可能会因为长时间扫描有比较长的 warmup 时间。我们可以尝试通过一些方式进行优化。

### 后缀优化

一种简单可行的优化是，在寻找 Module Name 到 Module Unit 的映射时，我们可以只扫描带有特殊后缀（.cppm, .ixx）的文件。

### Lazy Scanning

另一种优化是避免全局扫描，而是根据所需要查询的 Module Name 进行按需扫描，例如当我们需要 module a 时，当发现任何一个 module unit 声明了 `a module`，我们就可以停止扫描返回该结果了。
这个优化可以和后缀优化一起使用。

## BMI 持久化

现在 clangd 为了避免资源泄漏，会在关闭 Tab 时减少此 Tab 所需的 BMI 的引用。当一个 BMI 的引用归 0 时，这个 BMI 会被删掉。同时直接关闭 clangd 时，clangd 构建的所有 BMI 也会被删除。

这导致我们关闭 clangd 再开启 clangd 后，可能会等待较长的时间构建许多 BMI。我们可能设计一些策略，在避免资源泄漏的情况下，将 BMI 持久地存储下来。

## Proper Compilation Database for modules

在前文，我有意地避免提及当前 clangd 对 C++20 Modules 支持的两个缺陷，这两个缺陷需要更好的 Compilation Database 来支持。相关的工作有 [p2977r2](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2977r2.html)。

### 重复的 Module Name

考虑这么一个项目：

```C++
// a.cppm
export module a;
```

```C++
// a.v1.cppm
export module a;
```

```C++
// main.cc
import a;
int main() { return 0; }
```

```
add_executable(exe main.cc a.cppm)
add_library(a a.v1.cppm)
```
(fake cmake)

这是一个正确的项目吗？这个项目在不同的 TU 中包含了重复的 Module。这个项目是正确的。因为 ISOCPP 的规定是，在一个链接的程序中，不允许有重名的 Module。但如果没有链接在一起，那重名并不影响正确性。

在 clangd 中，我们错误的假设了整个项目中不包含重名的 module。在目前的设计中， clangd 也没法处理不同 module unit 声明相同的 module。因为 clangd 没有足够的信息来判断，对于一个特定的 TU，哪一个 module unit 才是正确的。

clangd 需要构建系统提供更细粒度的 Compilation 信息才可以做出正确的判断。

（当然 clangd 可能有办法进行一些猜测和尝试，但这里不做额外讨论了）

### 相同 module unit 的不同编译选项

例如

```
// a.cc
// compile with -std=c++23
import std;
...
```

```
// b.cc
// compile with -std=c++26
import std;
...
```

在这个例子中，理想情况下，我们需要将为 std module 提供 -std=c++23 和 -std=c++26 两个版本的 BMI。但目前我们缺乏足够的信息。

除了依赖构建系统提供更好的 Compilation Database 之外，对于这种情况 clangd 可能可以做一些更有效率的猜测。但因为这种情况现实里出现得不多，我们目前也没用遇到。

## BMI Building Library

如上文所述，clangd 支持 Modules 的逻辑其实和很多其他工具支持 Modules 所需的逻辑是高度类似的。在这种情况下，可能将这些逻辑抽象为一个单独的库会好很多。不然 clang-tidy 等工具可能每一个都得做类似的处理。

# 总结

本文介绍了 Clangd 中 C++20 Modules 支持的方法和现状，希望可以鼓励更多人参与到 Clangd 对 C++20 Modules 的支持中，也希望对大家使用 clangd C++20 Modules 有帮助。
