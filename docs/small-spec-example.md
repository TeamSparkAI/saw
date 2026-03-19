# Small Spec Example

**Task:** Logging Updates

**Summary:** Issue with logging errors on early exit led to discovery of issues with logging initialization, consistency of logging config across clients, and other minor startup and logging issues.

**Scope:** 1 hour (human + agent working time), 20 files changed, +178/-55 lines of code

**Process and Artifacts:**

Session 1: [Interactive spec creation](../artifacts/logging_and_error_handling_spec.md)

[Resulting spec (54 lines)](../artifacts/logging-file-logger-spec.md)

Generate code

Session 2: [Review of implementation](../artifacts/logging_and_error_handling_impl.md)

Submit PR  
[https://github.com/modelcontextprotocol/inspector/pull/1158](https://github.com/modelcontextprotocol/inspector/pull/1158)

**Claude PR Review**
- **Duration:** 2 minutes, 10 seconds
- **Cost:** $0.36
- **Found:** 2 issues, 3 minor notes. Both issues were valid.

Session 3: [Addressing PR review feedback](../artifacts/logging_and_error_handling_review.md)
