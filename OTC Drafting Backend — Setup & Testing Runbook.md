# OTC Drafting Backend — Setup & Testing Runbook

This runbook takes the OTC drafting backend from a fresh clone on your machine through to a fully deployed, Blob-connected App Service in the customer tenant. Follow the phases in order — each one assumes the previous passed. Where a value is environment-specific (an endpoint, an account name), it's marked `<LIKE THIS>` for your team to fill in.

## Phase A — Environment Setup (Install Prerequisites)

Run each check below one at a time; install only what's actually missing.

```powershell
py --version
git --version
az --version
```

If any show "not recognized", install via `winget` (no admin needed on most machines):

```powershell
winget install -e --id Python.Python.3.12
winget install -e --id Git.Git
```

If the Azure CLI installer asks for admin credentials you don't have, install it through `pip` instead — no admin required:

```powershell
py -m pip install --user azure-cli
```

After any install, **close and reopen the terminal** (PATH only refreshes on a new session), then re-run the three version checks.

If `az` still isn't found after reopening, find and permanently add its folder to PATH:

```powershell
$userScripts = (py -c "import site; print(site.USER_BASE)") + "\Scripts"
$currentUserPath = [Environment]::GetEnvironmentVariable("PATH", "User")
[Environment]::SetEnvironmentVariable("PATH", "$currentUserPath;$userScripts", "User")
```

Close and reopen the terminal once more, then confirm `az --version` works from any folder.

Finally, sign in:

```powershell
az login
```

## Phase B — Clone the Repo & Local Python Setup

```powershell
git clone <repo-url>
cd otc-drafting-backend
```

**If your project folder lives under OneDrive**, create the virtual environment in a separate, non-synced folder — OneDrive's real-time sync locks files faster than `pip` can write them, causing "Access is denied" errors. The project stays where it is; only the venv moves:

```powershell
py -m venv C:\otc-venv
C:\otc-venv\Scripts\Activate.ps1
```

(Otherwise, a normal `.venv` in the project folder is fine: `py -m venv .venv` then `.venv\Scripts\Activate.ps1`.)

Your prompt should now show the venv name at the start of the line. Install dependencies (run this from inside the project folder, so `requirements.txt` resolves):

```powershell
pip install -r requirements.txt
```

**If this fails with a permission/access-denied error even outside OneDrive**, check Windows Security → Protection history for a "blocked" entry — this is commonly a managed-device Attack Surface Reduction (ASR) rule or Controlled Folder Access policy blocking newly-created executable files, not a mistake in the setup. A non-admin user usually can't override this; ask IT to allow the specific rule/file it names.

Set up the environment file:

```powershell
Copy-Item .env.example .env
notepad .env
```

Fill in at minimum `AZURE_AI_PROJECT_ENDPOINT`. Leave `OTC_DATA_ROOT=./data` and `OTC_INDEX_BACKEND=local` for now — keep the first pass simple, no Blob or Azure AI Search involved yet.

## Phase C — Pre-flight Connectivity Checks (PowerShell)

Prove credentials and network reachability work **before** starting the full app — isolates "is my endpoint/auth right" from "is the pipeline logic right".

**Azure OpenAI / Foundry:**

```powershell
$endpoint   = "<AZURE_AI_PROJECT_ENDPOINT value>"
$deployment = "gpt-4o"
$token = az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv

$headers = @{ Authorization = "Bearer $token"; "Content-Type" = "application/json" }
$body = @{
    messages   = @(@{ role = "user"; content = "Reply with exactly: connection OK" })
    max_tokens = 10
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "$endpoint/openai/deployments/$deployment/chat/completions?api-version=2024-08-01-preview" -Method POST -Headers $headers -Body $body
```

A response of `"connection OK"` confirms the endpoint, deployment name, and signed-in identity's access are all correct.

If this fails with the connection being closed/reset (not an auth error), the endpoint is likely behind a Private Endpoint reachable only from inside the customer VNet — confirm whether this machine is actually on the customer VPN before troubleshooting further.

**Azure AI Search** (only if `OTC_INDEX_BACKEND=azure_search` — skip for `local`):

```powershell
$searchEndpoint = "<AI_SEARCH_ENDPOINT value>"
$token = az account get-access-token --resource https://search.azure.com --query accessToken -o tsv
Invoke-RestMethod -Uri "$searchEndpoint/indexes?api-version=2024-07-01" -Headers @{ Authorization = "Bearer $token" }
```

A `403` here (vs. a connection failure) means the network path is fine but the signed-in identity lacks a data-plane role on the Search resource — check **Search service → Access control (IAM)** for a `Search Index Data Reader` role, or confirm whether the resource requires an API key instead of AAD auth.

## Phase D — Local Execution with Sample/Dummy Data

**Sample data layout** (matches the `documents` field names the API expects):

```powershell
mkdir C:\otc-local-test\samples, C:\otc-local-test\CASE123
# put im.pdf, financial_model.xlsx, gdg_policy.pdf, tas.pdf under CASE123
# put BLANK OTC TEMPLATE.docx under samples
$env:OTC_DATA_ROOT = "C:\otc-local-test"
```

**Start the server:**

```powershell
aicoe-agents serve config.yaml
```

(Confirm this command against the real repo's README — it may differ.)

**Health check:**

```powershell
Invoke-RestMethod -Uri "http://localhost:8000/health" -Method GET
```

**Non-streaming test call:**

```powershell
$body = @{
    case_id          = "CASE123"
    opportunity_name = "Local Test"
    documents        = @{
        im              = "CASE123/im.pdf"
        financial_model = "CASE123/financial_model.xlsx"
        gdg             = "CASE123/gdg_policy.pdf"
        otc_template    = "samples/BLANK OTC TEMPLATE.docx"
    }
    include_tas = $false
    resume      = $false
    save_docx   = $true
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "http://localhost:8000/query/payload" -Method POST -Body $body -ContentType "application/json"
```

Check every field of the response is populated: `otc_draft`, `im_gaps`, `financial_gaps`, `gdg_analysis`, `markdown`, `output_docx_path`, `elapsed_seconds`. Open the generated `.docx` and confirm it actually rendered.

**Streaming variant** (PowerShell buffers the whole response rather than streaming it — use `curl.exe`):

```powershell
curl.exe -N -X POST "http://localhost:8000/query/payload/stream" -H "Content-Type: application/json" -d $body
```

**Resume test** — call again with `"resume": true` on the same `case_id`; it should be noticeably faster, confirming checkpointing works.

**Edge-case checks**: an invalid `case_id` (spaces, special characters) should return a clean validation error, not a crash. Omitting a required document on a brand-new `case_id` should return a clear "not ingested" error, not a silent failure.

## Phase E — Real Data Execution

Replace the files under `C:\otc-local-test\CASE123` with real (or realistically representative) IM, financial model, GDG policy, and TAS documents. Re-run the exact same calls from Phase D, but this time judge the **output quality**, not just "did it run":

- Are the identified gaps actually correct against the real documents?
- Does the GDG analysis read as sensible policy reasoning, not generic text?
- Does the drafted `.docx` match the real OTC template's layout — headings, tables, and the red "missing" formatting for blank fields?
- Check `elapsed_seconds` against realistic expectations. Watch specifically for a cold-start timeout — a sibling project hit exactly this (a \~57s cold start blew past a 60s timeout) even though the warm runtime was \~11s.
- Test `include_tas: true` and `false` both ways.
- Re-run with `resume: true` against real-sized content to confirm checkpointing still holds up.

## Phase F — Blob Storage & App Service Configuration

### F.1 — Create the Blob container and folder structure

AAD-authenticated, no account key or SAS:

```powershell
$storageAccount = "<storage-account-name>"

az storage container create --account-name $storageAccount --name otc-data --public-access off --auth-mode login

az storage blob upload --account-name $storageAccount --container-name otc-data --name "samples/BLANK OTC TEMPLATE.docx" --file "<local path>" --auth-mode login
```

Structure, matching the `documents` contract exactly — case folders are created automatically the moment a file is uploaded to them, nothing to pre-create:

```
otc-data/
├── samples/
│   └── BLANK OTC TEMPLATE.docx
└── <case_id>/
    ├── im.pdf
    ├── financial_model.xlsx
    ├── gdg_policy.pdf
    └── tas.pdf
```

`<case_id>` must match `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`.

### F.2 — App Service → Configuration → Path mappings → Azure Storage Mount

| Field | Value |
| --- | --- |
| Name | `otcdata` |
| Configuration options | Basic |
| Storage type | Azure Blob Container |
| Account name | the real storage account |
| Share name *(this field is mislabeled — it means container name)* | `otc-data` |
| Access key | the account's key — **only works if "Allow storage account key access" is enabled** on the storage account; check this first, Storage Mounts does not support managed identity |
| Mount path | `/mnt/otcdata` |

**Check the mount's read-only toggle.** If read-only, source document reads work but the drafted `.docx` write-back will fail at runtime — verify explicitly, don't assume read-write.

### F.3 — Application settings

```
AZURE_AI_PROJECT_ENDPOINT = <real Azure AI Foundry project endpoint>
OTC_AGENT_TYPE            = FOUNDRY
OTC_DATA_ROOT             = /mnt/otcdata
OTC_TEMPLATE_PATH         = samples/BLANK OTC TEMPLATE.docx
OTC_INDEX_BACKEND         = local
```

`OTC_DATA_ROOT` must exactly match the mount path in F.2 — that's the one line connecting the mount to the code.

### F.4 — Managed identity

Only needed if using the code-change route (`DefaultAzureCredential` in `resolve_under_root()`) instead of the key-based Storage Mount — grant `Storage Blob Data Contributor` on the storage account. **Not needed for the Storage Mount route**, which authenticates with the raw key instead of RBAC.

### F.5 — Before exposing this

Enable `auth_config.enabled` in `config.yaml` (confirmed `false` by default) before this endpoint is reachable by Bot Service or anything else on the VNet.

## Phase G — End-to-End Test Against the Deployed App Service

Re-run every Phase D and Phase E test against the live App Service URL instead of `localhost`, plus these deployment-specific checks:

**Health check against the live URL:**

```powershell
Invoke-RestMethod -Uri "https://<app-name>.azurewebsites.net/health" -Method GET
```

**Concurrency / lock test** — fire two overlapping requests for the *same* `case_id` at the same time. The second should get a `CaseBusyError`, not silent data corruption. Remember: this lock is a local OS file lock, so it only actually works while this App Service runs as a single instance — if it's ever scaled to multiple instances, each instance has its own disk and the lock stops providing real exclusivity.

**Auth enforcement test** — call the endpoint with no token at all. Expect a clean `401`/`403`, not open access. If this succeeds without a token, `auth_config.enabled` isn't actually taking effect — stop and fix before this goes anywhere near production traffic.

**Full Blob round trip** — upload a real document directly to the Blob container (as Bot Service would), call the deployed API referencing it, confirm the draft `.docx` lands back in Blob at the expected path.

## Known Issues & Caveats

- **`pip install` blocked by Windows Security ("risky action blocked")** — likely a managed-device Attack Surface Reduction rule or Controlled Folder Access policy, not a local misconfiguration. A non-admin user can't resolve this alone; needs IT to allow the specific rule/file.
- **VNet / Private Endpoint reachability is an open question** — whether the machine running these tests, and/or the deployed App Service, actually has a routed path to the Foundry and Blob Private Endpoints. Confirm this explicitly before assuming any connectivity failure is a code or config bug.
- **Azure Storage Mounts authenticate with an account key, not managed identity** — a real limitation of this specific App Service feature. If the storage account has key-based access disabled (common on security-hardened accounts), this mount approach is structurally unusable, and the fallback is the small code change extending `resolve_under_root()` to use `DefaultAzureCredential` directly.
- **The per-case lock (`CaseStore.lock()`) is a local OS file lock** — it does not coordinate across multiple App Service instances. Fine for a single instance; a real risk if this ever scales out.
- **Confirm whether you're testing against the sandbox tenant or the real customer environment** — the answers to "what's reachable" and "what's configured" differ meaningfully between the two, and conflating them mid-troubleshooting wastes time.
