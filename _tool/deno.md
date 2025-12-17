---
title: deno
tags: [cli, config-file, eval-js, eval-ts, first-order-tool, rce]
references: [https://docs.deno.com/runtime/manual/tools/task_runner]
files: [deno.json, deno.jsonc]
---

Secure runtime for JavaScript and TypeScript. **LOTP Classification:** First-Order Tool (RCE via `deno task`).

## `deno.json` - Task Hijacking (RCE)

The `deno task` command executes shell commands defined in the `tasks` configuration object.

\```json
{
  "tasks": {
    "build": "deno run -A malicious.ts"
  }
}
\```

**Sandbox Bypass:** Deno is secure by default (requires explicit permissions). However, `deno task` executes the command string exactly as defined. An attacker can inject permission flags (e.g., `-A`, `--allow-all`) into the task definition to disable the sandbox.

## `*_test.ts` - Test Auto-Discovery

\```ts
Deno.test("pwn", () => {
  // Code executes here
});
\```

- `deno test`: Auto-discovers and executes test files (e.g., `exploit_test.ts`).
- **Constraint:** Runs sandboxed by default. RCE is only possible if the CI workflow explicitly adds permission flags (e.g., `deno test -A`).

## Impact

- **File Read/Write:** Full access if `-A` or specific flags used.
- **Environment:** Secrets access via `Deno.env.get()`.
- **Network:** Exfiltration via `fetch()`.
