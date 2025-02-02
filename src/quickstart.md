# Quickstart

Currently, the only way to try BorrowSanitizer is to build [our fork of Rust](https://github.com/BorrowSanitizer/rust) from source. To do this, you will need to have [installed Rust](https://www.rust-lang.org/tools/install).

Also, we only guarantee support for the following compilation targets:

|         **target**        | **description** |
|-------------------------|---------------|
| `aarch64-apple-darwin` |   ARM64 macOS (M-series)   |
|  `aarch64-unknown-linux-gnu` |    ARM64 Linux    |
|  `x86_64-unknown-linux-gnu` |    X86 Linux    |

You can try to use our tool with other platforms, but there is no guarantee that it will work as expected, since we are only developing and testing it for these targets for the time being.

To begin, clone our git repository. If you plan on contributing to BorrowSanitizer, we recommend either a full clone (without any additional CLI options), or passing `--filter='blob:none`, which will speed up the initial download by  fetching [blob objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) on-demand. Alternatively, you can pass `--depth 1` for a shallow clone, but this is only useful for building 

```
git clone  https://github.com/BorrowSanitizer/rust.git 
cd rust
```

> Visit the [Rust Compiler Development Guide](https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html#how-to-build-and-run-the-compiler) for additional information on how to build the Rust compiler. 

Then, make a copy of our [development configuration file](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.bsan.dev.toml) and place it into the the root directory. This file provides the minimal settings required for building BorrowSanitizer.
```
cp ./src/bootstrap/defaults/config.bsan.dev.toml config.toml
```
Execute this next command to create a Stage 1 build of our development toolchain (without optimizations). This will take 1-2 hours to complete (depending on your hardware) since we need to build the LLVM backend from source to access our instrumentation pass and runtime. 
```
./x.py build
```
Link our toolchain and set it as the default.
```
rustup toolchain link bsan build/host/stage1 
rustup default bsan
```