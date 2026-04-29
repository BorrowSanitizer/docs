# Status Update - April 2026

### By: Ian McCormack, Molly MacLaren, and Joshua Sunshine

We are building BorrowSanitizer: an LLVM-based instrumentation tool for finding violations of Rust’s aliasing model in multilanguage applications. If you are new to the project, we recommend checking out the [introduction](https://borrowsanitizer.com/intro.html) and our [first status update](https://borrowsanitizer.com/status/january_2026.html) before continuing.

This month, we focused on answering some of the remaining open design questions about how pointer metadata should be tracked in memory. We:
* Implemented a new *shadow stack* for storing pointer metadata. 
* Finished implementing our retag intrinsics.
 
We'll be sending in a few PRs for our retags to the Rust compiler in the next week or so. It will take a bit of time split our changes into more easily reviewable chunks.

The RFC for our LLVM components is taking a little longer than expected, but it was worth taking the extra time to test out compiler changes and make sure that we had the core parts of the instrumentation pass settled. We'll be drafting the RFC throughout the next few weeks.

## Shadow Arrays
Most sanitizers need a way to pass metadata alongside the arguments to each function. A naive approach would be to modify the function's signature to accept additional arguments, but this would break compatibility with uninstrumented code. Sanitizers solve this problem by storing metadata within dedicated "shadow" memory spaces that correspond to the locations where arguments and return values are typically stored. 

Both [MemorySanitizer](https://clang.llvm.org/docs/MemorySanitizer.html) and [DataFlowSanitizer](https://clang.llvm.org/docs/DataFlowSanitizer.html) use multiple thread-local arrays to pass argument metadata. When a function is called, each argument's metadata is stored within a dedicated slot inside a shadow buffer.
```C++
// Function argument shadow (for MemorySanitizer).
// Each argument starts at the next available 8-byte
// aligned address.
SANITIZER_INTERFACE_ATTRIBUTE
THREADLOCAL u64 __msan_param_tls[kMsanParamTlsSize / sizeof(u64)];
```
The callee eagerly loads the metadata onto the stack before reusing the buffer for subsequent calls. A similar process happens for return values, using a different buffer. This ensures support for uninstrumented callers by preserving interface compatibility and providing stable locations for storing metadata. 

## A Shadow Stack

Instead of the thread-local arrays used by MemorySanitizer and DataFlowSanitizer, BorrowSanitizer uses a single shadow *stack* to pass metadata across function boundaries. To pass metadata, the Caller places argument data immediately below the current shadow stack pointer. The Callee then picks up exactly where the Caller left off, reading those values in sequence. To pass data back, the Callee reuses that same stack memory to store the return value's metadata. This allows the Caller to find the return value at a predictable offset before 'popping' the stack to the next available slot.  

In addition to arguments and return values, our shadow stack stores metadata created by `allocas`, `loads`, and our [retag intrinsics](https://borrowsanitizer.com/status/january_2026.html#compiler-changes). Each of these operations allocates a shadow stack slot. Metadata for arguments and `allocas` is stored first, as part of the "frame header." All other stack slots get allocated below the header, as necessary. 

Our implementation maintains two crucial invariants:
1. We always access the shadow stack *below* the current shadow stack pointer. 
2. The shadow stack is always a contiguous array of initialized metadata values. 

The first invariant supports uninstrumented code. If a sanitized function is called by an uninstrumented program, then its shadow stack pointer might be pointing to the top of the stack, so reading above it would be an access out-of-bounds. 

We need the second invariant for garbage collection. BorrowSanitizer, Miri, and any other Tree Borrows implementation can benefit from using a garbage collector to "prune" unreachable nodes from each Tree. Miri implements a *tracing* garbage collector: periodically, it scans every value within its interpreter to remove inaccessible metadata. Miri supports this by maintaining a detailed view of where metadata is stored in memory, across both its heap and thread stacks. 

For BorrowSanitizer, instead of tracing, we plan to implement a hybrid technique called [*deferred reference counting*](https://dl.acm.org/doi/10.1145/185009.185016). In this approach, reference counting is used to track metadata associated with values on the heap, while tracing is used to track values on the stack. Periodically, one or more garbage collector threads will scan the shadow stacks of each thread to identify and destroy unreachable provenance values. For this to have maximum efficiency, we never want the garbage collector to read a stale metadata value. 



### Shadow Frames
The shadow stack also has a role in supporting the semantics of Tree Borrows. When a reference-type argument is passed to a function, it receives a special "function-entry" retag. This applies a "protector" to the reference, making it undefined behavior for it to become invalid before the function returns. 
```Rust
let mut x = 0;
let ptr_x = &mut x as *mut _;
example(&mut x, &mut *ptr_x);

fn example(x: &mut i32, y: *mut i32) {
  // Undefined behavior! Writing through `y` invalidates
  // the "unique" permission for `x`, which has become
  // protected by the call.
  *y = 0;
}
```
This dynamically enforces the invariant that the lifetime of a reference-type argument must cover the entire function. Once a function returns, all protectors need to be removed, which means that BorrowSanitizer needs to know which permissions became protected by the call. We maintain a third invariant to help us with this:

3. For any path through the CFG, all function-entry retags must dominate every other instruction that allocates space on the shadow stack.

This means that the permissions created by function-entry retags will "cluster" adjacent to each other below the frame header. If we track the number `n` of function-entry retags along each path, then at each exit point, we can subtract `n` from the beginning of the shadow stack frame to get a pointer to the start of the list of protected permissions.

This is straightforward in the example above; there's one argument, and it always gets a function-entry retag. However, other types are a bit trickier:
```Rust
enum Tricky<'a, 'b> {
    A,
    B(&'a mut i32),
    C(&'a mut i32, &'b i32)
}
```
If an instance of `Tricky` is passed as an argument, then the number of retags varies along each execution path, depending on which variant we have. We have the same issue with loads, "regular" retags, and any other operation that needs stack slots. Different paths require a different number of slots, but we never want gaps in the shadow stack. 

Our solution for this is to treat the program as if we are dynamically allocating stack slots. We create `allocas` to track the total offset from the top of the shadow stack frame and the number of function-entry retags stored below the frame header. Then, we selectively apply LLVM's [mem2reg](https://llvm.org/docs/Passes.html#mem2reg-promote-memory-to-register) pass to these allocations, lifting what would ordinarily be dynamic operations into statically determined offsets. This makes allocating stack slots relatively straightforward without adding unnecessary run-time overhead.

## What's next?
We've glossed over several key aspects of our shadow stack design, like handling variadic functions and validating metadata from uninstrumented calls. You can expect to see more detail in the upcoming RFC for the LLVM components of BorrowSanitizer. We had expected to have a draft ready by now, but working out the details of our shadow stack took longer than we had planned. We're aiming for mid-May.

Until then, you can reach us [on Zulip](bsan.zulipchat.com) with any questions. Thanks for following along!
