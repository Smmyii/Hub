# Freelance Pipeline

This directory manages Sammy's freelance client pipeline — from first contact through delivery and ongoing maintenance.

## Lifecycle

Each client moves through these stages:

1. **lead** — Sammy meets someone, sends contact info via Jarvis or adds directly
2. **scoped** — BMO has assessed what the client needs, created a client file, estimated effort
3. **planned** — Approach approved by Sammy, project repo and agent created
4. **in-progress** — Larry and agents are building
5. **delivered** — Shipped to client, invoice sent
6. **maintenance** — Ongoing support / hosting

## How leads come in

- **Via Jarvis:** Sammy captures from phone -> Jarvis routes to BMO -> BMO creates client file
- **Direct:** Sammy or Hub creates the client file manually

## What BMO does with a new lead

1. Creates a client file in `clients/` using `_template.md`
2. Assesses scope (what they need, rough effort, suggested stack)
3. Creates a project stub repo if appropriate
4. Flags decisions for Sammy (budget, timeline, approach)

## Structure

```
freelance/
  clients/
    _template.md    — blank template for new clients
    elsa.md         — per-client relationship file
    [client].md     — one file per client
```

## Notes

- Client files are the source of truth for relationship status
- One file per client, not per project (a client may have multiple projects later)
- Keep status fields current — Jarvis reads these for routing
