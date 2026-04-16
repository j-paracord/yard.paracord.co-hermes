# xata/shared/

Org-wide knowledge shared by every agent in this org. Drop any markdown,
text, or reference files here that you want all agents to be able to read —
style guides, API docs, company policies, glossaries, whatever.

## How it's deployed

`hm sync` rsyncs this directory to `/srv/hm/xata/shared/` on the VM.
Each agent profile gets a `./shared/` symlink pointing at that location, so
from an agent's perspective the contents show up as `./shared/` inside its
profile dir.

## Read-only from the agent side

Currently the manager is the sole writer: agents can read `./shared/` but
should not write to it. Edit files here on your workstation and let `hm sync`
propagate the changes.

## Opt-in

This dir is scaffolded by `hm org create` as a hint — it's entirely optional.
If you don't want a shared knowledge base, delete this README and the
`shared/` directory; `hm sync` treats an absent `shared/` as a no-op and will
skip it gracefully.
