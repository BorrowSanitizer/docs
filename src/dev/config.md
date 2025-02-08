# Configuration
Building BorrowSanitizer from source requires setting the following configuration options:
```
[llvm]
download-ci-llvm = false
clang = true
```
This ensures that LLVM is built from source so that you have access to our instrumentation pass. This also builds a copy of Clang, which has been modified to accept `borrow` as a sanitizer. Our runtime library cannot be built from Stage 0 because it depends on Miri, which requires unstable features. This may change in the future if we determine that we need our runtime library to be `no_std`. 

We provide a [sample configuration file](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.bsan.dev.toml) for you to modify. If you are creating a distribution build, then you should reuse [Rust's default configuration](https://github.com/BorrowSanitizer/rust/blob/bsan/src/bootstrap/defaults/config.dist.toml).

## Faster LLVM Builds
It takes a while to rebuild our LLVM instrumentation pass in our default build configuration, since changing LLVM triggers a partial rebuild of the Rust compiler at each stage. We recommend using the following alternate build steps if you plan on making changes to our backend pass. This will build the LLVM toolchain from source as a dynamically linked library and then reuse this configuration for subsequent builds, making it easy to make changes to the backend without having to recompile the frontend.

First, , modify `config.toml` to set`llvm.link-shared` set to `true`. This ensures that LLVM will be built as a dynamically linked library. This option has already been set by default in our development configuration. 
```
[llvm]
...
link-shared=true
```
Now, build the full Rust toolchain
```
./x.py build
```
Once this has completed, you will be able to use the following series of commands to rebuild LLVM quicky *without* having to rebuild parts of the Rust compiler. 
```
rm ./build/host/llvm/llvm-finished-building
./x.py build --keep-stage 0 --keep-stage-std 0 --keep-stage 1 --keep-stage-std 1
```

<div class="warning">
Be extra careful when keeping prior compilation stages. This will ignore any changes made to the compiler or standard library. You will need to rebuild these components whenever you pull changes from upstream or make any changes to the Rust source yourself.
</div>