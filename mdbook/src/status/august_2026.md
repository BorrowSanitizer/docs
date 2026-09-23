# Status Update - August 2026

#### By Ian McCormack

We are building BorrowSanitizer: an LLVM-based instrumentation tool for finding violations of Rust's aliasing model in multilanguage applications. If you're new to the project, then we recommend checking out the [introduction](https://borrowsanitizer.com/intro.html) and our [first status update](https://borrowsanitizer.com/status/january_2026.html).

This month, we mainly focused on our [RustConf](https://rustconf.com) presentation. If at RustConf, then you can attend our session on Thursday morning at 11:50 AM, in Room 510bd. Molly MacLaren and Josh Sunshine will also be there! Come find us, and let's chat about securing interoperation between Rust and C++.

We also have a couple of exciting new developments to share.

* Our [RFC](https://discourse.llvm.org/t/rfc-experimental-support-for-borrowsanitizer/90938/22) for upstreaming BorrowSanitizer into LLVM has been accepted! Our plan is for BorrowSanitizer to become a general-purpose tool for validating provenance metadata at the LLVM level.

* Molly has ported our GC heuristics over to Miri, [leading to a 1.6x speedup](https://hackmd.io/@3zl3LBibTc6U57P3qU5NGQ/SyzafNRDzl)! 

* Thanks to feedback from [@cmazakas](https://github.com/cmazakas) and [@Wybxc](https://github.com/Wybxc), we found a couple of new false positives that show up in release mode. These are related to packed structs ([#315](https://github.com/BorrowSanitizer/bsan/issues/315)) and tail recursion elimination ([#306](https://github.com/BorrowSanitizer/bsan/issues/306)). We have fixes in progress!

That's all for now! You can expect to see one of our typical, longer status updates in September. In the meantime, you can reach us in person at RustConf or remotely [on Zulip](https://bsan.zulipchat.com/).