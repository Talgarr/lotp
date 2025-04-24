---
title: Dita
tags:
  - cli
  - config-file
  - eval-sh
references:
  - https://www.dita-ot.org/dev/parameters/dot-ditaotrc-file
  - https://www.dita-ot.org/dev/parameters/local-properties-file
files: [.ditaotrc, local.properties]
---

`dita` is a CLI tool for documentation publishing. It is highly extensible and
can be configured via a configuration file.

## PDF Formatter

The PDF formatter can be specified to any executable using its full path.
For example, the following configuration will execute `/tmp/pwn.sh` when
running `dita --project=app.xml`:

```toml
pdf.formatter = ah
axf.cmd = /tmp/pwn.sh
```

The `app.xml` project file needs to be modified to include PDF capability. Here is
a minimal example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-model href="https://www.dita-ot.org/rng/project.rnc" type="application/relax-ng-compact-syntax"?>
<project xmlns="https://www.dita-ot.org/project">
  <deliverable id="pdf">
    <context name="User Guide">
      <input href="test.ditamap"/>
    </context>
    <output href="."/>
    <publication transtype="pdf2">
    </publication>
  </deliverable>
</project>
```

It requires a minimal valid `test.ditamap` file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE bookmap PUBLIC "-//OASIS//DTD DITA BookMap//EN" "bookmap.dtd">
<bookmap xml:lang="en-US">
</bookmap>
```
