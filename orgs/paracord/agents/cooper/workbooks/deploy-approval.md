---
title: Deploy with Approval
secrets:
  provider: dotenv
  path: ../.env
---

# Deploy with Approval

Run pre-deploy checks, pause for human approval, then deploy.

## Pre-deploy checks

```bash
echo "=== Pre-deploy checks ==="
echo "Agent: $(whoami)@$(hostname)"
echo "Time: $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
echo ""
echo "Checking service health..."
echo "  API: ok"
echo "  DB: ok"
echo "  Redis: ok"
```

## Await approval

```wait
kind: slack
match:
  channel: agent-tasks
  reaction: thumbsup
bind: approved_by
timeout: 1h
on_timeout: abort
```

## Deploy

```bash
echo "=== Deploy approved by: $approved_by ==="
echo "Deploying at $(date -u +"%Y-%m-%dT%H:%M:%SZ")..."
echo "  Pulling latest..."
echo "  Restarting services..."
echo "  Done."
```

## Post-deploy verification

```bash
echo "=== Post-deploy ==="
echo "Verifying deployment..."
echo "  API responding: ok"
echo "  Version: v0.7.2"
echo "Deploy complete."
```
