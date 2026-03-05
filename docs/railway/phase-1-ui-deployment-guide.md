# Railway Phase 1 Deployment Guide (UI Steps)

This guide deploys this repo as a **single Railway service** with:

- one public domain,
- one persistent volume at `/data`,
- UI + REST + MCP exposed on the same public listener.

It assumes the phase-1 code changes in this repo are present (including `railway.toml` at repo root).

---

## 1) Create or Open the Railway Service

1. Open Railway and go to your target project.
2. If the service does not exist yet:
   - Click `+ New` and choose `GitHub Repo`.
   - Select this repository.
3. Open the service card/panel for this repo.

Note:
- Railway auto-detects root `Dockerfile`.
- Root `railway.toml` is also auto-detected for deploy/build overrides.

---

## 2) Ensure Config-as-Code Is Being Used

1. In the service settings, confirm the source is this repo/branch.
2. Deploy once (or redeploy latest commit) so Railway reads `railway.toml`.
3. Open the deployment details page.
4. Verify deploy settings have the file-source indicator (config from code), especially:
   - `startCommand`
   - `healthcheckPath`
   - `healthcheckTimeout`

Expected from this repo:
- `startCommand = "muninndb-server --daemon --data /data"`
- `healthcheckPath = "/mcp/health"`
- `healthcheckTimeout = 300`

---

## 3) Add Persistent Storage (Required)

1. On the project canvas, create a volume:
   - Right-click canvas and create volume, or
   - Use command palette (`⌘K`) and create volume.
2. Attach the volume to the MuninnDB service.
3. Set mount path to:

```txt
/data
```

4. Apply changes/deploy.

Why:
- MuninnDB data (Pebble/WAL/auth secret) must persist between deploys.

---

## 4) Set Service Variables

Open the service `Variables` tab and add:

Required:

```env
MUNINN_MCP_TOKEN=<strong-random-token>
```

Recommended optional variables (choose what you use):

```env
MUNINN_OPENAI_KEY=...
MUNINN_ENRICH_URL=openai://gpt-4o-mini
MUNINN_ENRICH_API_KEY=...
MUNINN_CORS_ORIGINS=https://your-frontend.example
```

Important:
- Variable edits create staged changes; click `Deploy` to apply.

---

## 5) Generate Public Domain

1. Go to service `Settings` -> `Networking` -> `Public Networking`.
2. Click `Generate Domain`.
3. Copy the generated `*.up.railway.app` domain.

If you need a custom domain:
- Add via `+ Custom Domain` and complete CNAME verification.

---

## 6) Verify Health and Endpoints

Use your generated domain:

```bash
export APP_URL="https://<your-service>.up.railway.app"

curl -i "$APP_URL/mcp/health"
curl -i "$APP_URL/api/health"
```

Expected:
- both return HTTP `200`.

Open in browser:

```txt
https://<your-service>.up.railway.app/
```

---

## 7) Verify MCP Auth

For MCP calls, include the token:

```bash
curl -i "$APP_URL/mcp/tools" \
  -H "Authorization: Bearer $MUNINN_MCP_TOKEN"
```

Without token, protected MCP endpoints should return unauthorized.

---

## 8) Persistence Smoke Test

1. Write some data through UI/API/MCP.
2. Redeploy the service.
3. Confirm data still exists.

Note:
- Railway docs indicate services with attached volumes can have brief downtime on redeploy to avoid data corruption.

---

## 9) Troubleshooting Checklist

1. Build fails with Dockerfile `VOLUME` banned:
   - Ensure deployment includes the commit where `VOLUME ["/data"]` was removed.

2. Healthcheck fails:
   - Confirm `healthcheckPath` is `/mcp/health`.
   - Confirm service is listening on Railway `PORT` (phase-1 code handles this via UI listener).
   - Check service logs for bind/startup errors.

3. Domain works but API/MCP 404:
   - Confirm you deployed code containing MCP mount on UI listener.
   - Re-test `/mcp/health` and `/api/health`.

4. Data missing after redeploy:
   - Confirm volume is attached to the same service.
   - Confirm mount path is exactly `/data`.

5. MCP unauthorized:
   - Confirm `MUNINN_MCP_TOKEN` is set on the service.
   - Confirm client sends `Authorization: Bearer <token>`.

---

## References

- Railway config as code (overview): https://docs.railway.com/deploy/config-as-code
- Railway config as code (reference): https://docs.railway.com/config-as-code/reference
- Railway healthchecks: https://docs.railway.com/deployments/healthchecks
- Railway volumes: https://docs.railway.com/volumes
- Railway public networking: https://docs.railway.com/networking/public-networking
- Railway domains: https://docs.railway.com/networking/domains
- Railway variables: https://docs.railway.com/variables
