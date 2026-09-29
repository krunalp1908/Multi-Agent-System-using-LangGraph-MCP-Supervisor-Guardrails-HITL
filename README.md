# TripMate AI

A multi-agent travel planner built with LangGraph, MCP, FastAPI, and Groq. A supervisor selects the travel specialists needed for a request, an input guardrail blocks unrelated requests, and human review is required before a draft itinerary is finalized.

## Features

- Supervisor routes requests to flight, hotel, weather, budget, and itinerary agents.
- Input guardrail checks that requests are travel-related.
- Flight planning uses the local `airportsdata` dataset for airport identification. Airline service, schedules, and fares are not live-confirmed.
- Hotel search uses Tavily MCP; current conditions and forecast data use the custom OpenWeather MCP server.
- LangGraph checkpoints conversation state in PostgreSQL and pauses at a human-approval step.
- Reviewers can approve a draft or request a revision with feedback.
- FastAPI serves the web interface and JSON API.

## Workflow

1. Submit a travel request in the web UI or through the API.
2. The guardrail checks the request, then the supervisor selects agents and extracts trip constraints.
3. Selected agents gather available travel context and the itinerary agent creates a draft.
4. The workflow pauses for human review. Approve the draft or send revision feedback.
5. The final agent generates the reviewed plan.

## Requirements

- Python 3.13 or newer
- A PostgreSQL database reachable by the application
- A Groq API key
- A Tavily API key for live hotel search
- An OpenWeather API key for weather requests

The AviationStack MCP client is included for experimentation, but the flight agent does not call its airport or airline endpoints because those functions may be restricted by the account plan. Airport identification uses the local dataset instead.

## Setup

Clone the repository and enter the project directory:

```powershell
git clone https://github.com/krunalp1908/Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL.git
cd Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL
```

Create a virtual environment and install dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Create a local `.env` file in the project root and provide the keys below. Use real credentials locally; never commit this file.

```dotenv
DATABASE_URL=postgresql://user:password@host:5432/database?sslmode=require
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
```

`DATABASE_URL` and `GROQ_API_KEY` are required when the backend starts. Tavily and OpenWeather are used by their respective agents. `.env` is excluded by `.gitignore`.

Start the app from the project root:

```powershell
.\.venv\Scripts\python.exe app.py
```

Open <http://127.0.0.1:8000/>. The API documentation is available at <http://127.0.0.1:8000/docs>.

## API

### `POST /api/travel`

Starts a travel-planning run and returns the draft when the graph pauses for approval.

```json
{
  "message": "Plan a 7-day trip to Japan from Bangladesh under 200,000 BDT.",
  "thread_id": null
}
```

The response includes `thread_id`, `answer`, `itinerary`, `selected_agents`, `supervisor_reasoning`, and `requires_approval`. Keep the returned thread ID to resume the same workflow.

### `POST /api/travel/approve`

Resumes a paused workflow. Set `approved` to `true` to accept the draft or `false` to request a revision; revision requests require feedback.

```json
{
  "thread_id": "thread-id-from-the-draft-response",
  "approved": false,
  "feedback": "Reduce accommodation costs and add more free activities."
}
```

### `GET /health`

Returns the service status and advertised workflow features.

## Project layout

| Path | Purpose |
| --- | --- |
| `app.py` | FastAPI application and HTTP endpoints |
| `backend.py` | LangGraph state, supervisor, agents, approval interrupt, and PostgreSQL checkpointer |
| `mcp_client.py` | MCP clients for Tavily, AviationStack, and the local weather server; destination extraction |
| `custom_weather_mcp_server.py` | OpenWeather-backed MCP tools for current conditions and forecast |
| `templates/` | HTML page for the web interface |
| `static/` | Browser JavaScript and CSS |

## Notes and limitations

- Flight suggestions are planning guidance, not booking results. Airport records are local; airline routes, schedules, and ticket prices are not verified live.
- Hotel and weather data depend on valid provider keys, network access, and each provider's service availability.
- PostgreSQL checkpointer setup runs during backend initialization, so the database must be reachable when starting the app.
- The repository does not currently include an automated test suite.

## Security

- Keep API keys and database credentials in `.env` or a secrets manager, not in source code.
- If a credential was ever committed or sent to a remote, rotate it even after removing it from the commit.
- Limit database credentials to the permissions required by the application.