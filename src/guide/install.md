# Installing BorrowSanitizer


## Requirements
Currently, we only support the following compilation targets:

|         **target**        | **description** |
|-------------------------|---------------|
| `aarch64-apple-darwin` |   ARM64 macOS (M-series)   |
|  `x86_64-unknown-linux-gnu` |    X86 Linux    |

You also need to have [installed Rust](https://www.rust-lang.org/tools/install).

## Installation

For the moment, the only way to try BorrowSanitizer is to build [our fork of Rust](https://github.com/BorrowSanitizer/rust) from source.
```
git clone https://github.com/BorrowSanitizer/rust.git && cd rust
```
Then, build our development toolchain.
```
./x.py build --config ./src/bootstrap/config.bsan.dev.toml
```
Our [development configuration file](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.bsan.dev.toml) provides the minimal settings required for building BorrowSanitizer.

Then, link our toolchain and set it as the default.
```
rustup toolchain link bsan build/host/stage1 
rustup default bsan
```
You can enable BorrowSanitizer by passing the flag `-Zsanitizer=borrow` to Cargo, like so:
```
RUSTFLAGS='-Zsanitizer=borrow' cargo ...
```
