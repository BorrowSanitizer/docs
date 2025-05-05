# Setup
Currently, the only way to try BorrowSanitizer is to build [our fork of Rust](https://github.com/BorrowSanitizer/rust) from source. 
We only guarantee support for the following compilation targets:

|         **target**        | **description** |
|-------------------------|---------------|
| `aarch64-apple-darwin` |   ARM64 macOS (M-series)   |
|  `x86_64-unknown-linux-gnu` |    X86 Linux    |

You can try to use our tool with other platforms, but there is no guarantee that it will work as expected, since we are only developing and testing it for these targets at the moment.

## Building from Source

You will need the following dependencies:
* [Rustup](https://www.rust-lang.org/tools/install)
* [Ninja](https://github.com/ninja-build/ninja/wiki/Pre-built-Ninja-packages)

Start by cloning our git repository. If you plan on contributing to BorrowSanitizer, we recommend either a full clone (without any additional flags) or passing `--filter='blob:none'`, which will speed up the initial download by fetching [blob objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) on-demand. Alternatively, you can pass `--depth 1` for a shallow clone, but you'll need to switch to a full clone if you want to contribute. 
```
git clone https://github.com/BorrowSanitizer/rust.git
cd rust
```

## Production Builds
If you want to install our toolchain, or if you do not plan on contributing, then use the default distribution settings.
```
./x.py setup dist
```
Next, create a directory for the installation and set the variable `DESTDIR` to its absolute path.
```
mkdir [path] && export DESTDIR=[path]
```
Then, build and install the toolchain. This will copy the distribution binaries into `$DESTDIR`.
```
./x.py build && ./x.py install
```
When the build is finished, link the toolchain and set it as the default. 
```
rustup toolchain link bsan $DESTDIR 
rustup default bsan
```

## Development Builds
If you plan on contributing to BorrowSanitizer, you should copy our development [config file](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.bsan.dev.toml) into the root directory.
```
cp ./src/bootstrap/defaults/config.bsan.dev.toml config.toml
```
Then, build our toolchain.
```
./x.py build
```
When the build is complete, link the toolchain and set it as the default. 
```
rustup toolchain link bsan build/host/stage1 
rustup default bsan
```
> Visit the [Rust Compiler Development Guide](https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html#how-to-build-and-run-the-compiler) for additional tips and tricks.

## Usage
You can manually enable BorrowSanitizer by passing the flag `-Zsanitizer=borrow` to the Rust compiler, like so: 
```
RUSTFLAGS="-Zsanitizer=borrow" cargo build
```
Replace `<host>` with your current target. You can find out what this is executing the command `rustc -vV`. If a crate has doctests, then you will need to enable BorrowSanitizer within *both* `RUSTFLAGS` and `RUSTDOCFLAGS` for it to compile correctly.

The distribution build of our toolchain includes a Cargo plugin that will handle this step automatically.
```
cargo bsan <build/test/run>
```
Our plugin also links Rust programs against an instrumented sysroot, so that you can avoid rebuilding the standard library when switching between projects.