# RHDH MCP Evaluation

This branch contains the evaluation resources for **RHDH 1.10.2**, covering the MCP tool-calling capabilities of Red Hat Developer Hub / Backstage MCP tools from [`rhdh-plugins/workspaces/mcp-integrations`](https://github.com/redhat-developer/rhdh-plugins/tree/main/workspaces/mcp-integrations).

**Canonical repo:** https://github.com/redhat-developer/rhdh-mcp-eval

## 📂 Repository Structure

| Path | Description |
| --- | --- |
| **[📂 dataset](./dataset)** | Gold dataset — 99 conversations with expected tool calls and responses |
| **[📂 evaluation-result](./evaluation-result)** | Detailed metrics and outcome reports from the model evaluations |
| **[📂 config](./config)** | Scoring configuration for `lightspeed-eval` |
| **[📂 scripts](./scripts)** | Gold builder, multi-provider trace gen, campaign runner |
| **[📄 categories.yaml](categories.yaml)** | Task category definitions for classifying conversations |

---

## 🧪 Evaluation Overview

For the RHDH 1.10.2 release, we evaluated MCP tool-calling performance across six models against 99 conversations spanning 13 MCP tools in 3 plugin domains (Catalog, TechDocs, Scaffolder).

**Models Evaluated:**

- **Gemini:** `gemini-2.5-pro`, `gemini-2.5-flash-lite`
- **GPT:** `gpt-5.5`, `gpt-5-mini`, `gpt-4o-mini`
- **Llama:** `llama-31-8b` (`redhataillama-31-8b-instruct` via OpenShift vLLM/3scale)

**Judge Models:** `gemini-2.5-pro`, `gpt-4o-mini` (aggregation: max)

> 📊 **View Results:** For a deep dive into the performance metrics, please refer to the **[Evaluation Results](./evaluation-result)** directory.

---

## ⚙️ Methodology

The dataset was constructed manually from the MCP tools exposed by `mcp-integrations`, covering:

- **Catalog:** entity queries, filtering by kind/owner/type/tag, entity lookups, model descriptions
- **TechDocs:** coverage analysis, doc fetching, content retrieval
- **Scaffolder:** template metadata, task listing, action listing, validation, dry runs
- **Multi-step:** chained tool calls (e.g. entity lookup → owner components)
- **Negative:** edge cases where the model should handle missing entities, wrong tool boundaries

**Evaluation Tool:** Scoring was executed using **[lightspeed-evaluation](https://github.com/lightspeed-core/lightspeed-evaluation)**, which consumes the dataset and calculates performance metrics using the configured judge panel.

---

## Fixture Assumptions

Local `mcp-integrations` demo catalog:

- Owners such as `payments-team`, `security-team`, `accounts-team`
- Entities such as `consent-management-api`, payment APIs
- **Template kind count is 0** → `execute-template` not scored; write coverage uses dry-run validate
- TechDocs indexing may be empty; retrieve-content is optional when fetch returns none

Prefer overlay tools (`*-mcp-extras.*`) over upstream duplicates.

## Prerequisites

1. Backstage/RHDH MCP on the backend: `http://localhost:7007/api/mcp-actions/v1`
2. `MCP_TOKEN` matching static MCP external access
3. OpenAI API key (agent models + `gpt-4o-mini` judge)
4. Vertex ADC for Gemini agents/judges (`GOOGLE_APPLICATION_CREDENTIALS`, project `rhdh-ai`)
5. Llama OpenAI-compatible endpoint + token (see campaign script defaults)
6. Clone of `lightspeed-evaluation` with `uv sync`

Credentials stay in the environment or gitignored local files.

## Running the Campaign

```bash
python3 -m venv .venv && .venv/bin/pip install -r scripts/requirements.txt

export MCP_TOKEN=...
# OpenAI / Vertex / Llama secrets as used by scripts/run_campaign.sh

chmod +x scripts/run_campaign.sh
./scripts/run_campaign.sh
```

Or step by step:

```bash
.venv/bin/python scripts/build_gold_dataset.py

.venv/bin/python scripts/generate_traces.py \
  --provider openai --model gpt-4o-mini --model-dir gpt-4o-mini \
  --openai-key-file "$HOME/Documents/openai-token.txt"

# ... repeat for other models (see scripts/run_campaign.sh)

cd /path/to/lightspeed-evaluation
uv run lightspeed-eval \
  --system-config "$REPO/config/system-offline-tool-and-judge.yaml" \
  --eval-data "$REPO/evaluation-result/gpt-4o-mini/evaluation_dataset.yaml" \
  --output-dir "$REPO/evaluation-result/gpt-4o-mini"

.venv/bin/python scripts/generate_comparison_graphs.py "$REPO/evaluation-result"
```

## Related

- Tools under test: `rhdh-plugins/workspaces/mcp-integrations`
- Runner: https://github.com/lightspeed-core/lightspeed-evaluation
- Pattern reference: https://github.com/redhat-developer/rhdh-intelligent-assistant-evaluation
- Jira: RHIDP-14578 (evals); RHIDP-14577 (feasibility)
