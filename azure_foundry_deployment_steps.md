# Deploying Thalamus SRE on Azure — Step-by-Step Runbook

> A practical, copy-paste deployment guide for standing up **Thalamus SRE** (the React UI,
> the `/api/*` backend, the agent fleet, and the LLMs) on Azure, using **GPT-5.4** as the
> reasoning model on **Azure AI Foundry**. For the architectural rationale behind the agent
> design, see [`05-azure-ai-foundry-deployment.md`](05-azure-ai-foundry-deployment.md); this
> document is the *how-to-deploy* companion.

---

## 0. What you are deploying (the three layers)

Thalamus SRE has three deployable concerns. The table maps each to the Azure service that hosts it.

| Layer | What it is (in this repo) | Azure service to use |
|---|---|---|
| **1. UI + backend (one unit)** | React 18 + Vite SPA **and** its `/api/*` server middleware (`app/vite.config.ts` → `app/server/*.mjs`) — these ship together; the `/api/*` routes must run in **Node**, not the browser. | **Azure App Service (Linux, Node 20)** — or **Azure Container Apps** if you prefer containers. |
| **2. LLMs** | The reasoning + fast models the agents call. | **Azure AI Foundry** model deployments: **GPT-5.4** (reasoning) + a fast model (e.g. GPT-5.4-mini). |
| **3. Agent fleet** | The SRE investigation / resiliency / tuner / vuln agents. | **Azure AI Foundry Agent Service** (project + agents), invoked by the backend. |

**Supporting Azure services:**

| Concern | Azure service |
|---|---|
| Secrets (PATs, OAuth client secrets, API tokens) | **Azure Key Vault** |
| App identity (no secrets in code) | **Managed Identity** + **Microsoft Entra ID** |
| Container images (if using Container Apps) | **Azure Container Registry (ACR)** |
| Durable incident state / long-running orchestration (optional, prod) | **Azure Durable Functions** + **Cosmos DB** (see doc 05 §3) |
| TLS, custom domain, WAF | **App Service managed certs** or **Azure Front Door** |
| Logs & metrics | **Application Insights** / **Azure Monitor** |

> **Key principle (carried from the app's security model):** secrets **never** ship in the
> client bundle. All credentials are read **server-side** by the `/api/*` layer from environment
> variables / Key Vault. The browser only ever calls `/api/*`.

---

## 1. Prerequisites

- An **Azure subscription** with rights to create resource groups, App Service, and Azure AI Foundry.
- **Azure CLI** (`az`) ≥ 2.60, logged in: `az login`.
- **Node 20+** locally (to build the app).
- Quota for **GPT-5.4** in your chosen Azure region (check **Foundry → Quotas**; request an
  increase if needed).
- The integration credentials you want live (Dynatrace OAuth, GitHub PAT, Jira/Confluence API
  token, ServiceNow, Azure DevOps PAT, etc.) — gathered, *not* pasted into source.

```bash
# set once for the whole runbook
export RG=thalamus-sre-rg
export LOC=eastus2                 # pick a region with GPT-5.4 quota
export APP=thalamus-sre-app        # globally-unique App Service name
export PLAN=thalamus-sre-plan
export KV=thalamus-sre-kv          # globally-unique Key Vault name
export FOUNDRY=thalamus-foundry    # Foundry resource name

az group create -n $RG -l $LOC
```

---

## 2. Provision the LLMs — Azure AI Foundry + GPT-5.4

This creates the model the agents reason with.

1. **Create a Foundry (AI) resource and project.**
   ```bash
   # Foundry resource (Azure AI Services kind)
   az cognitiveservices account create \
     -n $FOUNDRY -g $RG -l $LOC \
     --kind AIServices --sku S0 \
     --custom-domain $FOUNDRY \
     --assign-identity
   ```
   Then in the **Azure AI Foundry portal** (`ai.azure.com`): create a **Project** inside this
   resource. Copy its **Project endpoint** —
   `https://<proj>.services.ai.azure.com/api/projects/<name>` — you'll paste this into the app
   later.

2. **Deploy the models.** In **Foundry → Deployments → + Deploy model**, deploy:
   - **`gpt-5.4`** → name the deployment **`gpt-54-reasoning`** (the reasoning class).
   - A fast model (e.g. **`gpt-5.4-mini`**) → name it **`gpt-54-fast`** (high-volume triage/classification).

   These two **deployment names** are what the app references (it pins to *logical* names so models
   swap without code changes — see doc 05 §1.1).

   ```bash
   az cognitiveservices account deployment create \
     -n $FOUNDRY -g $RG \
     --deployment-name gpt-54-reasoning \
     --model-name gpt-5.4 --model-version "latest" \
     --model-format OpenAI --sku-capacity 50 --sku-name GlobalStandard
   ```
   (Repeat for `gpt-54-fast` with the mini model.)

3. **Note the values** for later:
   - `FOUNDRY_PROJECT_ENDPOINT` = the project endpoint URL
   - `MODEL_REASONING` = `gpt-54-reasoning`
   - `MODEL_FAST` = `gpt-54-fast`

---

## 3. Provision the agent fleet — Foundry Agent Service

The agents (supervisor + lifecycle + golden-signal specialists + resiliency/vuln/tuner agents)
are registered in the same Foundry **project** and invoked **by the backend**.

- **For a first deployment**, the app runs its agent workflows against the Foundry **models**
  directly (chat/Responses calls) — the runtime badge flips to **Foundry-live** the moment the
  app has a valid project endpoint (see §6). This is enough to "begin using" the deployed app.
- **For the full agent-service topology** (registered versioned agents, connected sub-agents,
  durable orchestration, human-approval state machine), follow
  [`05-azure-ai-foundry-deployment.md`](05-azure-ai-foundry-deployment.md) §1–§3 — it specifies
  the 11 agents, the depth-2 connected-agent layout, and the Durable-Functions orchestrator that
  owns long-running incident state in Cosmos DB.

> **Recommendation:** ship phase 1 (models + in-app workflows, Foundry-live) first so users can
> start; layer in the registered Agent Service + durable orchestrator as phase 2.

---

## 4. Secrets — Azure Key Vault + Managed Identity

Never bake integration secrets into the image or App Service plaintext settings.

1. **Create the vault and store secrets:**
   ```bash
   az keyvault create -n $KV -g $RG -l $LOC

   # examples — store every integration secret you plan to use live
   az keyvault secret set --vault-name $KV --name dynatrace-client-secret --value "<...>"
   az keyvault secret set --vault-name $KV --name github-pat            --value "<...>"
   az keyvault secret set --vault-name $KV --name jira-api-token         --value "<...>"
   az keyvault secret set --vault-name $KV --name confluence-api-token   --value "<...>"
   az keyvault secret set --vault-name $KV --name ado-pat                --value "<...>"
   az keyvault secret set --vault-name $KV --name dt-webhook-secret      --value "<long-random>"
   ```

2. **Grant the App Service managed identity read access** (done in §5 after the app exists).

> The `/api/*` server modules read these as **environment variables** at runtime; App Service
> injects them from Key Vault via **Key Vault references** (§5), so the app code stays unchanged.

---

## 5. Deploy the UI + backend — Azure App Service (Linux, Node 20)

The Vite SPA and its `/api/*` Node middleware deploy as **one** App Service.

### 5a. Build

```bash
cd app
npm ci
npm run build         # tsc -b && vite build  → produces app/dist (static) + the server/*.mjs run in Node
```

> **Important — the `/api/*` layer must run in Node.** In local dev these routes are served by the
> Vite dev-server middleware (`vite.config.ts`). For production you serve `app/dist` as static files
> **and** run a small Node process that mounts the same `app/server/*.mjs` handlers under `/api/*`
> (a thin Express/Connect adapter that reuses the existing handlers). Bundle that Node entrypoint
> with the app. (If you have not yet added the production server entry, create
> `app/server.mjs` that serves `dist/` and wires each `/api/*` route to its `server/*.mjs` handler —
> the handlers already export plain `(body) => result` functions.)

### 5b. Create the App Service

```bash
az appservice plan create -n $PLAN -g $RG --is-linux --sku B2
az webapp create -n $APP -g $RG -p $PLAN --runtime "NODE:20-lts"
az webapp identity assign -n $APP -g $RG          # system-assigned managed identity
```

### 5c. Grant Key Vault + Foundry access to the app identity

```bash
APP_PRINCIPAL=$(az webapp identity show -n $APP -g $RG --query principalId -o tsv)

# Key Vault: secret read
az keyvault set-policy -n $KV --object-id $APP_PRINCIPAL --secret-permissions get list

# Foundry: let the app call the project with its managed identity (no API key needed)
az role assignment create --assignee $APP_PRINCIPAL \
  --role "Cognitive Services User" \
  --scope $(az cognitiveservices account show -n $FOUNDRY -g $RG --query id -o tsv)
```

### 5d. App settings (env vars + Key Vault references)

```bash
az webapp config appsettings set -n $APP -g $RG --settings \
  NODE_ENV=production \
  FOUNDRY_PROJECT_ENDPOINT="$FOUNDRY_PROJECT_ENDPOINT" \
  MODEL_REASONING="gpt-54-reasoning" \
  MODEL_FAST="gpt-54-fast" \
  DT_WEBHOOK_SECRET="@Microsoft.KeyVault(VaultName=$KV;SecretName=dt-webhook-secret)" \
  GITHUB_PAT="@Microsoft.KeyVault(VaultName=$KV;SecretName=github-pat)" \
  JIRA_API_TOKEN="@Microsoft.KeyVault(VaultName=$KV;SecretName=jira-api-token)" \
  CONFLUENCE_API_TOKEN="@Microsoft.KeyVault(VaultName=$KV;SecretName=confluence-api-token)" \
  ADO_PAT="@Microsoft.KeyVault(VaultName=$KV;SecretName=ado-pat)" \
  DYNATRACE_CLIENT_SECRET="@Microsoft.KeyVault(VaultName=$KV;SecretName=dynatrace-client-secret)"
```

### 5e. Deploy the build

```bash
# zip the built app (dist + server + node entry + package.json) and push
az webapp deploy -n $APP -g $RG --type zip --src-path ./thalamus-sre.zip
# set the startup command to your Node server entry
az webapp config set -n $APP -g $RG --startup-file "node server.mjs"
```

The app is now at `https://$APP.azurewebsites.net`.

> **Container alternative.** Prefer containers? Build the image, push to **ACR**, and deploy to
> **Azure Container Apps** instead of App Service — same env vars, same managed-identity + Key Vault
> wiring. Use ACR for the image, Container Apps for the workload, Key Vault for secrets.

---

## 6. Point the app at Foundry + GPT-5.4 (make the agents "Foundry-live")

Two equivalent paths:

- **Via env (recommended for prod):** the App Service settings in §5d already set
  `FOUNDRY_PROJECT_ENDPOINT`, `MODEL_REASONING`, `MODEL_FAST`. The backend uses its **managed
  identity** to call Foundry — no key in the browser.
- **Via the UI (Configure Integrations):** open the deployed app → **Application Resiliency →
  Configure Integrations → Azure AI Foundry**, set:
  - **Project endpoint** = `https://<proj>.services.ai.azure.com/api/projects/<name>`
  - **Reasoning model deployment** = `gpt-54-reasoning`
  - **Fast model deployment** (if shown) = `gpt-54-fast`
  - **Entra tenant id** = your tenant
  Enable it. The runtime badge flips from **Ollama** to **Foundry-live**
  (`agentRuntimeBadge()` returns `FoundryLive` once a project endpoint is set), and the agents now
  reason on **GPT-5.4**.

> The app falls back to deterministic stubs / local Ollama when Foundry isn't configured — so a
> half-configured deploy still renders; it just won't run live GPT-5.4 reasoning until §6 is done.

---

## 7. Connect the integrations (so users can actually use it)

All integrations are connected on **one page**: **Application Resiliency → Configure Integrations**.
Each card has a **▶ Test connection** button — use it to validate live before relying on it.

**Connect these to light up the core product:**

| Integration | Azure/where | What it enables | Notes |
|---|---|---|---|
| **Dynatrace** | external | Golden signals, SLOs, Davis problems (the "what's happening") | OAuth client-credentials; needs Grail/DQL scopes. Optional **push** webhook. |
| **GitHub** | external | Repo reads for audits + code-cause; PRs | PAT with repo read (+ write for PRs). |
| **ServiceNow** | external | Incident/change system of record | — |
| **Jira** | external | Live issue creation from gap tickets/investigations | API token + **project key** (e.g. `SRE`); "List my Jira projects" helps find it. |
| **Confluence** | external | Publish docs (6 types) + two-way KnowledgeHub sync | Base URL ending in `/wiki` + space key. |
| **Azure DevOps** | external | Live work-item writes | PAT with Work Items read/write. |
| **AWS** | external | Cloud resiliency checks | Server-side; scoped IAM creds. |
| **Azure AI Foundry** | Azure (§2,§6) | The GPT-5.4 agent runtime | Project endpoint + deployment names. |
| **Slack** | external | Escalation notifications | Optional. |

> Every surface shows a **`SourceBadge`** (live vs. mock) so users always know whether they're
> looking at real data. The **Integration Call Log** (bottom of Configure Integrations) records
> every request/response for debugging — secrets redacted.

**Dynatrace push (optional):** if you want Dynatrace to *push* SLO/problem alerts, register
`https://$APP.azurewebsites.net/api/dynatrace-webhook` in Dynatrace with the shared
`DT_WEBHOOK_SECRET`. See [`dynatrace-webhook-setup.md`](dynatrace-webhook-setup.md). (Inbound, so it
needs the public App Service URL — no ngrok in cloud.)

---

## 8. First-run checklist (verify the deploy)

1. Browse `https://$APP.azurewebsites.net` → the **Thalamus SRE** UI loads.
2. **Configure Integrations** → Azure AI Foundry shows **enabled**; badge reads **Foundry-live**.
3. Run a quick agent action (e.g. **Resiliency Auditor** on a small repo, or **Investigation** on a
   service) → confirm it reasons (GPT-5.4) rather than falling back to stubs.
4. **Test connection** passes for Dynatrace / GitHub / Jira / Confluence / ADO as configured.
5. Pull live Dynatrace signals on the **Dashboard** → charts flip to **DT-live**.
6. Check **Integration Call Log** → calls recorded, no unexpected errors.
7. Confirm **Application Insights** is receiving logs/metrics.

---

## 9. Production hardening (recommended next)

- **Durable orchestration:** add **Azure Durable Functions** + **Cosmos DB** for the long-running
  incident state machine and human-approval waits (doc 05 §3). Foundry runs are short/synchronous
  and cannot own multi-hour approval state.
- **Networking:** put **Azure Front Door** (WAF) in front; restrict Foundry/Key Vault to a **VNet**
  with private endpoints.
- **Identity & RBAC:** front the app with **Entra ID auth** (App Service Easy Auth) and map users to
  the app's RBAC roles.
- **CI/CD:** GitHub Actions / Azure DevOps pipeline → build → push (ACR) → deploy (App Service /
  Container Apps), with dev/test/prod environments and Bicep/Terraform IaC.
- **Cost/scale:** scale the App Service plan (B2→P-series) under load; size GPT-5.4
  capacity (TPM) from real usage; set Foundry quota alerts.
- **Observability:** Application Insights dashboards + Azure Monitor alerts on error rate, latency,
  and model spend.

---

## 10. Quick reference — Azure service ↔ responsibility

| You need to… | Use this Azure service |
|---|---|
| Host the UI **and** `/api/*` backend | **App Service (Linux, Node 20)** or **Container Apps + ACR** |
| Run the LLMs (GPT-5.4 reasoning + fast) | **Azure AI Foundry** model deployments |
| Host/register the agent fleet | **Azure AI Foundry Agent Service** (project + agents) |
| Keep secrets out of code | **Key Vault** + **Managed Identity** (+ Entra ID) |
| Long-running incident state / approvals | **Durable Functions** + **Cosmos DB** |
| Logs, metrics, alerts | **Application Insights** / **Azure Monitor** |
| TLS, custom domain, WAF | **App Service certs** / **Azure Front Door** |
