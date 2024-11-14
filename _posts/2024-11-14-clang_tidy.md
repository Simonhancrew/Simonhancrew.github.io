---
title: 配一下clang-tidy
date: 2024-11-14 11:40:10 +0800
categories: [Blogging, tools, clang-tidy]
tags: [writing]
---


参考[clang-tidy checkers](https://clang.llvm.org/extra/clang-tidy/checks/list.html)

最后成品

```
UseColor: true

Checks: >
  android-cloexec-*,
  android-comparison-in-temp-failure-retry,
  bugprone-*,
  clang-analyzer-*,
  modernize-*,
  -modernize-use-trailing-return-type,
  readability-*,
  performance-*,
  cert-*,
  concurrency-*,
  cppcoreguidelines-init-variables,
  cppcoreguidelines-rvalue-reference-param-not-moved,
  cppcoreguidelines-special-member-functions,
  cppcoreguidelines-narrowing-conversions,
  google-build-explicit-make-pair,
  google-build-namespaces,
  google-build-using-namespace,
  google-default-arguments,
  google-explicit-constructor,
  google-global-names-in-headers,
  google-readability-avoid-underscore-in-googletest-name,
  google-readability-casting,
  google-readability-todo,
  google-runtime-int,
  google-runtime-operator,
  hicpp-avoid-goto,
  hicpp-exception-baseclass,
  hicpp-multiway-paths-covered,
  hicpp-no-assembler,
  hicpp-signed-bitwise,
  linuxkernel-must-use-errs,
  llvm-header-guard,
  llvm-include-order,
  llvm-namespace-comment,
  llvm-prefer-isa-or-dyn-cast-in-conditionals,
  llvm-prefer-register-over-unsigned,
  llvm-twine-local,
  misc-definitions-in-headers,
  misc-misplaced-const,
  misc-new-delete-overloads,
  misc-no-recursion,
  misc-non-copyable-objects,
  misc-redundant-expression,
  misc-static-assert,
  misc-throw-by-value-catch-by-reference,
  misc-unconventional-assign-operator,
  misc-uniqueptr-reset-release,
  misc-unused-alias-decls,
  misc-unused-parameters,
  misc-unused-using-decls,
  portability-restrict-system-includes,
  portability-simd-intrinsics,
  zircon-temporary-objects,
  clang-analyzer-core*,
  clang-analyzer-cplusplus.*
```
