---
title: Browser Login Flow
secrets:
  provider: dotenv
  path: ../.env
---

# Browser Login Flow

Start a browser session, pause for human login, then scrape the authenticated page.

## Start browser session

```python
import json, os, urllib.request

api_key = os.environ["BROWSERBASE_API_KEY"]
project_id = os.environ["BROWSERBASE_PROJECT_ID"]

req = urllib.request.Request(
    "https://www.browserbase.com/v1/sessions",
    data=json.dumps({"projectId": project_id}).encode(),
    headers={
        "x-bb-api-key": api_key,
        "Content-Type": "application/json",
    },
)
resp = json.loads(urllib.request.urlopen(req).read())
session_id = resp["id"]

print(f"Session: {session_id}")
print(f"Live view: https://www.browserbase.com/sessions/{session_id}/debug")
```

## Wait for human login

The operator opens the live view link above, logs into the target site,
and resumes when the session is authenticated.

```wait
kind: manual
match:
  action: browser-login
bind: login_status
timeout: 15m
on_timeout: abort
```

## Scrape authenticated content

```python
print(f"Login status: {login_status}")
print(f"Resuming with session: {session_id}")
print("Connecting to authenticated session...")
# session_id is still in scope from the first python block
# use it to connect playwright/CDP to the warm session and scrape
```
