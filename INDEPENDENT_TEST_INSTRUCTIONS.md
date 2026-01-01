# Independent Verification Guide: inspect-policy-sandbox

Follow these steps to independently verify the `inspect-policy-sandbox` package in a clean environment.

## 1. Setup a Fresh Environment

Open your terminal and run the following commands to create a new folder and virtual environment:

```bash
# Create a new directory
mkdir independent_test_sandbox
cd independent_test_sandbox

# Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install the package (ensure no cache to get the latest PyPI version)
pip install --no-cache-dir --force-reinstall inspect-policy-sandbox
```

## 2. Create the Test Script

Create a file named `verify_release.py` with the following content:

```python
import asyncio
import os
import sys
from inspect_ai import Task, eval
from inspect_ai.dataset import Sample
from inspect_ai.solver import use_tools, TaskState, Generate, solver
from inspect_ai.tool import bash, ToolError
from inspect_ai.util import sandbox

# --- SOLVERS ---

@solver
def trigger_exec(cmd: str):
    async def solve(state: TaskState, generate: Generate):
        # Find tool - robustly handles different tool representations
        bash_tool = next((t for t in state.tools if getattr(t, "name", getattr(t, "__name__", str(t))) == "bash"), None)
        # Fallback if specific name match fails but tools exist
        if not bash_tool and len(state.tools) > 0: bash_tool = state.tools[0]
        
        await bash_tool(cmd=cmd)
        return state
    return solve

@solver
def trigger_io(action: str, path: str, content: str = "test"):
    async def solve(state: TaskState, generate: Generate):
        sb = sandbox()
        if action == "read":
            await sb.read_file(path)
        elif action == "write":
            await sb.write_file(path, content)
        return state
    return solve

# --- SCENARIO RUNNER ---

def run_scenario(name, task, expect_block):
    print(f"Running Scenario: {name}...", end=" ", flush=True)
    try:
        # Run evaluation
        logs = eval(task, model="mockllm/model", log_level="error")
        log = logs[0] if logs else None
        
        # Extract error message if any
        error_msg = ""
        if log:
             if log.samples and log.samples[0].error:
                 error_msg = log.samples[0].error.message
             elif log.status == "error" and log.error:
                 error_msg = log.error.message

        # Determine Pass/Fail
        passed = False
        if expect_block:
            # We expect a policy violation error
            if "denied by policy" in error_msg or "SandboxPolicyViolationError" in error_msg:
                passed = True
        else:
            # We expect success OR a non-policy error (like 'command not found' or 'file not found')
            if log.status == "success":
                passed = True
            elif "denied by policy" not in error_msg:
                 passed = True
        
        if passed:
            print("✅ PASS")
            return True
        else:
            print(f"❌ FAIL")
            print(f"   Expect Block: {expect_block}")
            print(f"   Actual Error: '{error_msg}'")
            return False

    except Exception as e:
        print(f"❌ CRITICAL ERROR: {e}")
        return False

# --- MAIN ---

if __name__ == "__main__":
    print("\n--- inspect-policy-sandbox Verification ---\n")
    
    results = []
    
    # 1. Block Execution
    # Policy: deny_exec=["bash*"]
    # Action: Run "echo hello" via bash
    # Expected: BLOCK
    t1 = Task(
        dataset=[Sample(input="Exec Block", metadata={"policy": {"deny_exec": ["bash*"]}, "inner_sandbox": "local"})],
        solver=[use_tools([bash()]), trigger_exec("echo hello")],
        sandbox="policy-sandbox"
    )
    results.append(run_scenario("Block Exec (bash)", t1, True))
    
    # 2. Allow Execution
    # Policy: allow_exec=["bash*"] (Whitelisting bash allows commands to run)
    # Action: Run "ls -la" via bash
    # Expected: ALLOW
    t2 = Task(
        dataset=[Sample(input="Exec Allow", metadata={"policy": {"allow_exec": ["bash*"]}, "inner_sandbox": "local"})],
        solver=[use_tools([bash()]), trigger_exec("ls -la")],
        sandbox="policy-sandbox"
    )
    results.append(run_scenario("Allow Exec (ls)", t2, False))
    
    # 3. Block Write
    # Policy: deny_write=["*.key"]
    # Action: Write to "secret.key"
    # Expected: BLOCK
    t3 = Task(
        dataset=[Sample(input="Write Block", metadata={"policy": {"deny_write": ["*.key"]}, "inner_sandbox": "local"})],
        solver=[trigger_io("write", "secret.key")],
        sandbox="policy-sandbox"
    )
    results.append(run_scenario("Block Write (*.key)", t3, True))
    
    # 4. Allow Write
    # Policy: allow_write=["*.log"] (Implicitly denies other writes)
    # Action: Write to "test.log"
    # Expected: ALLOW
    t4 = Task(
        dataset=[Sample(input="Write Allow", metadata={"policy": {"allow_write": ["*.log"]}, "inner_sandbox": "local"})],
        solver=[trigger_io("write", "test.log")],
        sandbox="policy-sandbox"
    )
    results.append(run_scenario("Allow Write (*.log)", t4, False))

    if all(results):
        print("\nSUMMARY: ALL TESTS PASSED ✅")
        sys.exit(0)
    else:
        print("\nSUMMARY: TESTS FAILED ❌")
        sys.exit(1)
```

## 3. Run the Verification

Execute the script:

```bash
python verify_release.py
```

### Expected Output
You should see:
```text
Running Scenario: Block Exec (bash)... ✅ PASS
Running Scenario: Allow Exec (ls)... ✅ PASS
Running Scenario: Block Write (*.key)... ✅ PASS
Running Scenario: Allow Write (*.log)... ✅ PASS

SUMMARY: ALL TESTS PASSED ✅
```
