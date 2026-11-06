# Status Update - October 2026
#### By Ian McCormack
We are building BorrowSanitizer: an LLVM-based instrumentation tool for finding violations of Rust’s aliasing model in multilanguage applications. If you are new to this project, then we recommend checking out the [introduction](https://borrowsanitizer.com/intro.html) or our [first status update](https://borrowsanitizer.com/status/january_2026.html) before continuing.

We are in the midst of a large-scale evaluation of BorrowSanitizer, so we opted to keep things short this month. You can expect a much longer blog post next month, along with our experimental data. Here's what's new:

* Our [major change proposal](https://github.com/rust-lang/compiler-team/issues/1041#issuecomment-6099846007) for upstreaming BorrowSanitizer was accepted 🎉!
* Rafayel Amirkhanyan ([@Gitter499](https://github.com/Gitter499)) created an agent skill for BorrowSanitizer and used it to find a new cross-language aliasing bug [in Bun](https://github.com/oven-sh/bun/issues/46598).
* I submitted [a PR](https://github.com/llvm/llvm-project/pull/233624) to the LLVM Project with a small change to the build system for `compiler-rt`. Once this is merged and backported to Rust's LLVM fork, we will have everything that we need to upstream BorrowSanitizer on the Rust side.

* We attended the LLVM Developers Meeting, where I presented [at the Memory Safety Workshop](https://discourse.llvm.org/t/cfp-2026-llvm-memory-safety-workshop/92531) on the latest results from BorrowSanitizer.

* We started an experimental branch, where we have fixed a variety of bugs in our retag intrinsics and introduced a new `__rust_expose intrinsic` to mark provenance-exposing integer to pointer conversions. We will be upstreaming these fixes toward the end of November.

That's all for now. Thank you for your support! You can reach our team on [Zulip](https://bsan.zulipchat.com/).