# Status Update - September 2026
#### By Ian McCormack, Molly MacLaren, and Joshua Sunshine

We are building BorrowSanitizer: an LLVM-based instrumentation tool for finding violations of Rust’s aliasing model in multilanguage applications. If you are new to the project, then we recommend checking out the [introduction](https://borrowsanitizer.com/intro.html) or our [first status update](https://borrowsanitizer.com/status/january_2026.html) before you continue.

This month, we all attended [RustConf](https://rustconf.com) to present on BorrowSanitizer. It was great to meet so many fellow Rustaceans and to learn more about the diverse set of challenges associated with Rust interoperation. If you missed us at RustConf, then you can check out our talk [on YouTube](https://www.youtube.com/watch?v=yWZm26aAOLA). We will also be at the [LLVM developers meeting](https://llvm.swoogo.com/2026devmtg) in October, where we will be attending the memory safety workshop. 

Here's what else we've been up to this month:
* We began upstreaming improvements to Miri, and we submited a major change proposal ([MCP](https://forge.rust-lang.org/compiler/proposals-and-stabilization.html#proposals)) to the Rust compiler team for making the remaining components of BorrowSanitizer available for use experimentally in nightly toolchains.
* We reduced our garbage collection overhead by switching to thread-local and tree-local heuristics.
* We fixed a variety of soundness issues in our garbage collector.

### Upstreaming

Last week, Molly submitted [a PR](https://github.com/rust-lang/miri/pull/5339) to Miri with two new optimizations for its garbage collector. Combined, these changes contribute to a **1.50x** speedup in the overall mean runtime overhead! Once all of BorrowSanitizer's GC features are upstreamed to Miri (see [this proposal](https://hackmd.io/@3zl3LBibTc6U57P3qU5NGQ/SyzafNRDzl)), we expect to see a **2.46x** mean runtime speedup! 

Yesterday, we posted a major change proposal (MCP) for upstreaming BorrowSanitizer into the Rust toolchain. This somewhat like a mini-RFC; it's a necessary step for getting approval to make a significant internal change to the Rust compiler.  If [our proposal](https://github.com/rust-lang/compiler-team/issues/1041) is accepted, then our tooling will be available as an experimental component for use with nightly toolchains. This is similar to what has been done before with the [Enzyme project](https://github.com/rust-lang/enzyme). Enzyme's prior work on Rust integration—in particular, the unstable `-Zllvm-plugins` flag—has been incredibly helpful for BorrowSanitizer. 

### Local Heuristics for Faster Garbage Collection

BorrowSanitizer supports real-time analysis during multithreaded execution. This means that if you run `cargo bsan test` on a program, test cases execute concurrently, just as if you had run `cargo test`. Miri is single-threaded; it can simulate true multithreading, which is invaluable for detecting data races, but this also limits its performance in highly concurrent scenarios. 

In multithreaded programs, we have found that maintaining consistent global state is particularly costly for our category of runtime checking. For example, BorrowSanitizer's performance improved after we moved the protection status of an individual node from a global hash-map to a node-level flag. Additionally, our runtime overhead is usually concentrated in a small fraction of the trees created by a program.

Based on this intuition, our latest changes to the garbage collector (GC) have moved certain activation heuristics from the global level to the thread and tree level. This reduces the overall overhead of the GC by ensuring that it runs where it can have the greatest potential impact on performance.

#### Per-thread visit counting

Both BorrowSanitizer and Miri (with the changes in our [latest PR](https://github.com/rust-lang/miri/pull/5339)) use a tree visit-count interval to decide when it's time to prune unreachable nodes. Naively, this count should be a proxy for the number of memory accesses performed by the program between GC runs. However, this is not always accurate. Many of these visits are accumulated from validating relatively few accesses against the largest trees.

Making visit counts thread-local ensures that the first thread to reach the interval will activate the GC. The most Tree Borrows-intensive threads drive the activation, which benefit the most from being regularly pruned. Meanwhile, less intensive threads ultimately benefit from running the GC less overall. This allows us to eliminate a constant global visit-count-update overhead.

#### Per-tree dead node tracking
Until now, our GC has maintained a global "deadlist" of nodes with a zero reference-count. If a node is no longer reachable in memory, then we can remove it, as long as it satisfies one of these two conditions:

1. If it is a leaf node, then it can be trivially removed.
2. If it is an internal node, then we can only remove it if we can replace it with child nodes that have equivalent or stricter permission states.

Permission states are what indicates that a node has triggered undefined behavior. Even if a reference has gone out of scope, it can still detect UB as updates are percolated throughout the tree. Nodes that fall into the second category can become stuck in the deadlist for many cycles until they can be safely removed, and their permission states are re-checked every pass.

We've therefore swapped out our global deadlist for tree-local deadlists--which we don't want to check every time. This lets us support a new, cheap heuristic: we prune a tree only if it has new dead nodes since the last GC pass, indicating other potential updates to the tree. By checking for deadlist updates, we spend less time deciding whether to prune than a more fine-grained approach like checking for permission state updates.

### Stopping the World Safely

In addition to improving our garbage collection heuristics, we also redesigned how we invoke the garbage collector. Before, we used LLVM's existing "stop-the-world" functionality. This was easy to implement, but unsound; it left threads in an inconsistent state. 

For instance, BorrowSanitizer operates by tracking provenance metadata through shadow memory. If a thread is stopped in the middle of updating a provenance value, then our garbage collector might see a stale or invalid value and end up accidentally collecting something that we still need to keep around!

Luckily, LLVM has another built-in solution that can solve this problem for us: [safepoints](https://llvm.org/docs/Statepoints.html). A safepoint is a location where a program is guaranteed to be in a state that can be consistently observed by a garbage collector. When a thread reaches a safepoint, it checks to see if garbage collection has been requested. If so, it halts, waiting for the garbage collector to finish. 

LLVM provides an instrumentation pass that can insert placeholder function calls at each of these locations. We use its default `statepoint-example` mode, which places safepoints at the entry and exit of function calls, and within the backedges of loops. We have a prototype "version 2.0" of our garbage collector that relies on this mechanism, and early results indicate a ~5% speedup on average.

### Conclusion

That's all for now. You can find us at the LLVM Developers Meeting in a few weeks, and we will be back in October with our next status update! You can always reach us on [Zulip](https://bsan.zulipchat.com/).