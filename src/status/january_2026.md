# Status Update - January 2026
Rust provides static safety guarantees by restricting aliasing and mutability. Rust's borrow checker enforces these rules for safe code, but programs that use Rust's "unsafe" operations can easily break them, causing subtle safety errors.

Until now, the primary way to find these bugs was Miri, an interpreter. Miri is precise, and it's the only tool that can fully validate programs against Rust's latest [Tree Borrows](https://dl.acm.org/doi/10.1145/3735592) aliasing model. However, it is also quite slow, and it cannot "see" into C and C++ code to detect aliasing bugs that span language boundaries.

BorrowSanitizer is our solution: an LLVM-based sanitizer that instruments programs during compilation. It's like [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html), but for checking programs for Tree Borrows violations. It’s designed to be fast enough for fuzzing and capable of tracking pointer "provenance" metadata—even when code crosses over into C or C++.

## Where we stand in January 2026
We are transitioning from a research prototype to a functional tool. While the core logic is sound, we are still focused on three major engineering challenges: integrating with the Rust compiler, making the runtime fast enough for production use, and tracking pointer metadata in multithreaded programs. We have been testing BorrowSanitizer using the single-threaded programs from [a subset](https://github.com/BorrowSanitizer/bsan/tree/main/tests/miri-tests) of Miri's test suite that deals with aliasing violations, and we have similar outcomes; tests either pass or report an error in both Miri and BorrowSanitizer. Once these features are merged, we will be able to expand our support to the rest of Miri's tests, and we will begin auditing failing tests to make sure that we are finding the same errors as Miri. We will begin automated testing and benchmarking on real-world libraries, and you can expect to see benchmarking results in our February update.

We discuss these points in more detail below. We will continue to post monthly updates here and in our [project goal](https://rust-lang.github.io/rust-project-goals/2025h2/codegen_retags.html).

### Compiler Changes 

Retags are the central mechanism of Tree Borrows. Broadly, when a reference is created, assigned, or passed into a function, its underlying pointer is "retagged". This creates a new permission that's recorded in a tree of permissions associated with the allocation being accessed. Miri uses a combination of type information and explicit [`Retag`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.StatementKind.html#variant.Retag) instructions to determine when a retag needs to happen, but all of this information is lost at the LLVM level. Ordinarily, this would prevent us from distinguishing between raw pointers and references, and we need to do this to be able to find aliasing violations.

We modified the Rust compiler to emit retags as function calls. If we have the following MIR:
```
x = &mut y;
```
We emit the following function call:
```
%2 = @__retag(ptr %1, ...)
```
We use [a fork](https://github.com/BorrowSanitizer/rust/commits/main/) of the compiler with a prototype of these changes in our current implementation of BorrowSanitizer.

In December, we posted [a pre-RFC](https://internals.rust-lang.org/t/pre-rfc-emit-retags-in-codegen/23706) describing these changes, which was adapted into the [MCP](https://github.com/rust-lang/compiler-team/issues/958) that we posted earlier this week. Our current prototype depends on MIR [`Retag`](https://doc.rust-lang.org/beta/nightly-rustc/rustc_middle/mir/enum.StatementKind.html#variant.Retag) instructions to determine where to emit retags during codegen, but these instructions are [likely going away](https://github.com/rust-lang/unsafe-code-guidelines/issues/371#issuecomment-3346404729), so the final version of our changes will determine where to insert these instructions during codegen. We expect these changes to be relatively straightforward, so barring any unexpected concerns, we should be able to merge our changes before the start of the second project goal period in April.

Once these changes are merged, the rest of development can continue out-of-tree. However, there's another relevant proposal that would be helpful to have implemented to support BorrowSanitizer. Last September, an [MCP](https://github.com/rust-lang/compiler-team/issues/907) was accepted for including `clang` as a Rust component. Our instrumentation pass relies on the distribution of LLVM that's bundled with nightly, so we need `clang` to have a matching version to be able to instrument C and C++ programs. Having `clang` as part of the user's sysroot would make this far easier. 

### Remaining features
Three critical features have yet to be finished before we can expand our testing to the rest of Miri's test suite and more real-world crates. The following list is not comprehensive—there are other features that we have yet to support, such as thread-local storage, global variables, and constant allocations. However, we expect that these will be straightforward extensions of existing functionality, and not core features like the following:

#### Error Reporting
When Miri finds an aliasing violation, it reports both the primary location where the error occurred and the secondary locations of memory accesses that indirectly caused the error. It's relatively straightforward to provide a backtrace for the primary location, and we can record the instruction pointers for the secondary locations within the nodes of the tree. However, we have not implemented a method for translating these instruction pointers into source locations. We are currently adding support for using [`llvm-symbolizer`](https://llvm.org/docs/CommandGuide/llvm-symbolizer.html) to handle this, and once it is finished, we will introduce nicer error messages so that we can start auditing failing test cases.

#### Garbage Collection
Periodically, Miri "stops the world" and collects all of the permissions associated with pointers stored in memory. Then, it visits the stack or tree for each allocation and removes permissions that are no longer reachable in memory. This can [dramatically improve performance](https://github.com/rust-lang/miri/pull/2479).

Like Miri, we can scan our shadow memory spaces for provenance values. However, instead of halting execution, BorrowSanitizer's approach will continue to allow threads to execute. A thread will only be blocked when it attempts to update shadow memory. All other operations, including certain run-time checks, will still be able to continue concurrent to garbage collection.

We can already scan through our "shadow heap", which contains provenance values that are stored to memory. However, we still need a way to track which provenance values are accessible on the stack. At the moment, we load provenance into [virtual registers](https://llvm.org/docs/CodeGenerator.html#register-allocator), just like any other LLVM value. This is convenient to implement, but it prevents us from knowing where provenance is stored at run-time. At the moment, we are switching to using a "shadow stack" to store provenance values. Garbage collection will block any operations that update the shadow stack pointer.

#### Thread-Safe Pointer Updates
Each pointer has a provenance value, which identifies its permission to access memory. Each provenance value is three words: an allocation ID, a borrow tag, and a pointer to an allocation metadata object. Every regular load or store of a pointer requires three additional loads: one for each component of its provenance. These accesses are subject to data races, but that's acceptable, since any race on a provenance value would also be a race on the pointer's value. That would be a problem for [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) to detect.

We cannot use the same approach for atomic pointer operations. If we atomically stored each component of a provenance value in succession, then this would not guarantee that all three values would be read consistently as a unit. Our instrumentation would be introducing a data race that does not exist in the original program. When provenance values are read in an inconsistent state, this all but guarantees a false positive error.

We have two options to address this. The easiest approach would be to block shadow memory on atomic accesses. This is expensive, but we can mitigate it somewhat by only blocking the region of memory that is being accessed. Another option would be to remove allocation IDs from our provenance values altogether and use the allocation info pointer as an ID instead. In that case, provenance values would be 128 bits, which is small enough for atomic loads and stores. Support for atomic operations on 128-bit integers is inconsistent but improving, and it could be worth the performance increase. 

### Evaluation
We do not have a clear idea of what performance looks like yet—we've been focused on feature implementation over benchmarking. We are slower than Miri for certain programs at the moment. We can attribute this to our lack of garbage collection and our eagerness in locking global state to validate accesses. Expect to see initial benchmark results in our February update. Additionally, one of our team members—[@Gitter499](https://github.com/Gitter499)—has been working on a dashboard to display performance results that we can integrate into CI. We also plan on running an instance of [Crater](https://github.com/rust-lang/crater) once we figure out a configuration that works with our driver. 

## Conclusion
Let us know if you have any questions! You can reach us [on Zulip](https://bsan.zulipchat.com/). Otherwise, you'll hear from us again in February for our next update.

We thank Google and the Rust Foundation for funding this project, as well as [@RalfJung](https://github.com/ralfjung) and [Tyler Mandry](https://github.com/tmandry) for their time and effort in providing feedback.
