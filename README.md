# geopolitical-news-agent-gcp

An agent deployed on GCP that connects to an MCP server deployed as an AWS Lambda function and writes a cited briefing.

The agent is built with Google's [Agent Development Kit (ADK)](https://google.github.io/adk-docs/) and runs Gemini on Vertex AI. It calls one tool, `get_feeds`, on the MCP server in [geo-news-mcp-server](https://github.com/RajkiranVS/geo-news-mcp-server). The server reads allowlisted RSS feeds (BBC World, The Hindu) and needs no changes to work with this agent.

```
User → ADK agent (Gemini on Vertex AI) → MCP over Streamable HTTP + x-api-key → AWS Lambda (FastMCP) → RSS feeds
```

## Layout

```
geo_news_agent/
  agent.py        # McpToolset + system prompt + root_agent
  __init__.py
  .env.example    # copy to .env and fill in
  requirements.txt  # used by the Agent Engine deploy
requirements.txt    # points to the agent's requirements for local installs
```

## Run locally

Prerequisites: Python 3.10+, the [gcloud CLI](https://cloud.google.com/sdk/docs/install), a GCP project with billing and the Vertex AI API enabled, and a deployed geo-news MCP server.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1          # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
gcloud auth application-default login
copy geo_news_agent\.env.example geo_news_agent\.env   # then fill in the values
adk web .
```

Open the dev UI, pick `geo_news_agent` and ask something like "What's new with the Russia–Ukraine war?". The events panel shows the `get_feeds` call and the tool response.

## Deploy to Vertex AI Agent Engine

```powershell
gcloud services enable aiplatform.googleapis.com cloudresourcemanager.googleapis.com
adk deploy agent_engine --project=<your-gcp-project-id> --region=us-central1 --display_name=geo-news-agent --otel_to_cloud geo_news_agent
```

The deploy reads `geo_news_agent/.env` and passes its values (including `MCP_API_KEY` and `MCP_URL`) to the deployed agent as environment variables. `--region` sets where Agent Engine runs; `GOOGLE_CLOUD_LOCATION=global` in `.env` still applies to the Gemini calls. It also needs `geo_news_agent/requirements.txt`, so the `[mcp]` extra is installed in the container.

The deployed agent is listed under Vertex AI → Agent Engine in the Cloud Console.

## Configuration (`geo_news_agent/.env`)

| Variable | Value |
| --- | --- |
| `GOOGLE_GENAI_USE_VERTEXAI` | `TRUE` |
| `GOOGLE_CLOUD_PROJECT` | Your GCP project ID |
| `GOOGLE_CLOUD_LOCATION` | `global` |
| `MCP_API_KEY` | The plain API key the MCP server expects |
| `MCP_URL` | The Lambda Function URL, ending in `/mcp` |

`.env` is gitignored. Keep the key and the URL out of the code.

## Gotchas

- **`ImportError: cannot import name 'McpToolset'`:** in ADK 2.15, MCP support is an optional extra. Install `google-adk[mcp]` (quote it in PowerShell).
- **Model not available in your region:** set `GOOGLE_CLOUD_LOCATION=global`.
- **`adk` not recognised:** install into a virtual environment and activate it.
- **401 from the MCP server:** `MCP_API_KEY` must be the plain key, not the JSON secret string.

## Related

- MCP server: [RajkiranVS/geo-news-mcp-server](https://github.com/RajkiranVS/geo-news-mcp-server)
