---
layout: post
title: "C++20 Modules Support in Clangd"
date: 2025-12-3 +0800
categories: C++
toc: true
---

*The post was written in [Chinese](./Clangd-support-for-Modules.html) and translated by LLM. Feel free to contact me if any phrasing seems unnatural.*

* TOC
{:toc}

The purpose of this blog is to encourage more people to contribute to C++20 Modules support in Clangd.  

The motivation for writing it is that I’ve noticed a mismatch between three things:  
the number of people complaining about C++20 Modules code completion,  
the number of contributors to C++20 Modules support in Clangd,  
and the actual complexity of C++20 Modules support in Clangd.  

So I suspect most people don’t realize that supporting C++20 Modules in Clangd is actually quite simple.  
That’s why I wrote this blog: to explain to more people that, unlike the complexity of implementing Modules in the compiler itself, the complexity of supporting C++20 Modules in clangd (and similar tools such as clang-tidy or other language servers) is not high.

Also, I’d like to recommend [Clice](https://github.com/clice-io/clice). It doesn’t support Modules yet, but it looks promising.

# Background: C++20 Modules

Before C++20 Modules, any translation unit could be compiled directly. Virtually all C++ tools (build systems, static analysis tools, etc.) rely on this assumption.  

After C++20 Modules were introduced, this assumption became invalid.  
This is why C++20 Modules have a huge impact on tooling, and also one of the reasons real-world adoption has been relatively slow.

Take a minimal C++20 Modules example:

```C++
// a.cppm
export module a;
```

```C++
// b.cppm
export module b;
import a;
```

To compile `b.cppm`, we now “need” two compilation commands:

```bash
$ clang++ -std=c++20 a.cppm --precompile -o a.pcm
$ clang++ -std=c++20 b.cppm -fmodule-file=a=a.pcm -fsyntax-only
```

(For this extremely simple case, the compiler might be able to handle everything with a single invocation, but that’s not scalable unless we embed a build system into the compiler itself. That direction is out of scope here.)

Here `a.pcm` is called a BMI*, which is the serialized form of the information about `a.cppm` that the compiler needs to expose externally. All translation units that directly or indirectly `import a;` need the BMI of `a.cppm` in order to compile.

\* BMI stands for Binary Module Interface or Built Module Interface. The exact wording is debated, but the acronym BMI itself is not.

# What the Toolchain Must Do to Support Modules

For tools other than the compiler, to support Modules they must do the following:

- For any translation unit containing `import`, we need to compute the module units it directly and indirectly imports. We’ll call these module units the **Required Module Units**. Using this information, we compute a DAG (directed acyclic graph) of dependencies for the translation unit.
- Obtain the BMIs corresponding to these Required Module Units. If a BMI doesn’t exist, the tool must call the compiler in dependency order (according to the DAG) to build the required BMIs.
- For a translation unit that has `import` statements, we must use the above information to synthesize the compilation command required to compile that translation unit.

In particular, after we build a project and generate a [Compilation Database](https://clang.llvm.org/docs/JSONCompilationDatabase.html), for each translation unit in it we already have both the compilation command and all the BMIs required to build that unit. In such a situation, many tools are actually usable.  

This is why we can integrate clang-tidy into projects using Modules without explicit clang-tidy support: compiling the project once provides enough BMIs.

For this reason, we recommend that build systems provide a special mode like “build BMIs only”, so that when users only need clang-tidy-like workflows, they can just build all BMIs in the project without compiling the entire project.

# Steps Needed for Clangd to Support Modules

Similarly, once a project is built and a [Compilation Database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) is generated, if we feed this Compilation Database into Clangd, we may find that Clangd suddenly starts working in a Modules-using project.

However, this is not ideal, because:

- It requires the Clangd version and the Clang used for building to be exactly the same. This implies we cannot use GCC or MSVC as the build tool, which conflicts with the design philosophy of Clangd. This applies to all tools in general.
- As a language server, Clangd must provide immediate feedback. If Clangd doesn’t explicitly support Modules, then when we modify a Module Interface, its corresponding BMI won’t be updated automatically, and we cannot see the effect of the change unless we rebuild the entire project. The user experience is poor.

The second point is the more urgent reason to explicitly support C++20 Modules in Clangd.  
Other tools like clang-tidy can accept a “build first, then analyze” workflow, but for clangd the difference in user experience is huge if changes are not reflected immediately.

# How Clangd Works

At a high level, the core logic of Clangd is in `clang-tools-extra/clangd/TUScheduler.cpp`, with detailed comments for those interested.

Clangd creates a Worker for each open tab. Each Worker runs in a separate thread. The Worker uses the Clang API to “compile” (parse) the file corresponding to its tab and obtain an AST. When further events arrive (hover, go to definition, etc.), the Worker first checks whether the existing AST is still valid; if it is outdated, the Worker recompiles, and then uses the AST to answer the requests.

From this perspective, supporting C++20 Modules in Clangd mainly impacts two parts:
- the compilation process, and  
- checking whether a compilation result is outdated.

For the parts that *use* the AST, if we don’t require new features and only want clangd to keep working, we can temporarily ignore them. An example of a new feature would be suggesting possible module names after typing `import A` – this is not implemented yet. From personal experience, current AI-based suggestions already do a decent job here, so I don’t have strong needs in this area.

## Compilation

As mentioned earlier, prior to C++20 Modules, each translation unit was independent and just needed its compilation command and files to compile. Clangd gets the compile command for each translation unit from the Compilation Database, then compiles the current file to obtain an AST. Most of this logic is in `TUScheduler.cpp`.

To speed up compilation, the Clangd Worker uses PCH (precompiled headers) for each open tab: it precompiles header files and saves the result as a **Preamble**. The benefit is that when we open a tab and edit the file (without modifying the “header” portion), recompilation of that tab can reuse the precompiled header portion, avoiding recompiling the headers – a reasonable and effective optimization that greatly speeds up Clangd.

This approach has downsides: each Preamble belongs to a different AST Worker, so opening a new tab causes Clangd to spend extra time building a Preamble for that tab. Also, since each tab has its own Preamble, opening many tabs increases Clangd’s memory usage.

From this perspective, for projects that heavily use C++20 Modules, once warmup is complete, we can reasonably expect Clangd’s memory consumption to decrease significantly.

## Detecting Outdated Compilation Results

Another benefit of the Preamble design is that it divides the content of a tab into two parts: dependencies (the Preamble) and the file’s own body. Detecting whether the compilation result is outdated naturally becomes:

- checking whether the current file content is outdated, and  
- checking whether the dependencies (the Preamble) are outdated.

Checking whether the file itself is stale is easy. We only need to check whether the dependencies / Preamble are outdated.

This is implemented in [isPreambleCompatible](https://github.com/llvm/llvm-project/blob/822fc449985553c609e44915374f935672c0db50/clang-tools-extra/clangd/Preamble.cpp#L745-L757).

# C++20 Modules Support in Clangd

## High-Level Status

Clangd already has basic support for C++20 Modules. You can enable it via the `--experimental-modules-support` option. Locally this works as expected, but there are still many complaints in the community, and many issues are hard to reproduce.

This is another main reason for this blog: you can quickly test on your own setup, and if you find bugs or feel like implementing a small feature, you can contribute it.

As mentioned, using C++20 Modules requires a Compilation Database. The Compilation Database must include compile options for *all* TUs in the project. Many common reports in the community are either missing the Compilation Database, or the Compilation Database doesn’t contain all required translation units and their commands.  
For example, if your project uses `import std;`, we must see a translation unit that builds the `std` module in the Compilation Database passed to clangd.

## Testing and Debugging

Because Clangd can reuse BMIs built by the build system via the Compilation Database, when testing C++20 Modules support in Clangd you can delete all BMIs after generating the Compilation Database, to avoid false negatives.

If you run into problems with Clangd’s C++20 Modules feature, you can start by enabling `--log=verbose` and checking Clangd’s output; there may be useful clues.

If Clangd reports errors while compiling module units, you should first check whether your project can be built successfully by the same version of the Clang compiler. If so, but Clangd *still* reports errors or crashes when compiling those module units, try `--debug-modules-builder`.  

Without `--debug-modules-builder`, Clangd deletes all BMIs it built on exit, which makes reproducing errors or crashes harder. With `--debug-modules-builder` enabled, Clangd keeps its BMIs. Combined with the compilation commands printed under `--log=verbose`, you can reproduce Clangd’s errors in a terminal.

Another probable issue, especially when you heavily mix `#include` and `import`, is incompatibility between Clangd’s PCH and C++20 Modules. Theoretically PCH, Clang header modules, and C++20 Modules are compatible, but due to a lack of tests and experience, many problems I encountered early on came from interactions between Clangd’s PCH and C++20 Modules.  

Currently I don’t see a Clangd option to disable PCH, but we could add one in the future (for example, avoid building PCH for TUs that use `import`).

## Design Approach

As described earlier, supporting C++20 Modules in clangd means that when opening a tab we need to:

1. Find all direct and indirect module units it imports (its **Required Module Units**).
2. Find or build the BMIs corresponding to these module units.
3. Use that information to assemble the compilation options.

The first problem is: from the source code we only see statements like `import a;`, `import b;`, `import c;`, but we don’t know which module unit provides `module a`.  

To solve this, **for each tab page**, once we see `import`, we scan the entire project to obtain a mapping from module names to module units. Globally we maintain a cache mapping `ModuleName -> Module Unit`.

Then we can recursively compute the direct and indirect module unit dependencies of the current TU (the Required Module Units). We then process them bottom-up with a `GetOrBuildBMI` logic. We also maintain a global cache mapping `ModuleName -> BMI file`.

Because both `import` and `#include` bring dependencies into the current TU, and imports are typically placed at the top of the file, the BMIs needed by the current TU naturally become part of its Preamble. (Unlike PCH, each tab only keeps references to the global BMI files, avoiding repeated construction and storage of identical BMIs.)  

Therefore we must also manage whether imported modules are still reusable within `isPreambleCompatible`. We rely on compiler functionality to test whether a BMI is out of date.

That’s the entire high-level logic for C++20 Modules support in Clangd. The two truly complex bits—“how to build BMIs” and “how to detect whether BMIs are outdated”—are actually handled by Clang. Clangd only needs to manage how to call the BMI builder and how to manage BMI lifetimes.

## Implementation

This section covers implementation details in Clangd. Skip it if you’re not interested in the code-level view.

You can understand C++20 Modules support in Clangd through these three abstractions:

1. **ProjectModules** – query module information in a project  
2. **PrerequisiteModules** – modules required by a file (also part of the Preamble’s required modules)  
3. **ModulesBuilder** – builds module files (.pcm)

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

`ProjectModules` is responsible for querying module information in a project.  

`setCommandMangler(CommandMangler Mangler)` handles commands that Clang can’t process directly, typically in GCC-based build environments.

Other methods are for querying:
- the modules directly imported by a file,  
- the module name of a given source file,  
- and the source file providing a given module name.

Because its role is similar to a Compilation Database—providing project information to clangd—the interface is designed so that we obtain `ProjectModules` from the `GlobalCompilationDatabase`.

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

`ScanningAllProjectModules` implements the logic of scanning all files in the Compilation Database, as described earlier. Under the hood it uses the same API as `clang-scan-deps` for module scanning.

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

`CachingProjectModules` globally caches the mapping from module names to module units to avoid repeatedly scanning all files.

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

`PrerequisiteModules` represents the module information required by a TU. We expose two main interfaces:

- `canReuse()` – check whether the BMIs are outdated  
- `adjustHeaderSearchOptions(HeaderSearchOptions &Options)` – splice BMI information into the compile command, without exposing BMI details.

As mentioned, `PrerequisiteModules` naturally acts as prerequisite information for a TU, so it is placed in `PreambleData`:

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

`ModulesBuilder` is the core of C++20 Modules implementation in clangd. Most logic is in `ModulesBuilder.cpp`:

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

`ModulesBuilder::buildPrerequisiteModulesFor` calls `ModulesBuilderImpl::getOrBuildModuleFile`.

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

Ignoring the case where the module is already built, the logic is:

1. Try to obtain a prebuilt BMI from the build’s compile commands (BMIs built by the build system).  
2. Recursively find all directly or indirectly imported module units.  
3. If we can’t find a BMI in the cache, call `buildModuleFile` to build it.

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

The core logic here is:
- get compile options from the Compilation Database,  
- then call Clang’s APIs to compile.

In particular:

```C++
BuiltModuleFiles.adjustHeaderSearchOptions(CI->getHeaderSearchOpts());
```

This replaces BMIs provided by the Compilation Database with BMIs built by clangd. In terms of the command line, you can think of this as replacing or augmenting options like `-fmodule-file=<module-name>=<BMI-path>`.

```C++
CI->getHeaderSearchOpts().ValidateASTInputFilesContent = true;
```

This enables the compiler’s consistency checks: when writing a BMI, the compiler records all input files (module unit and its headers). Later, when `ValidateASTInputFilesContent` is true, importing this BMI will cause the compiler to recompute hashes of those files and check whether they still match.

### Reuse

Using compiler support, we can test whether a BMI is still reusable by attempting an import:

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

The biggest drawback of this approach is that loading BMIs is expensive. There is room for substantial optimization here.

## Related Patches

[1] https://github.com/llvm/llvm-project/pull/66462

[2] https://github.com/llvm/llvm-project/pull/106683

[3] https://github.com/llvm/llvm-project/pull/125988


# Future TODOs

We’ve now described the current design and status of C++20 Modules support in Clangd. If you run into issues, you can try to reason about them based on this description and perhaps fix them yourself. Besides bug fixes, there are also various optimizations and features that can be added.

## Reuse Optimization

As mentioned, `ReusablePrerequisiteModules` currently relies on the compiler to load BMIs to check whether they’re up to date. This may be too slow.  

We might maintain a record in Clangd of all files each BMI depends on, and when reusing, only compare those files instead of loading the BMI.

## Scanning Optimization

Even with a scanning cache, very large projects might still suffer from long warmup times due to full scans. We can explore several optimizations.

### Suffix-Based Optimization

A simple and practical optimization: when finding module name → module unit mappings, only scan files with special suffixes (`.cppm`, `.ixx`, etc.).

### Lazy Scanning

Another optimization is to avoid global scanning. Instead, we scan on demand for specific module names. For example, when we need module `a`, once we find *any* module unit that provides `module a`, we can stop scanning and return the result.  
This can be combined with suffix-based optimizations.

## BMI Persistence

Currently, to avoid resource leaks, Clangd reduces references to BMIs when tabs are closed, and when a BMI’s reference count drops to zero, it is deleted. Also, when Clangd exits, all BMIs it built are removed.

This means that if we restart Clangd, we may have to wait a long time to rebuild many BMIs. We might design strategies to persist BMIs while still avoiding leaks and unbounded disk growth.

## Proper Compilation Database for Modules

Earlier, I deliberately avoided mentioning two limitations in Clangd’s current C++20 Modules support. They both require a better Compilation Database. Related work includes [p2977r2](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2977r2.html).

### Duplicate Module Names

Consider a project like:

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

```cmake
add_executable(exe main.cc a.cppm)
add_library(a a.v1.cppm)
```

(fake CMake)

Is this a valid project?  
Yes. There are duplicate modules in different TUs, but the project is valid. ISO C++ only forbids having duplicate module names *within a linked program*. If they’re not linked together, duplicate names are fine.

In clangd we wrongly assume that there are no duplicate modules in an entire project. Under the current design, clangd also has no way to handle different module units with the same module name, because it doesn’t know which module unit is the right one for a given TU.

Clangd needs more fine-grained compilation information from the build system to make the right choice.

(Of course clangd might try some heuristics, but that’s beyond the scope here.)

### Different Compile Options for the Same Module Unit

For example:

```C++
// a.cc
// compile with -std=c++23
import std;
...
```

```C++
// b.cc
// compile with -std=c++26
import std;
...
```

Ideally we want two versions of the `std` module BMI: one built with `-std=c++23` and one with `-std=c++26`. Currently we don’t have enough information to properly handle this.

Besides asking the build system for a better Compilation Database, clangd might make some efficient guesses. But since such cases are rare in practice, we haven’t encountered many real-world examples.

## BMI Building Library

As described, the logic required for clangd to support Modules is very similar to what many other tools need. In that case, it might be better to extract this logic into a separate library. Otherwise, tools like clang-tidy might have to reimplement similar functionality themselves.

# Summary

This article describes how C++20 Modules support works in Clangd and its current status, with the hope of encouraging more contributions in this area and helping users understand and use clangd’s C++20 Modules support more effectively.
