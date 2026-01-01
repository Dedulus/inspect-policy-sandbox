# Changelog

## [0.1.2] - 2026-01-01

### Fixed
- Fixed critical `TypeError` in `exec`, `read_file`, and `write_file` due to missing `**kwargs` support, ensuring compatibility with Inspect AI runtime (which passes arguments like `timeout_retry`).

## [0.1.1] - 2026-01-01

### Fixed
- Fixed `sample_init` factory method to correctly instantiate `PolicySandboxEnvironment` and resolve inner sandboxes from registry.

## [v0.1.0] - 2025-12-31

### 🚀 Features
- **Policy Enforcement**: `PolicySandboxEnvironment` wrapper to enforce allow/deny lists on `exec`, `read_file`, and `write_file`.
- **Fail-Closed Security**: Deny rules take precedence; unlisted operations blocked if allow-list exists.
- **Inspect Integration**: Registered as `policy-sandbox`. Supports standard configuration via task metadata.
- **Observability**: Emits structured `SandboxEvent` (result=1) on policy violations.

### 📦 Installation
```bash
pip install inspect-policy-sandbox
```

### 📋 Usage
```python
from inspect_ai import Task

task = Task(
    sandbox="policy-sandbox",
    sandbox_config={
        "policy": {
            "deny_exec": ["rm", "sudo"],
            "allow_read": ["/data/*"]
        }
    }
)
```
