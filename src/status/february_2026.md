# Status Update - February 2026

## Where we stand in February 2026

#### Error Reporting

When BorrowSanitizer detects aliasing violations, diagnostic `help` messages now include snippets of the original source lines.
This moves us closer to consistent error message styling with the format used by Miri and the Rust compiler.

For example, [miri-tests/fail/both_borrows/alias_through_mutation.rs](https://github.com/BorrowSanitizer/bsan/blob/main/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs) contains:
```
help: the accessed tag <649>(unprotected) was created here, in the initial state Frozen
      --> bsan/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs:6:14
         |
       6 | *x = &mut *(target as *mut _);
         |

    help: the accessed tag <649>(unprotected) later transitioned due to a foreign write
      --> bsan/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs:15:5
         |
      15 | *target = 13;
         |
```

#### LLVM Upstreaming

For BorrowSanitizer to work upstream, we need it to be able to build and run it entirely within the LLVM source tree without our "core" Rust runtime in bsan-rt. We've added duplicate no-op definitions for our API function within the llvm-wrapper component annotated with the `weak` sanitizer.

#### Profiling Miri and BorrowSanitizer

While Miri is Rust's flagship unsafe sanitizer, programmers have reported that it can run upwards of 1000x slower than native execution (just Google "miri slow"). Extensive work has not yet gone into identifying Miri's overheads and areas for optimization, which would also impact and inform the design of BorrowSanitizer.

We have begun work on profiling Miri's runtime on test cases of the most used crates with [Perf](https://perfwiki.github.io/main/), two of Valgrind's built-in profilers—[DHAT](https://valgrind.org/docs/manual/dh-manual.html) and [Callgrind](https://valgrind.org/docs/manual/cl-manual.html)—and by extending Miri's existing tracing macros. The profiler data allows us to track time spent within Miri functions, memory usage, and instruction calls, while tracing logs can provide aliasing model-specific metrics such as number of memory allocations tracked and nodes visited.

We are currently focused on comparing different Miri configurations (such as stacked borrows vs tree borrows) but plan on eventually profiling BorrowSanitizer.