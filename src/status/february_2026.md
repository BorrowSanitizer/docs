# Status Update - February 2026
#### By Ian McCormack and Molly MacLaren

We are building BorrowSanitizer: an LLVM-based instrumentation tool for finding violations of Rust's aliasing model in multilanguage applications. If you are new to the project, then we recommend checking out our [introduction](https://borrowsanitizer.com/intro.html) and our [first status update](https://borrowsanitizer.com/status/january_2026.html) before you continue. 

February has been *busy*! This month, we:

* Implemented detailed error messages.
* Gained a better understanding of our performance.
* Improved our integration with LLVM.

We also proposed [a 2026 Rust Project Goal](https://github.com/rust-lang/rust-project-goals/blob/main/src/2026/borrowsanitizer.md) for transitioning BorrowSanitizer from a research prototype into a usable tool.

### Error Messages
When BorrowSanitizer detects an aliasing violation, its error messages now include snippets of the source code that triggered the error. This moves us closer to error message styling consistent with the format used by Miri and the Rust compiler. For example, here's what BorrowSanitizer prints when it detects an error in one of Miri's [test cases](https://github.com/BorrowSanitizer/bsan/blob/main/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs).
```
error: Undefined Behavior: read through <TAG>(unprotected) at ALLOC[0x0] is forbidden
help: the accessed tag <TAG>(unprotected) has state Disabled which forbids this read
    help: the accessed tag <TAG>(unprotected) was created here, in the initial state Frozen
      --> bsan/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs:6:14
         |
       6 | *x = &mut *(target as *mut _);
         |

    help: the accessed tag <TAG>(unprotected) later transitioned due to a foreign write
      --> bsan/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs:15:5
         |
      15 | *target = 13;
         |

    help: this transition corresponds to a loss of read permissions
backtrace:
  0:
    bsan/tests/miri-tests/fail/both_borrows/alias_through_mutation.rs:16:16
  1:
    RUSTLIB/core/src/ops/function.rs:250:5
  2:
    RUSTLIB/std/src/sys/backtrace.rs:166:18
```

### Performance
We want BorrowSanitizer to provide both better FFI support and better performance than Miri. We have plenty of [anecdotal evidence](https://github.com/rust-lang/miri/issues/654) that Miri can run upwards of 1000x slower than native execution in certain situations. However, we are still missing the kind of detailed evaluation that we need to identify the root causes of Miri's overhead and potential areas for optimization. This will help inform our design for BorrowSanitizer, which extends Miri's existing Tree Borrows implementation. 

We are collaborating with [Shinhae Kim](https://shinhae-kim.github.io/) from Cornell to profile Miri on test cases from the most popular crates. He's advised by [Saikat Dutta](https://www.cs.cornell.edu/~saikatd/) and [Owolabi Legunsen](https://www.cs.cornell.edu/~legunsen/). We are using [Perf](https://perfwiki.github.io/main/), two of Valgrind's built-in profilers ([DHAT](https://valgrind.org/docs/manual/dh-manual.html) and [Callgrind](https://valgrind.org/docs/manual/cl-manual.html)), and Miri's existing tracing macros. Profiling data allows us to track the time that we spend within each of Miri's functions, its memory usage, and the number of instructions executed. Miri's tracing logs can provide aliasing model-specific metrics, such as the number of memory allocations tracked and the number of nodes visited within the tree for each allocation. We are currently focused on evaluating Miri under different configurations, but we will be able to add BorrowSanitizer to this benchmarking pipeline in the future.

We were also able to shrink BorrowSanitizer's provenance metadata, which should help improve performance. At run-time, we originally associated every pointer with an allocation ID, a "borrow tag" identifying its access permission, and a pointer to a metadata object. This took up three words, which is too large for atomic memory accesses. We were able to remove the allocation ID without affecting precision. This will make provenance small enough to load and store with 128-bit atomic operations. 

In our initial testing, we noticed that BorrowSanitizer ran significantly slower than Miri for an empty crate, without any tests. Any runtime overhead had to be coming from Rust's [libtest](https://github.com/rust-lang/rust/tree/main/library/test) harness, or the prelude. Our logs indicated that BorrowSanitizer was emitting nearly *four times* as many retags as Miri for the same program. At first, we thought that this was a bug. It seemed like we should have had strictly fewer retags than Miri, since we skip retagging after plain assignments. However, many of the functions that triggered the majority of BorrowSanitizer's "extra" retags were not being executed by Miri. It turns out that libtest has a ["fast path"](https://github.com/rust-lang/rust/blob/eeb94be79adc9df7a09ad0b2421f16e60e6d932c/library/test/src/term/terminfo/mod.rs#L80) for Miri:
```rust
/// Creates a TermInfo for the named terminal.
pub(crate) fn from_name(name:  &str) ->  Result<TermInfo, Error> {
	if cfg!(miri) {
    	// Avoid all the work of parsing the terminfo (it's pretty slow under Miri), and just
    	// assume that the standard color codes work (like e.g. the 'colored' crate).
    	return  Ok(TermInfo { .. });
	}
	...
}
```
There are also certain features (like [signal handlers](https://github.com/rust-lang/rust/blob/eeb94be79adc9df7a09ad0b2421f16e60e6d932c/library/std/src/sys/pal/unix/stack_overflow.rs#L43)) that Miri does not support, but we do. We are not able to use `--cfg=miri` to get around this, since we cannot guarantee that Miri's configuration will work for native instrumentation, or vice versa. This makes it tricky to have a perfectly fair comparison between the tools.

We do not have enough data to be confident about BorrowSanitizer's performance overhead yet. However, there's an interesting test case from [`hashbrown`](https://github.com/rust-lang/hashbrown) that is still worth taking a look at. The unit test [`test_lots_of_insertions`](https://github.com/rust-lang/hashbrown/blob/a41940ac86f6d08396594e4c062b4039dcb6099f/src/map.rs#L5326) is notoriously slow in Miri; it's even disabled by default, using `#[cfg(not(miri))]`. Broadly, this test does exactly what it says: it inserts and removes lots of entries from a single `HashMap` instance. [Krit](https://github.com/kritdass), one of our collaborators, created [a variant](https://github.com/BorrowSanitizer/bsan/blob/main/tests/pass-dep/lots_of_insertions.rs) of this test that lets us control the number of entries (N) being inserted on each iteration of its inner loop. The original test from `hashbrown` uses N=1000. Here are results for N=1 using BorrowSanitizer, AddressSanitizer, ThreadSanitizer, and both debug and release builds. Measurements are displayed as the average of 100 iterations. 

![Average execution time for Miri (2.06 sec), BorrowSanitizer (1.26 sec), AddressSanitizer (0.005 sec), ThreadSanitizer (0.01 sec) and both debug (0.0006 sec) and release (0.0004 sec) builds for `test_lots_of_insertions`.](images/february_2026_all.png =500x center)

We disabled most of Miri's checking and nondeterminism, enabled Tree Borrows, and disabled the garbage collector, since BorrowSanitizer does not support garbage collection yet. You can double-check our benchmarking configuration and raw dataset [here](https://github.com/BorrowSanitizer/feb-update). This is not an entirely fair comparison. In addition to the limitations surrounding Miri's compilation directives, our timing measurements include the invocation of the cargo plugins for both Miri and BorrowSanitizer, which work differently. BorrowSanitizer does not instrument global, thread-local, or constant allocations. Since this is a single test case, our results are not representative of how each of these tools performs in every situation. That said, BorrowSanitizer does appear to be significantly faster than Miri (note the logarithmic scale), but both tools are in a similar category of performance here. We have also noticed that BorrowSanitizer tends to be a bit noisier than Miri, but we are not sure why.

![Average and standard deviation execution time for BorrowSanitizer (1.26 sec, 0.043 sec) and Miri (2.06 sec, 0.015 sec).](images/february_2026_mean_stdev.png =500x center)

These results indicate that BorrowSanitizer's overall performance is still quite dependent on the performance of Miri's Tree Borrows implementation. However, BorrowSanitizer still has plenty of room for optimization, since we have mostly focused on feature support up to this point.

### LLVM Integration
We have a tentative plan for upstreaming BorrowSanitizer this year. We also improved our integration with the LLVM toolchain by overhauling our "retag" intrinsics, which tell us when new references are created at the LLVM level.

#### Upstreaming
We will start the work required for upstreaming an experimental version of BorrowSanitizer this year as part of our Project Goal. This process will need to begin with the parts of the tool that have to live inside LLVM.  This means that we need to be able to test BorrowSanitizer entirely within the LLVM project's source tree, which disallows any Rust dependencies. To support this, we are splitting our runtime into two components: an LLVM component and a Rust component. The LLVM component will define BorrowSanitizer's outermost API. It also implements the parts of shadow memory management and error reporting that rely on LLVM's [`compiler-rt`](https://compiler-rt.llvm.org/) libraries. On its own, the LLVM runtime will effectively be a no-op. But, when the Rust runtime component is linked in, it will override several weak symbols within the LLVM runtime, effectively "enabling" our checking. The Rust component will define everything that is specific to Rust's aliasing semantics, and we intend for it to live as part of Rust's source tree. 

We will start [the RFC process](https://llvm.org/docs/RFCProcess.html) for the LLVM component this spring, once we finish stabilizing our API. Any progress on merging the Rust component will start later this year, depending on the status of our 2026 Project Goal proposal and the requirements from our stakeholders on the Rust Team. 

#### Retag Intrinsics
BorrowSanitizer relies on inserting special "retag" intrinsic calls into programs as they are compiled to LLVM. This is the subject of our [current Project Goal](https://rust-lang.github.io/rust-project-goals/2025h2/codegen_retags.html) and our ongoing [MCP](https://github.com/rust-lang/compiler-team/issues/958).  This month, we made significant changes to what retags look like and where they are emitted, so that we can maintain compatibility with LLVM's `noalias` metadata.

We originally planned on handling every kind of retag using a single intrinsic function. Consider the following MIR statement:
```rust
x = &mut y;
```
Usually, we would not see any distinction between the variables `x` and `y` at the LLVM level. Both would be collapsed into a single SSA variable (or "virtual register" in LLVM terminology). Our retag intrinsics preserve each alias:
```llvmir
%x = call ptr @__rust_retag(%y, ..)
```
This works great if the pointer being retagged is already available in a register.  However, sometimes we need to retag a pointer that is stored at another memory location. Consider the following function:
```rust
fn example<'a, T>(value: Foo<'a, T>) { .. }
```
The lifetime `'a` indicates that `Foo` contains a reference. Reference-type parameters receive a special "function-entry" retag. Depending on the architecture that we are compiling for, `value` might be passed to the function `example` using a pointer. Here's what the LLVM IR would look like in that case:
```llvmir
define void @example(ptr %0) { .. }
```
In this situation, we need to load the field that contains the reference to be able to retag it. After we finish the retag, we need to store the new alias back into the original location. 
```
%1 = getelementptr ... %0
%2 = load ptr %1
%3 = call ptr @__rust_retag(%2, ..)
store ptr %3, ptr %1
```
Retagging does not affect a pointer's address, but it does change its *provenance.* Every time we update the value of a pointer, we need to update its provenance metadata at the corresponding "shadow" memory location. The `store`  tells our instrumentation pass to update the provenance for the location pointed to by `%1` with the provenance of the pointer `%3`.  

However, this additional store caused issues for parameters with LLVM's `readonly` metadata annotation. In our example, if the parameter `%0` has this annotation, then all write accesses to pointers derived from `%0` are undefined behavior and will be removed during compilation—even if we disable optimizations in LLVM. 

To get around this, we had to switch to using two different forms of retag. If the pointer being retagged is available in a register, then we use `__rust_retag_reg`, which has the same semantics as the original `__rust_retag` shown above. If we need to load the pointer from memory, then we use the form `__rust_retag_mem`. For the function `example`, we would see the following LLVM IR:
```
%1 = getelementptr ... %0
call ptr @__rust_retag_mem(%1, ..)
```
Our instrumentation pass inserts the necessary `load` from `%1`, as well as all of the other shadow memory accesses that we need to update the provenance at this location. Only the provenance changes, and not the address, so we never actually need to write directly to `%1`. 

The other parameters to the retag intrinsic are the same in both of its forms.  However, we made a key change to these parameters that will make it easier for BorrowSanitizer to be merged upstream. Originally, retags had a "permission type" parameter (an `i64`) which was determined by a Rust compiler plugin.  To compile a program with BorrowSanitizer, we had to use our special `cargo-bsan` plugin, which would call a secondary `bsan-driver` plugin to inject our retag permission type values into `rustc`. We can avoid this rigmarole by passing the information that we need to determine the "permission type" value as parameters to the intrinsics. Then, we compute the permission type at run-time.

#### Retagging on Assignment 

We also had to change *where* retags are emitted, which has a subtle effect on the types of undefined behavior that BorrowSanitizer can detect. Miri retags the target of every assignment statement involving a reference—even for assignments that do not explicitly create new references. This means that the following program (from Miri's [test suite](https://github.com/rust-lang/miri/blob/master/tests/fail/both_borrows/load_invalid_shr.rs)) has undefined behavior :
```rust
let x = &mut 42;
let xraw = x as *mut _;
let xref = unsafe { &*xraw };
let xref_in_mem = Box::new(xref);

// Writing through `xraw` invalidates
// the read-only permission of `xref`
unsafe { *xraw = 42 };

let _val = *xref_in_mem;
```
The last line of this program is what triggers the error. After we load `xref` from `*xref_in_mem`, we need to retag it, which performs a read access. However, the reference `xref` was invalidated by the write through `xraw`—it no longer has the permission to access anything, so this retag is undefined behavior.

This is unexpected. We are not using `_val` for anything, and the read access for `*xref_in_mem` is completely fine. The point that triggers the error in this program is somewhat invisible, but it is still undefined behavior under both Stacked and Tree Borrows. It is [not clear yet](https://github.com/rust-lang/unsafe-code-guidelines/issues/601) whether this pattern *should* be undefined behavior, but for now, it's an error.

The `AddRetag` pass excludes certain temporary assignment statements from retagging. We originally used Rust's MIR [`Retag`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.StatementKind.html#variant.Retag) statements to determine where to emit our LLVM retags, so we were able to skip these temporary retags, too. However, it seems like Miri might be moving toward [removing retag statements](https://github.com/rust-lang/unsafe-code-guidelines/issues/371#issuecomment-3641415965). In anticipation, we decoupled our LLVM retags from Rust's MIR retags. Instead of relying on the [`AddRetag`](https://github.com/rust-lang/rust/blob/eeb94be79adc9df7a09ad0b2421f16e60e6d932c/compiler/rustc_mir_transform/src/add_retag.rs#L10) MIR pass, we decide where to emit a retag during codegen. However, the distinction between temporary and non-temporary assignments is lost once we reach codegen, which prevents BorrowSanitizer from skipping retags for temporary assignments. 

We have chosen to skip emitting retags after assignment statements. This means that BorrowSanitizer does not report an error for the program above. We still emit a retag when a reference is created and at function boundaries. So far, this does not seem to have had a significant impact on our precision, and it seems possible that Rust's aliasing model will change to accept these programs in the future. Some differences between BorrowSanitizer and Miri are to be expected, anyway, and this is one that we can live with for now.

### Conclusion
Thanks for taking a look! We'll be back in March with our next status update. In the meantime, you can reach us on [Zulip](https://bsan.zulipchat.com/).
