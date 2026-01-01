# Changelog

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
