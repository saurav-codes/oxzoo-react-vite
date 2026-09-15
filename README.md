# oxzoo-react-vite

An official ox deploy example: a React 18 single-page app built with Vite 5, backed by an Express 4 API, deployed to a single Ubuntu VPS by the [ox](https://github.com/saurav-codes/vps-ctl) control plane from one `ox.toml` manifest at the repo root. ox runs the install and build steps, starts `node server/index.js` as a systemd process, and configures nginx to serve the built `dist/` folder statically while proxying only `/api` and `/health` to the Node process.

## Stack

| Layer | Tool | Role |
|---|---|---|
| Frontend | React 18 + Vite 5 | SPA built to `dist/`, served by nginx |
| API | Express 4 | `GET /api/greeting` and `GET /health`, binds `127.0.0.1:9102` |
| Package manager | pnpm 9 via corepack | lockfile (`pnpm-lock.yaml`) is committed |
| Deploy | ox | `ox.toml` defines processes, frontend, domain |

## Environment flow

One variable, two paths:

**`GREETING_TAG`**

- **Runtime path (API):** `server/index.js` reads `process.env.GREETING_TAG` on every request to `GET /api/greeting`. A restart with a new value is enough to change it.
- **Build-time path (SPA):** `vite.config.js` sets `envPrefix: ["GREETING_", "VITE_"]`, so any `GREETING_*` variable in the build environment is exposed to `import.meta.env`. `client/src/App.jsx` renders `import.meta.env.GREETING_TAG`, which is baked into the bundle during `pnpm run build`. No duplicated `VITE_GREETING_TAG` is needed.

**Set `GREETING_TAG` in the ox Environment editor BEFORE the first deploy.** The SPA value is baked during the deploy build step, so changing it later requires a redeploy; the API value updates as soon as the process restarts. `.env.example` documents the variable with a placeholder; real values live in the ox dashboard, never in git.

## Deploy with ox

1. Add the repo in the ox dashboard: paste the clone URL `https://github.com/saurav-codes/oxzoo-react-vite.git`.
2. In the Environment editor, set `GREETING_TAG` (for example `v1`).
3. Press **Deploy**. ox runs `corepack pnpm install --frozen-lockfile`, then `corepack pnpm run build`, starts `node server/index.js`, and waits for `http://127.0.0.1:9102/health` to return `ok`.

## Expected output

Visiting the domain shows the project heading plus the two labeled lines:

```
oxzoo-react-vite
frontend: hello world oxzoo-react-vite_<GREETING_TAG>
backend: hello world oxzoo-react-vite_<GREETING_TAG>
```

`<GREETING_TAG>` is whatever you set in the Environment editor. `backend:` shows `loading` until the fetch resolves, and an error message if `/api/greeting` fails.

## How nginx fits

ox configures nginx with `spa = true`: it serves `dist/` from the current release with `try_files $uri $uri/ /index.html`, so deep links fall back to the SPA entry. Only the `[frontend].api_paths` prefixes `/api` and `/health` are proxied to the web process on `127.0.0.1:9102`; everything else is static files.

## Local development

```bash
npx -y pnpm@9 install
GREETING_TAG=localtest npx -y pnpm@9 run build   # bakes GREETING_TAG into dist/
GREETING_TAG=localtest PORT=9102 node server/index.js
```

Pass env inline per the commands above; never commit a real `.env`.
