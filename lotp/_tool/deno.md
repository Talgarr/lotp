---
title: deno
tags:
  - cli
  - config-file
  - eval-sh
  - eval-ts
  - env-var
references:
  - https://deno.land/manual@v1.43.0/toolchain/deno_json
  - https://deno.land/manual@v1.43.0/toolchain/tasks
  - https://deno.land/manual@v1.43.0/runtime/permissions
files: [deno.json, deno.jsonc, package.json]
---

Deno is a modern, secure-by-default runtime for JavaScript, TypeScript, and WebAssembly. It integrates essential development tools, making it popular in CI/CD workflows for testing, linting, formatting, and deployment.

## LOTP Exploitation Method

Deno's secure-by-default posture requires explicit permission grants for sensitive operations. However, it is vulnerable to Living Off The Pipeline (LOTP) exploitation primarily through the `deno task` subcommand and, secondarily, when CI pipelines explicitly grant broad permissions to attacker-controlled scripts.

### 1. `deno task` (Arbitrary Command Execution)

The `deno task` subcommand executes arbitrary shell commands defined in `deno.json` (or `deno.jsonc`) or `package.json`'s `scripts` field. If a CI/CD pipeline invokes `deno task <task_name>` (e.g., `deno task build`), an attacker can define or modify this task in the repository to execute arbitrary commands on the CI runner.

### 2. `deno run -A <attacker_controlled_script.ts>` (Script Execution with All Permissions)

If a CI pipeline explicitly grants `--allow-all` (`-A`) permissions to `deno run` for an attacker-controlled script (e.g., `deno run -A malicious.ts`), the attacker can embed arbitrary Deno API calls and system commands within that script, leading to full compromise.

## Configuration Details

Deno automatically discovers `deno.json` or `deno.jsonc` in the current or parent directories. It also respects the `scripts` section of `package.json`.

### `deno.json` / `deno.jsonc`

The `tasks` field within `deno.json` allows defining arbitrary shell commands, which are executed when `deno task <task_name>` is invoked. These tasks can include `deno run` commands with specific permissions (e.g., `--allow-all`) or direct shell commands.

**Example `deno.json` for RCE:**

```json
{
  "tasks": {
    "build": "echo \"$FLAG\" > /tmp/pwned",
    "exfiltrate": "deno run --allow-net malicious_exfil.ts"
  }
}
```

### `package.json`

Deno supports reading `scripts` from `package.json` for Node.js compatibility. These scripts are treated similarly to `deno.json` tasks and can be executed via `deno task <script_name>`.

**Example `package.json` for RCE:**

```json
{
  "name": "project",
  "version": "1.0.0",
  "scripts": {
    "malicious_pkg_task": "echo \"$FLAG\" > /tmp/pwned_package_json"
  }
}
```

### `importMap` Configuration

An `importMap` in `deno.json` can redirect module imports to attacker-controlled URLs or local file paths, potentially leading to supply chain attacks by loading malicious code instead of legitimate dependencies.

## Impact

An attacker exploiting Deno as a LOTP tool can achieve:

*   **Execute arbitrary commands or code (RCE)**: By defining malicious `tasks` in `deno.json`/`package.json` or by controlling scripts executed with `--allow-all`.
*   **Read from or write to files**: Through shell commands in tasks or Deno APIs in scripts, if `deno run` has `--allow-read` or `--allow-write` permissions.
*   **Modify or inject environment variables**: Shell commands within tasks can set or modify environment variables, influencing subsequent pipeline steps.
*   **Initiate network connections**: Tasks or scripts can use `deno run --allow-net` to exfiltrate secrets or other sensitive data from the CI environment.

## CI/CD Workflow Example (GitHub Actions)

This example demonstrates how an attacker can leverage a `deno task` invocation in a CI/CD workflow:

```yaml
name: Build and Test
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: denoland/setup-deno@v2
      with:
        deno-version: v1.x
    - name: Run build task (attacker controls deno.json or package.json)
      run: deno task build
```

In this scenario, if the attacker modifies the `build` task in `deno.json` (or `package.json`'s `scripts` for `build`), their malicious command will be executed.

Similarly, if the CI workflow explicitly runs an attacker-controlled script with broad permissions:

```yaml
name: Deploy
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: denoland/setup-deno@v2
      with:
        deno-version: v1.x
    - name: Run deploy script with all permissions (attacker controls malicious_deploy.ts)
      run: deno run -A malicious_deploy.ts
```

An attacker can then embed arbitrary malicious code within `malicious_deploy.ts` to compromise the runner.
