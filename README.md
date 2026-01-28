# Eliane - AI-Powered SDR Agent

High-performance Sales Development Representative agent using the **Model Context Protocol (MCP)** to qualify WhatsApp leads from Meta Ads and push them to C2S CRM.

## Architecture

### MCP Pattern
```
┌─────────────────────────────────────┐
│      Agent Host (FastAPI)           │
│  - Receives Webhooks                │
│  - Manages LLM Context              │
│  - Acts as MCP Client               │
└──────────────┬──────────────────────┘
               │ MCP Protocol
               │ (stdio transport)
┌──────────────▼──────────────────────┐
│    MCP Server (SalesTools)          │
│  - Tools: send_whatsapp,            │
│           push_to_c2s,              │
│           schedule_followup         │
│  - Resources: lead://, project://   │
└─────────────────────────────────────┘
```

## Tech Stack

- **Python 3.11+** with async/await
- **MCP SDK** - Official Model Context Protocol
- **FastAPI** - High-performance web framework
- **PostgreSQL + SQLModel** - Database & ORM
- **Celery + Redis** - Task queue for follow-ups
- **OpenAI / Anthropic** - LLM integration

## Project Structure

```
app/
├── host/                      # Application Entry Point
│   ├── api.py                 # FastAPI webhooks (Meta/Z-API)
│   ├── agent_loop.py          # LLM + MCP Tool execution loop
│   └── settings.py            # Host configuration
├── mcp_server/                # Capabilities Layer
│   ├── server.py              # FastMCP server instance
│   ├── tools/
│   │   ├── communication.py   # send_whatsapp
│   │   ├── crm.py             # push_to_c2s
│   │   └── scheduling.py      # schedule_followup
│   └── resources/
│       ├── leads.py           # lead://{id}/context
│       └── inventory.py       # project://{id}/prices
├── core/
│   ├── models.py              # SQLModel database models
│   ├── db.py                  # Database connection
│   └── celery_worker.py       # Celery task definitions
└── main.py                    # Application launcher
```

## Quick Start

### 1. Setup Environment

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your API keys
nano .env
```

### 2. Start Services

```bash
# Start all services (Postgres, Redis, App, Celery Worker)
docker-compose up -d

# View logs
docker-compose logs -f app
```

### 3. Run Migrations

```bash
# Create tables
docker-compose exec app alembic upgrade head
```

### 4. Test Webhook

```bash
curl -X POST http://localhost:8000/webhooks/meta \
  -H "Content-Type: application/json" \
  -d '{
    "entry": [{
      "changes": [{
        "value": {
          "metadata": {"phone_number_id": "123"},
          "contacts": [{"wa_id": "5511999999999"}],
          "messages": [{
            "from": "5511999999999",
            "text": {"body": "Olá, vim do anúncio"}
          }]
        }
      }]
    }]
  }'
```

## MCP Tools

### `qualify_lead_step`
Updates lead qualification state (Finalidade, Momento, Perfil).

### `send_whatsapp`
Sends a message via Z-API to the lead's WhatsApp.

### `push_to_c2s`
Creates/updates the lead in C2S CRM with AI-generated summary.

### `schedule_followup`
Schedules a delayed message using Celery.

### `enable_auto_followup` ✨ NEW
Enables smart automatic follow-up with escalation (30min → 2h → 24h). Cancels if lead responds.

### `handoff_human`
Marks the conversation for human takeover.

## MCP Resources

### `lead://{lead_id}/context`
Returns the full conversation history for a lead.

### `project://{project_id}/prices`
Returns pricing information for real estate projects.

## Development

### Run Locally (without Docker)

```bash
# Install dependencies
poetry install

# Start Postgres & Redis
docker-compose up postgres redis -d

# Run migrations
alembic upgrade head

# Start FastAPI app
uvicorn app.main:app --reload --port 8000

# Start Celery worker (separate terminal)
celery -A app.core.celery_worker worker --loglevel=info
```

### Code Quality

```bash
# Format code
poetry run black app/

# Lint
poetry run ruff app/

# Type checking
poetry run mypy app/
```

## Configuration

Key environment variables in `.env`:

- `LLM_PROVIDER`: `openai` or `anthropic`
- `LLM_MODEL`: Model to use (e.g., `gpt-4-turbo-preview`)
- `ZAPI_BASE_URL`: Z-API endpoint
- `C2S_BASE_URL`: C2S CRM endpoint
- `DEFAULT_FOLLOWUP_HOURS`: Hours to wait for follow-up

## Agent Behavior

**Eliane** follows these rules:
- Tone: Formal-casual (professional but approachable)
- Never hallucinates prices; always reads `project://` resource
- Qualifies leads through conversation (Finalidade, Momento, Perfil)
- Hands off to humans for technical questions
- **Smart Follow-ups**: Automatic escalation system (30min → 2h → 24h)
  - Checks if lead responded before sending
  - Cancels scheduled messages if lead becomes active
  - Resets escalation when lead responds

## License

Proprietary - Internal Use Only
