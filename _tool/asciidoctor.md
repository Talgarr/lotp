---
title: asciidoctor
tags: [cli, input-file, file-read, first-order-gadget]
references: [https://docs.asciidoctor.org/asciidoc/latest/directives/include/]
files: ["*.adoc"]
---

Text processor for converting AsciiDoc files to HTML, PDF, and other formats. **LOTP Classification:** First-Order Gadget (File Read).

## `*.adoc` - Arbitrary File Read

The `include::` directive allows importing content from other files. By default, the `asciidoctor` CLI runs in **Unsafe Mode**, which permits absolute paths and paths outside the project root.

```asciidoc
= Leaked Document

// Include sensitive file content into the output
include::/etc/passwd[]

// Include environment variables if dumped to a known location
include::/tmp/secret_file[]
```

**Impact:** If the build output (HTML/PDF) is published (e.g., GitHub Pages, build artifacts) or accessible to the attacker, they can read any file on the build system.

## Configuration & Sandbox

### Safe Modes
Asciidoctor has four safe modes. The CLI defaults to **Unsafe** (0).

- **Unsafe (0):** Allows reading any file.
- **Safe (1):** Prevents including files outside the parent directory of the source file.
- **Server (10) / Secure (20):** Disables `include::` for arbitrary paths.

### Remediation
Run with safe mode enabled:
```bash
asciidoctor -S safe input.adoc
```

## RCE Potential (Extensions)

While core Asciidoctor is primarily a File Read gadget, using extensions like `asciidoctor-diagram` (often installed alongside) can theoretically introduce RCE by overriding executable paths (e.g., `:graphviz-dot: /bin/sh`), though this often requires specific conditions or wrapper scripts to exploit effectively.
