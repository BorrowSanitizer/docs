# Implementation
BSan consists of the following components:

| Name                       | Location | Purpose |
|---------------------------------|----------|---------|
| UI (🚧)                     | [Rust](https://github.com/borrow-sanitizer/rust/tree/bsan/src/tools/bsan) | Implementation of the `cargo bsan` CLI utility. Invokes rustc with the configuration flags required for BSAN and initializes any other components necessary for large-scale data collection and evaluation. |
| Borrow Tracking Dynamic Library | [Rust](https://github.com/borrow-sanitizer/rust/tree/bsan/src/tools/bsan/borrowtracker) | The core run-time library, which tracks the size and liveness of allocations, as well as the permissions associated with pointers under Rust's aliasing models. |
| Static Analysis Pass (🚧)   | Rust | The static analysis pass for BSan, which removes unnecessary run-time checks. |
| Sanitizer Dynamic Library       | [LLVM](https://github.com/borrow-sanitizer/llvm-project/tree/3d8a627fc2c651e0cb65f0e48f25cf341986176c/compiler-rt/lib/bsan) | The BSAN run-time checking interface. Mostly acts as FFI glue code, since the borrow tracking module handles all permission and bounds tracking. |
| Sanitizer Instrumentation Pass  | [LLVM](https://github.com/borrow-sanitizer/llvm-project/blob/3d8a627fc2c651e0cb65f0e48f25cf341986176c/llvm/lib/Transforms/Instrumentation/BorrowSanitizer.cpp) | Inserts run-time checks as calls into the sanitizer dynamic library, which in turn calls into the borrow tracker. |


