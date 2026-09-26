# Railway deployment (OmniRoute)

This is a deployment guide for the `railway-deploy` branch. Railway should build the upstream root `Dockerfile`. No application code is changed here.

## Create services

1. In one Railway project and environment, create an OmniRoute service from GitHub repo `bhagyabanghadai/OmniRoute`, branch `railway-deploy`. Use the repository root and Dockerfile. Leave the image's default start command unchanged.
2. Attach a Railway Volume mounted at `/app/data`. This persists SQLite settings, provider connections, and API keys. Keep one OmniRoute replica per SQLite volume.
3. Add a Redis service in this Railway project (Railway managed Redis is suitable). Name it `Redis`, then set OmniRoute variable `REDIS_URL=${{Redis.REDIS_URL}}` using Railway's reference-variable picker.
4. Add these OmniRoute service variables in Railway (generate unique secrets, do not commit them):
   - `REQUIRE_API_KEY=true`
   - `JWT_SECRET=<unique random secret>`
   - `API_KEY_SECRET=<unique random secret>`
   - `INITIAL_PASSWORD=<long unique initial dashboard password>`
   - `STORAGE_ENCRYPTION_KEY=<unique random secret>`
   - `STORAGE_ENCRYPTION_KEY_VERSION=v1`
   - `DATA_DIR=/app/data`
   - `OMNIROUTE_ALLOW_PRIVATE_PROVIDER_URLS=true`
   - `OMNIROUTE_ALLOW_LOCAL_PROVIDER_URLS=true`
   - `REDIS_URL` reference described above
   Let Railway provide the runtime `PORT`; do not set a conflicting port. `OMNIROUTE_ALLOW_PRIVATE_PROVIDER_URLS` is needed only because the FreeLLMAPI custom provider node uses Railway private networking. Keep the dashboard admin-only and the API key requirement enabled.
5. Set the Railway healthcheck path to `/healthz`. Generate a public Railway domain for OmniRoute; Railway provides HTTPS. Later you can attach `ai.xbandglobal.com` in its Networking settings and configure the displayed DNS records.

## Add FreeLLMAPI as a private provider

Deploy the FreeLLMAPI `railway-deploy` branch in the same Railway project/environment, attach its volume, and complete first-time admin setup. Remove its public domain after setup. In OmniRoute's admin dashboard:

1. Add an OpenAI-compatible provider node with prefix `freellm`, type `chat`, and base URL `http://<FreeLLMAPI RAILWAY_PRIVATE_DOMAIN>:<FreeLLMAPI PORT>/v1`.
2. Use the FreeLLMAPI unified key as the provider connection credential. Keep this key private to OmniRoute.
3. Import/check the model list and use the exact returned model ID when creating combos/fallback order.

Railway private networking requires both services to share the same project and environment. Use the service's actual `RAILWAY_PRIVATE_DOMAIN` and `PORT` values in the provider URL; internal traffic uses `http://`.

## Create XBand's gateway key

In OmniRoute's API Manager, create a dedicated inference key for XBand, require authentication, restrict it to the needed models/combos, and set per-key rate limits/no-log options. Put that plaintext key only in XBand's backend secret store. XBand connects to the public HTTPS base URL `https://<OmniRoute Railway domain>/v1`; the browser must call XBand's backend, never OmniRoute or a provider directly.

## Build/runtime notes

This project is large and its Dockerfile does a full production build with native dependencies and a substantial Next.js bundle. A Railway build can fail if the builder runs out of memory; inspect build logs before changing the source. Use a Railway plan/builder with enough memory for the upstream build. At runtime, allow enough memory for the app, and watch Railway metrics. Keep the SQLite volume mounted and do not scale to multiple replicas sharing one SQLite database.

Railway configuration (volume, Redis reference, healthcheck, domain, variables) is set in the Railway dashboard. Railway's legacy `railway.json` config-as-code is deprecated for new services, so this branch does not add a misleading config file.
