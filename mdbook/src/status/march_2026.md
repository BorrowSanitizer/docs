# Status Update - March 2026
#### By Ian McCormack, Molly MacLaren, and Joshua Sunshine

We are building BorrowSanitizer: an LLVM-based instrumentation tool for finding violations of Rust's aliasing model in multilanguage applications. If you are new to the project, then we recommend checking out the [introduction](https://borrowsanitizer.com/intro.html) and our [first status update](https://borrowsanitizer.com/status/january_2026.html) before continuing. 

This month, we implemented a variety of new features that helped us build a systematic understanding of the types of programs that BorrowSanitizer can support. To do this, we:
1. Ran BorrowSanitizer on hundreds of relevant tests from Miri’s test suite.
2. Improved tooling support for multilanguage programs.

There’s also some exciting news: our MCP was seconded, so we’re getting closer to having our [retag intrinsics](https://github.com/rust-lang/compiler-team/issues/958) upstream.

## Miri's Test Suite
Prior to March, we were using 51 of Miri's tests for Stacked and Tree Borrows. This month, we expanded that number to 323 tests. This required more effort than a simple copy + paste, since not every test is relevant to our goals. Aliasing violations and memory safety errors are only a subset of the kinds of undefined behavior that Miri can find. If we include tests with errors that BorrowSanitizer cannot detect, then we will have no way to know whether these tests are behaving correctly.

To find appropriate test cases, we started with the 860 tests in Miri's [`pass`](https://github.com/rust-lang/miri/tree/master/tests/pass) and [`fail`](https://github.com/rust-lang/miri/tree/master/tests/fail) directories. We excluded tests that panic, rely on third-party dependencies, or use Miri-specific features. We will expand our testing to include these other categories eventually. For now, we feel that this subset gives us the best signal as to what's working and what remains to be fixed.

We executed each of these tests in Miri under a configuration that is similar to BorrowSanitizer. That is, we enabled Tree Borrows and disabled other forms of checking. There were 405 test cases that exited with "0" or terminated with a kind of undefined behavior that BorrowSanitizer should be able to detect. At the moment, 80% of these tests have the correct behvavior in BorrowSanitizer:

![80% of relevant test cases have the correct behavior in BorrowSanitizer](images/march_2026.png =300x center)

The remaining 20% of tests are failing for a variety of reasons, but at least 27 of them involve Miri's "wildcard" provenance semantics. When a pointer is created from an integer, Miri assigns it a "wildcard" provenance value. In the "permissive provenance" mode, Miri validates accesses through wildcard pointers using a global mapping between addresses and memory allocations. BorrowSanitizer does not have a similar mapping; at the moment, we ignore the effects of accesses with wildcard pointers. We plan to match Miri's capabilities here eventually. 

For more information, you can check out our [filtering script](https://github.com/BorrowSanitizer/bsan/blob/main/tests/miri-tests/filter_miri_tests.py) and the [raw dataset](https://github.com/BorrowSanitizer/updates/blob/main/mar26/miri.csv).

## Multilanguage Libraries
Back in 2023, we extended Miri to execute foreign functions by communicating with an LLVM interpreter. Our prototype never made it upstream, but it was effective: we found 46 instances of undefined or undesired behavior in 37 libraries. The first bug that we found using our extension was an aliasing violation in bzip2: a popular compression library. Now, we can recreate this bug using BorrowSanitizer!
```
error: Undefined Behavior: read through <1067>(unprotected) at alloc453[0x8] is forbidden
   --> bzip2-sys-0.1.13+1.0.8/bzip2-1.0.8/bzlib.c:454:45
    |
454 | if (s->avail_in_expect != s->strm->avail_in)
    |
    = help: the accessed tag <1067>(unprotected) has state Disabled which forbids this read
help: the accessed tag <1067>(unprotected) was created here, in the initial state Reserved
   --> bzip2-0.6.1/src/mem.rs:123:41
    |
123 | ffi::BZ2_bzCompressInit(&mut *raw, lvl.level() as c_int, 0, work_factor as c_int),
    |
help: the accessed tag <1067>(unprotected) later transitioned due to a foreign write
   --> bzip2-0.6.1/src/mem.rs:149:9
    |
149 | self.inner.raw.avail_in = input.len().min(c_uint::MAX as usize) as c_uint;
    |
help: this transition corresponds to a loss of read and write permissions

stack backtrace:
0: BZ2_bzCompress
      at bzip2-sys-0.1.13+1.0.8/bzip2-1.0.8/bzlib.c:454:45
1: <bzip2::mem::Compress>::compress_inner
      at bzip2-0.6.1/src/mem.rs:153:19
2: <bzip2::mem::Compress>::compress
      at bzip2-0.6.1/src/mem.rs:177:23
```
We added the bzip2 bug to our test suite, and we will continue to import the rest of the bugs we found in our [earlier evaluation](https://dl.acm.org/doi/10.1109/ICSE55347.2025.00167) of multilanguage libraries. This bug has already [been reported](https://github.com/trifectatechfoundation/bzip2-rs/issues/94).

BorrowSanitizer's Cargo plugin automatically configures compatible versions of `rustc`, `clang`, and `lld` to run our LLVM pass and link in our runtime library. No manual configuration is required, aside from ensuring that C/C++ code is statically linked and installing the correct version of each tool. Most multilanguage crates either statically link by default or provide a feature to enable. 

## What’s next?
BorrowSanitizer has two key components. The LLVM component includes our instrumentation pass and the outer "shell" of our runtime library, which is responsible for shadow memory management and interception. This should support a "no-op" mode that we can use to test BorrowSanitizer entirely within the LLVM source tree. The second component is a Rust library that implements the semantics of Tree Borrows. When this library is statically linked alongside our LLVM component, it overrides weak symbols to enable our run-time checks. 

Our goal in April is to finish an initial version of the LLVM component. This will come with an RFC posted to the LLVM community, which will propose upstreaming our "no-op" mode. Thanks to [@obraunsdorf](https://github.com/obraunsdorf), we have a strategy for supporting atomic pointer operations, which was one of the last points of uncertainty in the design of the LLVM component. The remaining features that we have left to implement are necessary to support garbage collection. In addition to our shadow heap, we also need to be able to identify where each pointer's provenance value is stored on the stack to be able to prune unreachable nodes from each tree.

That’s all until next month. In the meantime, you can reach us on [Zulip](https://bsan.zulipchat.com/).