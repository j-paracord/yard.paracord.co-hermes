**Runbooks Location & Status Check**

Runbooks are stored in: `/home/hm-scout/.hermes/profiles/scout/.trigger-state/runbooks/`

Structure:
- `runs/` — contains JSON files with individual run records (UUID-named)
- `cache/` — cached markdown files 
- `config.json` — runbook configuration (org: xata, agent: scout)

How to check today's status:
1. Search `runs/` directory with today's date (format: YYYY-MM-DD)
2. Parse JSON files for: workbook name, status, exit_code, started_at/ended_at timestamps
3. Successful runs: status="complete" and exit_code=0
4. Failed runs: status="complete" but exit_code!=0

Current workbooks tracked: google-drive-probe, google-sheets-probe