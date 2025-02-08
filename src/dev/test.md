# Testing & CI
We have unit tests for our runtime library and UI tests for the entire toolchain. Our unit tests are within `bsanrt`, and our UI tests are within `bsan-driver`. You can execute each test suite by passing the relative path to these components from the root directory as an argument to `./x.py test`. 

For example, this command executes our unit tests...
```
./x.py test src/tools/bsan/bsanrt
```
...while this one executes our integration tests:
```
./x.py test src/tools/bsan/bsan-driver
```
Our CI pipeline checks for formatting, and then it builds and tests our toolchain on each of our supported architectures. Make sure to run `./x.py fmt` and execute each of our test cases before opening a pull request. All pull requests should be made using the branch `BorrowSanitizer/bsan` as the base for comparison. When using GitHub, be careful to select the correct branch, since the interface defaults to submitting a pull request against the main branch of the Rust toolchain! 

All pull requests must pass our CI pipline. External pull requests must be reviewed by a member of the team. 
