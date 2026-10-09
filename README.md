# oxzoo-react-vite

Deployed with [ox](https://deploywithox.com): deploy a repo to your own server with one command, no Docker. [Docs](https://deploywithox.com/docs) · [Guide for this stack](https://deploywithox.com/docs/guides/react-vite)

An [ox](https://deploywithox.com) deploy example: a React 18 SPA (Vite 5) with an Express 4 API and pnpm, deployed to your own Ubuntu server. systemd runs Express, and Caddy serves the built SPA with an `index.html` fallback while sending only `/api` and `/health` to Express, so one variable powers both halves of the demo.

## Stack

| Layer | Tool | Version |
| ----- | ---- | ------- |
| Frontend | React | 18 |
| Bundler | Vite | 5 |
| API | Express | 4 |
| Package manager | pnpm | 9.15.9 (`packageManager` in `package.json`) |
| Runtime | Node.js | 24 (ox's default; mise installs it) |

## ox.toml

```toml
# Express API + React SPA with pnpm (packageManager pins pnpm).

[app]
health = "/health"

[static]
dir = "dist"
spa = true
api = ["/api", "/health"]
```

ox detects `pnpm install --frozen-lockfile` from `pnpm-lock.yaml`, the pnpm version from `packageManager`, and `pnpm run build` and `pnpm run start` (`node server/index.js`) from `package.json`.

## Environment flow

1. **Run time (API):** `server/index.js` reads `process.env.GREETING_TAG` on every `GET /api/greeting` and returns `hello world oxzoo-react-vite_<GREETING_TAG>`.
2. **Build time (SPA):** `vite.config.js` sets `envPrefix: ["GREETING_", "VITE_"]`, so `client/src/App.jsx` reads `import.meta.env.GREETING_TAG` and Vite bakes it into `dist/`.

ox sets your variables before the build, and changing one with `ox vars set` redeploys, which rebuilds the SPA.

## Deploy with ox

```sh
curl -fsSL https://deploywithox.com/install.sh | sh
ox login
ox new https://github.com/saurav-codes/oxzoo-react-vite
printf 'GREETING_TAG=demo\n' | ox review oxzoo-react-vite --from-file - --wait
```

The plan, offline:

```console
$ ox check .
ox check . (manifest: ox.toml)

  app.start                  pnpm run start                                       detected:package.json
  app.health                 /health                                              declared
  static.dir                 dist                                                 declared
  static.spa                 true                                                 declared
  static.api                 /api, /health                                        declared
  build.install              pnpm install --frozen-lockfile                       detected:pnpm-lock.yaml
  build.commands[0]          pnpm run build                                       detected:package.json
  tools.node                 24                                                   default
  tools.pnpm                 9.15.9                                               detected:package.json

  Provided by ox: PORT, HOST, OX_ENV, OX_PROJECT, OX_RELEASE, OX_DATA_DIR, PUBLIC_URL, PUBLIC_HOST
  Set on the dashboard before the first deploy: GREETING_TAG

Ready to deploy.
```

## Expected output

```
oxzoo-react-vite
frontend: hello world oxzoo-react-vite_<GREETING_TAG>
backend: hello world oxzoo-react-vite_<GREETING_TAG>
```

## Local development

```sh
npx -y pnpm@9 install
GREETING_TAG=localtest npx -y pnpm@9 run build
GREETING_TAG=localtest PORT=9102 node server/index.js
```
