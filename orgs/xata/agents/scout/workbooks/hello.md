---
title: Hello Workbook
secrets:
  provider: dotenv
  command: ../.env
---

# Smoke test

Verify that `wb` is installed and callbacks are flowing.

```bash
echo "hello from $(whoami) on $(hostname)"
```

```bash
echo "workbooks directory: $(pwd)"
ls -la
```

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```
