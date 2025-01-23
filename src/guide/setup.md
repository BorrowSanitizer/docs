# Setup

At the moment, the only way to try BorrowSanitizer is to build our custom Rust toolchain from source. We support the following compilation targets:

|         **target**        | **description** |
|-------------------------|---------------|
| `aarch64-apple-darwin` |   ARM64 macOS (M-series)   |
|  `x86_64-unknown-linux-gnu` |    X86 Linux    |∂

## Development Builds
Our fork of the Rust toolchain is [`BorrowSanitizer/rust`](https://github.com/BorrowSanitizer/rust). We also maintain a fork of the LLVM project at [`BorrowSanitizer/llvm-project`](), which replaces the LLVM submodule in our Rust fork. 

Begin by cloning our Rust fork.
```
git clone https://github.com/rust-lang/rust.git && cd rust
```
Then, build our development toolchain.
```
./bbsan
```
This command will create a toolchain named `bsan-stage1` that is linked into the `build/stage1` directory. It will also copy all tool binaries into the "bin" folder of the installation root (`$HOME/.cargo/bin/`) so that commands like `cargo bsan` work correctly.  

We recommend using this script *instead* of `x.py`, since it will speed up subsequent builds by skipping stages that have already been completed. If you use `x.py`, then your `config.toml` must contain (at minimum) the settings in our [development configuration file](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.bsan.dev.toml).