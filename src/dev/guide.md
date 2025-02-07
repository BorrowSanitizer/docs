# Development Guide

BorrowSanitizer uses several extensions to the Rust and LLVM toolchains.

| Toolchain | Component                  |`./x.py build` | Link |
|-----------|----------------------------|:-----:|:----:|
| Rust      | Cargo Plugin               | `cargo-bsan` | [{{#include ../images/link.svg}}](https://github.com/BorrowSanitizer/rust/tree/bsan/src/tools/bsan-driver/cargo-bsan)     |
| Rust      | Rustc Plugin               | `bsan` |[{{#include ../images/link.svg}}](https://github.com/BorrowSanitizer/rust/tree/bsan/src/tools/bsan-driver/)     |
| LLVM      | Instrumentation Pass       | - |[{{#include ../images/link.svg}}](https://github.com/BorrowSanitizer/llvm-project/blob/bsan/llvm/lib/Transforms/Instrumentation/BorrowSanitizer.cpp)     |
| LLVM      | Sanitizer Interface        |`bsanrt` |[{{#include ../images/link.svg}}](https://github.com/BorrowSanitizer/llvm-project/tree/bsan/compiler-rt/lib/bsan)      |
| Rust    | Sanitizer Runtime Library  |`bsanrt` |[{{#include ../images/link.svg}}](https://github.com/BorrowSanitizer/rust/tree/bsan/src/tools/bsan/bsanrt)      |


## System Overview

Our Cargo plugin builds an instrumented [sysroot](https://rustc-dev-guide.rust-lang.org/building/bootstrapping/what-bootstrapping-does.html?highlight=sysroot#what-is-a-sysroot) by redirecting compiler invocations through our Rustc plugin, which adds the flags for enabling BorrowSanitizer. This saves time when switching between projects. Otherwise, users would need to rebuild Rust's standard library by passing `-Zbuild-std` for every project that they want to use with BorrowSanitizer. Any static optimizations that require Rust-specific type information will be implemented within the Rustc plugin. 

We modified the Rust compiler to accept `borrow` as one of the options for `-Z sanitizer=...`. If this option is enabled, then the compiler will emit "retag" instructions, which create new permissions for references. This has a  effect similar to the flag `-Zmir-emit-retag`, which Miri uses to emit retags. However, Miri only emits retags for a subset of operations, the rest are executed implicitly as side-effects of the interpreter. We emit retags everywhere that they are necessary for Tree Borrows. Our retags are lowered to LLVM as calls to a new intrinsic function—`@llvm.retag`.k Our LLVM insturmentation pass replaces this function with a call to our runtime library. The arguments of this intrinsic will capture all of the Rust-specific type information that we need to instrument LLVM IR for detecting aliasing violations. All other run-time checks are inserted in the backend by the LLVM pass. 

Our runtime library is implemented as two components. The first component is implemented in C++ using the LLVM Sanitizer API. This component redirects our runtime checks to a *Rust* runtime library, where we implement the semantics of Rust's aliasing model and all necessary operations for initializing and accessing values in shadow memory. This design allows us to reuse Miri's implementation of Tree Borrows. It will also make it easier to port our approach to other Rust backends, since developers will only need to reimplement our instrumentation pass. Eventually, we will distribute these libraries as LLVM IR, which our Cargo plugin will statically link against the source program using Cross-Language LTO. This will allows us to gain performance through inlining. They are dynamically linked for now, though.

## Configuration (`config.toml`)

Building BorrowSanitizer from source requires setting the following configuration options:several configuration options. 
```
[llvm]
download-ci-llvm = false
clang = true
```
This ensures that LLVM is built from source so that you have access to our instrumentation pass. This also builds a copy of Clang, which has been modified to accept `borrow` as a sanitizer. Our runtime library cannot be built from Stage 0 because it depends on Miri, which requires unstable features. This may change in the future if we determine that we need our runtime library to be `no_std`. For distribution builds, you can reuse Rust's default configuration.

We provide a [sample configuration file](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.bsan.dev.toml) for you to modify. If you are creating a distribution build, then you should reuse [Rust's default configuration](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.dist.toml).

## Testing & Contributing
We have unit tests for our runtime library and UI tests for the entire toolchain. Our unit tests are within `bsanrt`, and our UI tests are within `bsan-driver`. You can execute each test suite by passing the relative path to these components from the root directory as an argument to `./x.py test`. 

For example, this command executes our unit tests...
```
./x.py test src/tools/bsan/bsanrt
```
...while this one executes our integration tests:
```
./x.py test src/tools/bsan/bsan-driver
```
Our CI pipeline checks for formatting, and then it builds and tests our toolchain on each of our supported architectures. Make sure to run `./x.py fmt` and execute each of our test cases before opening a pull request. All pull requests should be made using the branch `BorrowSanitizer/bsan` as the base for comparison. When using GitHub, be careful to select the correct branch, since the interface defaults to submitting a pull request against the main branch of the Rust toolchain! 

All pull requests must pass our CI pipline. External pull requests must be reviewed by a member of the team. 
