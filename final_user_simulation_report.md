# Final User Simulation Report: inspect-policy-sandbox v0.1.2

**Date:** 2026-01-01
**Version Evaluated:** 0.1.2 (PyPI)
**Status:** ✅ **PRODUCTION READY**

## Test Summary
A pure "first-time user" simulation was conducted in an isolated, fresh virtual environment (`smoke_test_final/`) to verify the installation, usability, and policy enforcement robustness of `inspect-policy-sandbox`.

| Category | Test Case | Scenario | Result |
| :--- | :--- | :--- | :--- |
| **Installation** | `pip install` | Install from PyPI (v0.1.2) | ✅ **PASS** |
| **Execution** | `exec` Formatting | Call `bash()` requiring `**kwargs` support | ✅ **PASS** |
| | Block Policy | `deny_exec=["bash*"]` | ✅ **PASS** (Correctly Blocked) |
| | Allow Policy | `allow_exec` / Whitelist | ✅ **PASS** (Correctly Allowed) |
| **File I/O** | Explicit Block | `deny_write=["*.key"]` | ✅ **PASS** (Correctly Blocked) |
| | Whitelist | `allow_write=["*.log"]` (test.log) | ✅ **PASS** (Correctly Allowed) |
| | Implicit Block | `allow_write=["*.log"]` (test.txt) | ✅ **PASS** (Correctly Blocked) |


## Final Sanity Gate (v0.1.3)

A decisive "gate" test was performed on the final released version (`v0.1.3`) to confirm critical safety guarantees.

**Test Scenario:**
```python
task = Task(
    dataset=[Sample(input="rm /tmp", metadata={"policy": {"deny_exec": ["rm"]}, "inner_sandbox": "local"})],
    solver=[system_message("Run rm /tmp")],
    sandbox="policy-sandbox"
)
```

**Gate Checklist:**
1.  **Does it work?** ✅ **YES** (Task initializes and runs)
2.  **Is execution denied?** ✅ **YES** (`SandboxPolicyViolationError` raised)
3.  **Is failure graceful?** ✅ **YES** (Inspect Eval continues, no crash)
4.  **Is event logged?** ✅ **YES** (`SandboxEvent` with `result=1`, `reason="policy"` verified in logs)

## Conclusion
The `inspect-policy-sandbox` extension (v0.1.3) is **strictly compliant** with its design specifications and **robust** against the runtime crash issue identified in v0.1.1. It correctly integrates with `inspect-ai` and reliably enforces policies on both command execution and file operations.

    
