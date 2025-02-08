# Usage
You can enable BorrowSanitizer by passing the flag `-Zsanitizer=borrow` to the Rust compiler, like so: 
```
RUSTFLAGS="-Zsanitizer=borrow" cargo build -Zbuild-std --target <host>
```
Replace `<host>` with your current target. You can find out what this is by checking in the `build` folder or executing the following command:
```
rustc -Vv | awk '/host:/ {print $2}'
```
The distribution build of our toolchain includes a Cargo plugin that will handle this step automatically. The following command is equivalent to the one above.
```
cargo bsan build
```
Our plugin also creates and links against an instrumented sysroot so that you can avoid rebuilding the standard library when switching between projects.