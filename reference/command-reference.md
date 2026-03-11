# Nano Command Reference

Quick reference for all tools and commands across the system. Organized by where you are.

---

## From Your Phone / Desktop (Jarvis Chat)

Talk naturally — Jarvis picks the right tool.

### Pipeline Tasks
| Say | Tool |
|-----|------|
| "What's in the pipeline?" | `pipeline_task_list` |
| "Create a task for BMO: build login page for Elsa" | `pipeline_task_create` |
| "Show me task #3" | `pipeline_task_get` |
| "Assign task 3 to larry:licom" | `pipeline_task_update` |
| "Mark task 3 as done" | `pipeline_task_update` |

### Items & Research
| Say | Tool |
|-----|------|
| "Search for API design notes" | `search_items` |
| "Create a note about..." | `create_item` |
| "Find my research on OAuth" | `search_research` |
| "Remember that I prefer dark mode" | `remember_fact` |

### Canvas
| Say | Tool |
|-----|------|
| "List my canvases" | `canvas_list` |
| "Create a canvas called Architecture" | `canvas_create` |
| "Add a node about auth flow" | `canvas_create_node` |
| "Connect node A to node B" | `canvas_connect` |

### Web
| Say | Tool |
|-----|------|
| "Search the web for..." | `web_search` |
| "Fetch this article: [url]" | `fetch_url` |

---

## From Desktop Claude Code (MCP Tools)

Two MCP servers available in any Claude Code session with the right config.

### nano-tasks (Pipeline API bridge)
| Tool | What |
|------|------|
| `task_list` | List tasks with filters (status, department, assigned_to, type) |
| `task_get` | Get single task by ID |
| `task_create` | Create a pipeline task |
| `task_update` | Update task status/assignment/result |
| `task_stats` | Counts by status, department, type |

### cross-claude (Agent Message Bus)
| Tool | What |
|------|------|
| `register_instance` | Register as an agent (name + description) |
| `send_message` | Send message to a channel |
| `check_messages` | Read messages from a channel |
| `wait_for_message` | Block until new message arrives |
| `create_channel` | Create a new channel |
| `list_channels` | List all channels |
| `find_channel` | Find channel by name |
| `list_instances` | See online agents |
| `search_messages` | Search message history |
| `share_data` | Store shared data (key-value) |
| `get_shared_data` | Retrieve shared data |
| `list_shared_data` | List all shared data keys |
| `heartbeat` | Keep agent presence alive |

### Permanent Channels
| Channel | Purpose | Who |
|---------|---------|-----|
| `#dispatch` | BMO posts task assignments | BMO → Larrys |
| `#status` | Progress updates | Everyone → BMO, Sammy |
| `#decisions` | Escalations needing approval | BMO, Larrys → Sammy |
| `#bob-nano` | Nano project comms | BMO ↔ Bob:Nano |
| `#larry-licom` | Direct comms | BMO ↔ Larry:Licom |
| `#larry-elsa` | Direct comms | BMO ↔ Larry:Elsa |

---

## From VPS Terminal

### Agent Management
```bash
# See running agents
tmux ls

# Peek at BMO
tmux attach -t bmo

# Detach from tmux (without killing)
Ctrl+B, then D

# Start/stop agents (when agents.sh is set up)
./opt/agents/scripts/agents.sh start bmo
./opt/agents/scripts/agents.sh start larry-licom
./opt/agents/scripts/agents.sh status
./opt/agents/scripts/agents.sh stop all
```

### Server Management
```bash
# Rebuild API server
cd ~/nano/deploy
docker compose -f docker-compose.prod.yml up -d --build api

# Rebuild frontend
docker compose -f docker-compose.prod.yml up -d --build frontend

# Check running containers
docker ps

# View API logs
docker logs nano-api --tail 50 -f
```

### Git (on VPS)
```bash
cd ~/nano/core
git pull origin master

# For Hub
cd ~/Documents/Hub
git pull origin main
```

### Database Quick Checks
```bash
# Pipeline tasks
psql -d nano -c "SELECT id, title, status, assigned_to FROM jarvis_tasks WHERE status != 'deleted' ORDER BY id DESC LIMIT 10"

# Agent channels
psql -d nano -c "SELECT name, description FROM channels ORDER BY name"

# Online agents
psql -d nano -c "SELECT instance_id, description, last_heartbeat FROM instances ORDER BY last_heartbeat DESC"
```

---

## REST API (curl / HTTP)

Base URL: `https://nanoli.dev`
Auth: `Authorization: Bearer <token>` on all requests except /health.

### Pipeline Tasks
```bash
# List tasks
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/tasks

# Get single task
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/tasks/3

# Create task
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"title":"Build login page","department":"elsa","assigned_to":"larry:elsa"}' \
  https://nanoli.dev/tasks

# Update task
curl -X PATCH -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"status":"done"}' \
  https://nanoli.dev/tasks/3

# Pipeline stats
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/tasks/stats
```

### Canvas
```bash
# List canvases
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/canvases

# Full canvas with nodes + edges
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/canvases/<id>/full
```

### AI
```bash
# Available models
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/ai/models

# Usage stats
curl -H "Authorization: Bearer $TOKEN" https://nanoli.dev/ai/usage
```

### Health
```bash
curl https://nanoli.dev/health
```

---

*Last updated: 2026-03-11*
