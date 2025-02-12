

## Faster LLVM Builds
It takes a while to rebuild our LLVM instrumentation pass in our default build configuration, 
since changing LLVM triggers a partial rebuild of the Rust compiler.
We recommend using the following alternate build steps if you plan on making changes to our backend pass. 
This will build the LLVM toolchain from source as a dynamically linked library,
allowing you to make changes to the backend without having to recompile the frontend.

First, modify `config.toml` and enable `llvm.link-shared`. This ensures that LLVM will be built as a dynamically linked library.
```
[llvm]
link-shared=true
```
This option is already enabled in our development configuration file. Next, build LLVM from source. 
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

