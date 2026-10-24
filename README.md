# ServiceNow E2E Integration — React + Express

An end-to-end plan and reference implementation for managing ServiceNow records (Incidents, Change Requests, Service Catalog Requests, Tasks, Attachments) from a React application, backed by an Express server that lives in the same repository and serves the React build in production.

---

## 1. Goals

| Goal | Description |
|---|---|
| Full lifecycle | Create, read, update, resolve, close, and comment on ServiceNow records from the React UI |
| Secure | ServiceNow credentials and tokens never reach the browser; Express is the only caller of ServiceNow |
| Real-time | ServiceNow pushes changes to Express (webhooks); Express streams them to React (SSE) |
| Single deployable | One Node process serves both the API (`/api/*`) and the React build |
| Observable | Structured logs, correlation IDs, health checks, audit trail |

---

## 2. Architecture

```
            ┌────────────────────────────────┐
            │      React (Vite) SPA          │
            │  - React Router                │
            │  - TanStack Query              │
            │  - EventSource (SSE)           │
            └────────────────────────────────┘
                 │                     ▲
   HTTPS /api/*  │                     │  SSE /api/events
   (session)     │                     │  (live updates)
                 ▼                     │
            ┌────────────────────────────────┐
            │        Express server          │
            │  - Auth (session / SSO)        │
            │  - ServiceNow client           │
            │  - Token cache + refresh       │
            │  - Webhook receiver            │
            │  - Serves client/dist          │
            └────────────────────────────────┘
                 │                     ▲
 HTTPS + OAuth   │                     │  POST /webhooks/servicenow
 Table API,      │                     │  (Business Rule →
 Attachment API, │                     │   Outbound REST, HMAC signed)
 Catalog API     ▼                     │
            ┌────────────────────────────────┐
            │      ServiceNow instance       │
            │  - Table API                   │
            │  - Business Rules              │
            │  - Outbound REST               │
            └────────────────────────────────┘
```

**Flow summary**

1. User signs in to the app (session cookie, or SSO via OIDC).
2. React calls `/api/...` on the same origin — no CORS in production.
3. Express validates the session, calls ServiceNow with a cached OAuth token, normalizes the response, returns JSON.
4. When a record changes in ServiceNow, a Business Rule fires an Outbound REST Message to `/webhooks/servicenow`.
5. Express verifies the signature and broadcasts the event over SSE; React Query invalidates the affected cache.

---

## 3. Project Structure

```
servicenow-e2e/
├── client/                      # React app (Vite)
│   ├── src/
│   │   ├── api/                 # fetch wrapper + typed endpoints
│   │   │   ├── http.ts
│   │   │   └── incidents.ts
│   │   ├── hooks/               # TanStack Query hooks + useServerEvents
│   │   ├── pages/               # IncidentList, IncidentDetail, ChangeList, CatalogOrder
│   │   ├── components/          # Forms, tables, status badges, attachment uploader
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── index.html
│   ├── tsconfig.json
│   └── vite.config.ts           # root=client, dev proxy /api -> Express
├── server/                      # Express app
│   ├── src/
│   │   ├── config.ts            # env validation (zod)
│   │   ├── servicenow/
│   │   │   ├── auth.ts          # OAuth token fetch + cache + refresh
│   │   │   ├── client.ts        # axios instance, retries, error mapping
│   │   │   └── tables.ts        # generic Table API helpers
│   │   ├── routes/
│   │   │   ├── incidents.ts
│   │   │   ├── changes.ts
│   │   │   ├── catalog.ts
│   │   │   ├── attachments.ts
│   │   │   ├── events.ts        # SSE stream
│   │   │   └── webhooks.ts      # inbound from ServiceNow
│   │   ├── middleware/          # auth, error handler, rate limit, request id
│   │   └── index.ts             # app bootstrap + static serving of client/dist
│   └── tsconfig.json
├── .env.example
├── package.json                 # ONE common package.json for client + server
└── README.md
```

---

## 4. ServiceNow Setup (Instance Side)

### 4.1 Integration user
1. Create user `svc_react_integration` (System Security → Users).
2. Check **Web service access only**.
3. Assign least-privilege roles, e.g. `itil` (incidents/changes), `catalog` (catalog requests), plus a custom role if you restrict tables via ACLs.

### 4.2 OAuth application
1. **System OAuth → Application Registry → New → Create an OAuth API endpoint for external clients.**
2. Note the **Client ID** and **Client Secret**.
3. Choose a grant type:

| Grant | When to use | Notes |
|---|---|---|
| `client_credentials` | Server-to-server, actions run as the integration user | Supported on recent releases; may require enabling the client-credentials system property and mapping an OAuth application user |
| `authorization_code` | Actions must run as the signed-in end user (their ACLs, their name on audit) | Requires redirect URI `https://<app>/api/auth/servicenow/callback` |
| `password` | Legacy / PoC only | Avoid in production |

**Recommendation:** start with `client_credentials` and record the real end user in a field (e.g. `caller_id` / `opened_by` lookup by email). Move to `authorization_code` if audit requires the true user identity.

### 4.3 APIs used

| Purpose | Endpoint |
|---|---|
| CRUD on any table | `/api/now/table/{table}` |
| Attachments | `/api/now/attachment`, `/api/now/attachment/file` |
| Catalog ordering | `/api/sn_sc/servicecatalog/items/{sys_id}/order_now` |
| Aggregates / dashboards | `/api/now/stats/{table}` |
| Comments / work notes | `PATCH /api/now/table/incident/{sys_id}` with `comments` / `work_notes` |

### 4.4 Outbound notifications (ServiceNow → Express)
1. **System Web Services → Outbound → REST Message**: `React App Webhook`, endpoint `https://<app>/webhooks/servicenow`, method POST.
2. Add HTTP header `X-SN-Signature` (HMAC computed in the script below).
3. **Business Rule** on `incident` (and `change_request`, `sc_req_item`): *After* insert/update, async.

```javascript
// Business Rule script (server-side, ServiceNow)
(function executeRule(current, previous) {
  var payload = JSON.stringify({
    table: current.getTableName(),
    sys_id: current.getUniqueValue(),
    number: current.getValue('number'),
    state: current.getValue('state'),
    operation: current.operation(),
    updated_on: current.getValue('sys_updated_on')
  });
  var secret = gs.getProperty('x_react.webhook_secret');
  var sig = new GlideDigest().getSHA256Hex(secret + payload); // or HMAC via script include
  var r = new sn_ws.RESTMessageV2('React App Webhook', 'post');
  r.setRequestHeader('X-SN-Signature', sig);
  r.setRequestBody(payload);
  r.executeAsync();
})(current, previous);
```

Keep payloads small (IDs + state). React re-fetches the full record through Express so ACLs and normalization stay in one place.

---

## 5. Environment Variables

`.env.example`

```bash
# Server
PORT=4000
NODE_ENV=development
SESSION_SECRET=change-me
APP_BASE_URL=http://localhost:5173

# ServiceNow
SN_INSTANCE_URL=https://your-instance.service-now.com
SN_CLIENT_ID=xxxxxxxx
SN_CLIENT_SECRET=xxxxxxxx
SN_GRANT_TYPE=client_credentials        # or authorization_code
SN_WEBHOOK_SECRET=shared-secret-for-hmac
SN_TIMEOUT_MS=15000
```

Never commit `.env`. In production, load secrets from a vault (Azure Key Vault, AWS Secrets Manager, etc.).

---

## 6. Server Implementation

### 6.1 Install

All dependencies live in the single root `package.json` (see [Section 8](#8-common-packagejson)). Run `npm install` once from the repo root.

### 6.2 OAuth token cache — `server/src/servicenow/auth.ts`

```ts
import axios from 'axios';
import { config } from '../config';

let cached: { token: string; expiresAt: number } | null = null;
let inflight: Promise<string> | null = null;

export async function getAccessToken(): Promise<string> {
  if (cached && Date.now() < cached.expiresAt - 60_000) return cached.token;
  if (inflight) return inflight; // de-duplicate concurrent refreshes

  inflight = (async () => {
    const body = new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: config.SN_CLIENT_ID,
      client_secret: config.SN_CLIENT_SECRET,
    });
    const { data } = await axios.post(`${config.SN_INSTANCE_URL}/oauth_token.do`, body, {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    });
    cached = { token: data.access_token, expiresAt: Date.now() + data.expires_in * 1000 };
    return cached.token;
  })().finally(() => (inflight = null));

  return inflight;
}

export function invalidateToken() { cached = null; }
```

### 6.3 ServiceNow client — `server/src/servicenow/client.ts`

```ts
import axios, { AxiosError } from 'axios';
import { config } from '../config';
import { getAccessToken, invalidateToken } from './auth';

export const sn = axios.create({
  baseURL: `${config.SN_INSTANCE_URL}/api`,
  timeout: config.SN_TIMEOUT_MS,
  headers: { Accept: 'application/json' },
});

sn.interceptors.request.use(async (req) => {
  req.headers.Authorization = `Bearer ${await getAccessToken()}`;
  return req;
});

sn.interceptors.response.use(undefined, async (err: AxiosError) => {
  const cfg: any = err.config;
  const status = err.response?.status;

  if (status === 401 && !cfg._retried401) {          // expired token
    cfg._retried401 = true; invalidateToken(); return sn(cfg);
  }
  if ((status === 429 || (status && status >= 500)) && (cfg._retries ?? 0) < 3) {
    cfg._retries = (cfg._retries ?? 0) + 1;
    const retryAfter = Number(err.response?.headers['retry-after']) || 2 ** cfg._retries;
    await new Promise((r) => setTimeout(r, retryAfter * 1000));
    return sn(cfg);
  }
  throw err;
});
```

### 6.4 Generic table helpers — `server/src/servicenow/tables.ts`

```ts
import { sn } from './client';

const DEFAULT_PARAMS = { sysparm_display_value: 'all', sysparm_exclude_reference_link: 'true' };

export const table = (name: string) => ({
  list: (q: { query?: string; fields?: string; limit?: number; offset?: number }) =>
    sn.get(`/now/table/${name}`, {
      params: { ...DEFAULT_PARAMS, sysparm_query: q.query, sysparm_fields: q.fields,
                sysparm_limit: q.limit ?? 20, sysparm_offset: q.offset ?? 0 },
    }).then((r) => ({ items: r.data.result, total: Number(r.headers['x-total-count'] ?? 0) })),

  get: (id: string) => sn.get(`/now/table/${name}/${id}`, { params: DEFAULT_PARAMS }).then((r) => r.data.result),
  create: (body: object) => sn.post(`/now/table/${name}`, body).then((r) => r.data.result),
  update: (id: string, body: object) => sn.patch(`/now/table/${name}/${id}`, body).then((r) => r.data.result),
  remove: (id: string) => sn.delete(`/now/table/${name}/${id}`),
});
```

### 6.5 Incident routes — `server/src/routes/incidents.ts`

```ts
import { Router } from 'express';
import { z } from 'zod';
import { table } from '../servicenow/tables';

const r = Router();
const incidents = table('incident');

const createSchema = z.object({
  short_description: z.string().min(5).max(160),
  description: z.string().max(4000).optional(),
  urgency: z.enum(['1', '2', '3']).default('3'),
  impact: z.enum(['1', '2', '3']).default('3'),
  category: z.string().optional(),
});

r.get('/', async (req, res, next) => {
  try {
    const { state, q, page = '0', size = '20' } = req.query as Record<string, string>;
    const parts = ['ORDERBYDESCsys_updated_on'];
    if (state) parts.push(`state=${state}`);
    if (q) parts.push(`short_descriptionLIKE${q}^ORnumberLIKE${q}`);
    res.json(await incidents.list({
      query: parts.join('^'),
      fields: 'sys_id,number,short_description,state,priority,assigned_to,sys_updated_on',
      limit: +size, offset: +page * +size,
    }));
  } catch (e) { next(e); }
});

r.get('/:id', async (req, res, next) => {
  try { res.json(await incidents.get(req.params.id)); } catch (e) { next(e); }
});

r.post('/', async (req, res, next) => {
  try {
    const body = createSchema.parse(req.body);
    res.status(201).json(await incidents.create({ ...body, caller_id: req.session.user.snUserSysId }));
  } catch (e) { next(e); }
});

r.patch('/:id', async (req, res, next) => {
  try { res.json(await incidents.update(req.params.id, req.body)); } catch (e) { next(e); }
});

r.post('/:id/comments', async (req, res, next) => {
  try {
    const { text, internal } = z.object({ text: z.string().min(1), internal: z.boolean().default(false) }).parse(req.body);
    res.json(await incidents.update(req.params.id, internal ? { work_notes: text } : { comments: text }));
  } catch (e) { next(e); }
});

r.post('/:id/resolve', async (req, res, next) => {
  try {
    const { close_code, close_notes } = req.body;
    res.json(await incidents.update(req.params.id, { state: '6', close_code, close_notes }));
  } catch (e) { next(e); }
});

export default r;
```

> Sanitize user input before building `sysparm_query` — strip `^`, `^OR`, and `^NQ` from free-text values to prevent encoded-query injection.

### 6.6 Webhooks + SSE — `server/src/routes/webhooks.ts`, `events.ts`

```ts
// events.ts
import { Router, Response } from 'express';
const clients = new Set<Response>();
export const broadcast = (evt: object) =>
  clients.forEach((c) => c.write(`data: ${JSON.stringify(evt)}\n\n`));

export default Router().get('/', (req, res) => {
  res.set({ 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' });
  res.flushHeaders();
  clients.add(res);
  const ping = setInterval(() => res.write(': ping\n\n'), 25_000);
  req.on('close', () => { clearInterval(ping); clients.delete(res); });
});
```

```ts
// webhooks.ts
import { Router, raw } from 'express';
import crypto from 'crypto';
import { config } from '../config';
import { broadcast } from './events';

export default Router().post('/', raw({ type: '*/*' }), (req, res) => {
  const body = req.body.toString('utf8');
  const expected = crypto.createHash('sha256').update(config.SN_WEBHOOK_SECRET + body).digest('hex');
  const got = String(req.headers['x-sn-signature'] ?? '');
  if (got.length !== expected.length ||
      !crypto.timingSafeEqual(Buffer.from(got), Buffer.from(expected))) return res.sendStatus(401);
  broadcast(JSON.parse(body));
  res.sendStatus(202);
});
```

> For multiple server instances, fan out through Redis Pub/Sub instead of the in-memory `Set`.

### 6.7 Bootstrap & serving React — `server/src/index.ts`

```ts
import express from 'express';
import path from 'path';
import helmet from 'helmet';
import compression from 'compression';
import rateLimit from 'express-rate-limit';
import pinoHttp from 'pino-http';
import session from 'express-session';
import { config } from './config';
import incidents from './routes/incidents';
import changes from './routes/changes';
import catalog from './routes/catalog';
import attachments from './routes/attachments';
import events from './routes/events';
import webhooks from './routes/webhooks';
import { requireAuth } from './middleware/auth';
import { errorHandler } from './middleware/error';

const app = express();
app.use(helmet());
app.use(compression());
app.use(pinoHttp());

app.use('/webhooks/servicenow', webhooks);     // before json parser (needs raw body)

app.use(express.json({ limit: '1mb' }));
app.use(session({ secret: config.SESSION_SECRET, resave: false, saveUninitialized: false,
  cookie: { httpOnly: true, sameSite: 'lax', secure: config.NODE_ENV === 'production' } }));
app.use('/api', rateLimit({ windowMs: 60_000, max: 300 }));

app.get('/api/health', (_req, res) => res.json({ ok: true }));
app.use('/api/incidents', requireAuth, incidents);
app.use('/api/changes', requireAuth, changes);
app.use('/api/catalog', requireAuth, catalog);
app.use('/api/attachments', requireAuth, attachments);
app.use('/api/events', requireAuth, events);

// Serve React build
const dist = path.resolve(__dirname, '../../client/dist');
app.use(express.static(dist));
app.get(/^(?!\/api|\/webhooks).*/, (_req, res) => res.sendFile(path.join(dist, 'index.html')));

app.use(errorHandler);
app.listen(config.PORT, () => console.log(`Server on :${config.PORT}`));
```

### 6.8 Error mapping — `server/src/middleware/error.ts`

```ts
import { ErrorRequestHandler } from 'express';
import { AxiosError } from 'axios';
import { ZodError } from 'zod';

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  if (err instanceof ZodError) return res.status(400).json({ error: 'VALIDATION', details: err.issues });
  if (err instanceof AxiosError && err.response) {
    const status = err.response.status;
    const msg = (err.response.data as any)?.error?.message ?? 'ServiceNow error';
    req.log.warn({ status, msg }, 'servicenow_error');
    return res.status(status === 401 ? 502 : status).json({ error: 'SERVICENOW', message: msg });
  }
  req.log.error(err);
  res.status(500).json({ error: 'INTERNAL' });
};
```

---

## 7. Client Implementation

### 7.1 Install

No separate install — the client shares the root `package.json` and `node_modules`.

### 7.2 Vite config — `client/vite.config.ts`

Because there is only one `package.json` at the repo root, Vite is pointed at the `client/` folder explicitly.

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  root: path.resolve(__dirname),                 // client/
  plugins: [react()],
  build: { outDir: 'dist', emptyOutDir: true },  // -> client/dist (served by Express)
  server: {
    port: 5173,
    proxy: {
      '/api': 'http://localhost:4000',
      '/webhooks': 'http://localhost:4000',
    },
  },
});
```

### 7.3 API + hooks

```ts
// src/api/http.ts
export async function http<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, { credentials: 'include',
    headers: { 'Content-Type': 'application/json', ...init?.headers }, ...init });
  if (!res.ok) throw Object.assign(new Error((await res.json()).message ?? res.statusText), { status: res.status });
  return res.status === 204 ? (undefined as T) : res.json();
}

// src/hooks/useIncidents.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { http } from '../api/http';

export const useIncidents = (params: { state?: string; q?: string; page?: number }) =>
  useQuery({
    queryKey: ['incidents', params],
    queryFn: () => http<{ items: any[]; total: number }>(`/api/incidents?${new URLSearchParams(params as any)}`),
  });

export const useIncident = (id: string) =>
  useQuery({ queryKey: ['incident', id], queryFn: () => http(`/api/incidents/${id}`) });

export const useCreateIncident = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: object) => http('/api/incidents', { method: 'POST', body: JSON.stringify(body) }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['incidents'] }),
  });
};

// src/hooks/useServerEvents.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';

export function useServerEvents() {
  const qc = useQueryClient();
  useEffect(() => {
    const es = new EventSource('/api/events', { withCredentials: true });
    es.onmessage = (e) => {
      const evt = JSON.parse(e.data);
      const key = evt.table === 'incident' ? 'incident' : evt.table;
      qc.invalidateQueries({ queryKey: [`${key}s`] });
      qc.invalidateQueries({ queryKey: [key, evt.sys_id] });
    };
    return () => es.close();
  }, [qc]);
}
```

### 7.4 Screens

| Page | Features |
|---|---|
| Dashboard | Counts by state/priority (`/api/now/stats`), my open items |
| Incident list | Search, filter by state, server-side pagination, live updates |
| Incident detail | Fields, activity stream (comments + work notes), attachments, resolve/close actions |
| New incident | Validated form (react-hook-form + zod), attachment upload |
| Change requests | List, detail, approval status, schedule window |
| Service catalog | Browse items, render variables dynamically, order, track RITM |

### 7.5 State mapping (Incident)

| Value | Label | UI action available |
|---|---|---|
| 1 | New | Assign, Comment |
| 2 | In Progress | Comment, On Hold, Resolve |
| 3 | On Hold | Resume, Comment |
| 6 | Resolved | Reopen, Close |
| 7 | Closed | Read-only |
| 8 | Canceled | Read-only |

Confirm values against your instance — customers often customize choice lists. Fetch them from `sys_choice` at startup and cache.

---

## 8. Common package.json

One `package.json` at the repo root holds every dependency and script for both the React client and the Express server. There are no `package.json` files inside `client/` or `server/`.

```json
{
  "name": "servicenow-e2e",
  "version": "1.0.0",
  "private": true,
  "type": "commonjs",
  "engines": { "node": ">=20" },
  "scripts": {
    "dev": "concurrently -k -n server,client -c blue,green \"npm:dev:server\" \"npm:dev:client\"",
    "dev:server": "tsx watch --env-file=.env server/src/index.ts",
    "dev:client": "vite --config client/vite.config.ts",
    "build": "npm run build:client && npm run build:server",
    "build:client": "tsc -p client/tsconfig.json --noEmit && vite build --config client/vite.config.ts",
    "build:server": "tsc -p server/tsconfig.json",
    "start": "node --env-file-if-exists=.env server/dist/index.js",
    "preview": "npm run build && npm start",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "lint": "eslint .",
    "typecheck": "tsc -p server/tsconfig.json --noEmit && tsc -p client/tsconfig.json --noEmit"
  },
  "dependencies": {
    "axios": "^1.7.0",
    "compression": "^1.7.4",
    "express": "^4.21.0",
    "express-rate-limit": "^7.4.0",
    "express-session": "^1.18.0",
    "helmet": "^8.0.0",
    "multer": "^1.4.5-lts.1",
    "pino": "^9.5.0",
    "pino-http": "^10.3.0",
    "zod": "^3.23.0",

    "@hookform/resolvers": "^3.9.0",
    "@tanstack/react-query": "^5.59.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-hook-form": "^7.53.0",
    "react-router-dom": "^6.27.0"
  },
  "devDependencies": {
    "@playwright/test": "^1.48.0",
    "@testing-library/react": "^16.0.0",
    "@types/compression": "^1.7.5",
    "@types/express": "^4.17.21",
    "@types/express-session": "^1.18.0",
    "@types/multer": "^1.4.12",
    "@types/node": "^20.16.0",
    "@types/react": "^18.3.11",
    "@types/react-dom": "^18.3.1",
    "@types/supertest": "^6.0.2",
    "@vitejs/plugin-react": "^4.3.0",
    "concurrently": "^9.0.0",
    "eslint": "^9.12.0",
    "jsdom": "^25.0.0",
    "msw": "^2.4.0",
    "nock": "^13.5.0",
    "supertest": "^7.0.0",
    "tsx": "^4.19.0",
    "typescript": "^5.6.0",
    "vite": "^5.4.0",
    "vitest": "^2.1.0"
  }
}
```

> Versions shown are a known-good baseline; run `npm outdated` and bump to current releases when you scaffold. In `dependencies`, the server packages come first and the client packages after the blank line. `npm install` will re-sort them alphabetically.

### TypeScript configs

The two halves still compile differently, so each keeps its own `tsconfig.json`.

`server/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["src"]
}
```

`client/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "types": ["vite/client"]
  },
  "include": ["src", "vite.config.ts"]
}
```

### Script reference

| Script | What it does |
|---|---|
| `npm run dev` | Runs Express (`:4000`, auto-reload via tsx) and Vite (`:5173`, HMR) together |
| `npm run build` | Type-checks + builds React to `client/dist`, compiles server to `server/dist` |
| `npm start` | Production: Express serves `/api`, `/webhooks` and the React build on one port |
| `npm test` | Vitest for both server and client tests |
| `npm run test:e2e` | Playwright against a running app |
| `npm run typecheck` | Type-checks both projects without emitting |

### Run locally

```bash
cp .env.example .env          # fill in ServiceNow values
npm install
npm run dev                   # React on :5173, Express on :4000
```

For webhooks in local dev, expose Express with a tunnel (ngrok, Cloudflare Tunnel) and point the Outbound REST Message at it.

### Production

```bash
npm run build
NODE_ENV=production npm start   # Express serves /api and the React build on one port
```

---

## 9. Security Checklist

- [ ] ServiceNow secrets only on the server; loaded from a vault in prod
- [ ] Integration user has least-privilege roles and table ACLs
- [ ] Session cookies `httpOnly`, `secure`, `sameSite=lax`; CSRF token on mutating routes if cross-site embedding is possible
- [ ] Input validation with zod on every route; encoded-query characters stripped from free text
- [ ] Field allow-list on `PATCH` (don't pass arbitrary fields through to ServiceNow)
- [ ] Webhook HMAC verification with timing-safe compare; IP allow-list for the instance if possible
- [ ] Attachment upload size/type limits (multer) and virus scan if required
- [ ] Rate limiting on `/api`; respect ServiceNow `429` / `Retry-After`
- [ ] Helmet CSP configured for the React bundle
- [ ] No PII in logs; correlation ID propagated to ServiceNow via `X-Request-Id`

---

## 10. Testing Strategy

| Layer | Tooling | What to cover |
|---|---|---|
| Server unit | Vitest/Jest + `nock` | Token caching, retry/backoff, error mapping, query building |
| Server integration | Supertest | Route validation, auth guard, webhook signature |
| Client unit | Vitest + React Testing Library + MSW | Forms, hooks, state-based action buttons |
| Contract | Recorded ServiceNow fixtures | Detect breaking field/choice changes |
| E2E | Playwright against a ServiceNow **sub-production** instance (or PDI) | Create → comment → resolve → close incident; catalog order → RITM |

---

## 11. Delivery Plan

| Phase | Duration | Deliverables |
|---|---|---|
| **0. Discovery** | 1 week | Confirm tables, fields, choice lists, grant type, roles; ServiceNow dev instance access |
| **1. Foundation** | 1 week | Monorepo, Express bootstrap, config validation, OAuth token service, health check, CI pipeline |
| **2. Incidents MVP** | 2 weeks | List/detail/create/update/comment/resolve; React screens; unit + integration tests |
| **3. Real-time** | 1 week | Business Rules + Outbound REST, webhook receiver, SSE, cache invalidation |
| **4. Attachments & Changes** | 1–2 weeks | Attachment upload/download, Change Request flows, approvals view |
| **5. Service Catalog** | 2 weeks | Catalog browse, dynamic variable rendering, order, RITM tracking |
| **6. Hardening** | 1 week | Security checklist, load test, observability dashboards, Playwright E2E |
| **7. Release** | 1 week | UAT, runbook, production cutover, hypercare |

**Total:** ~10–11 weeks for one full-stack dev + part-time ServiceNow admin.

---

## 12. Operations

- **Health:** `/api/health` (app) and `/api/health/servicenow` (performs a `sysparm_limit=1` call).
- **Logging:** pino JSON logs → your log platform; include `requestId`, route, ServiceNow status, latency.
- **Metrics:** request count/latency per route, ServiceNow error rate, token refresh count, SSE client count.
- **Alerts:** ServiceNow 5xx > 5% for 5 min, token refresh failures, webhook signature failures spike.
- **Scaling:** stateless Express behind a load balancer; sessions in Redis; SSE fan-out via Redis Pub/Sub.

---

## 13. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401` from `oauth_token.do` | Wrong client ID/secret or grant type not enabled | Check Application Registry and grant-type property |
| `403` on table call | Integration user lacks role or ACL | Add role / adjust ACL |
| Empty `result` with 200 | Query matches nothing or ACL filters rows silently | Test the encoded query in the ServiceNow list view as the integration user |
| Display values missing | `sysparm_display_value` not set | Use `all` to get both `value` and `display_value` |
| Webhooks not arriving | Business Rule not async, wrong endpoint, MID server needed for private networks | Check Outbound HTTP log (`sys_outbound_http_log`) |
| `429 Too Many Requests` | Instance rate-limit rules | Backoff honours `Retry-After`; reduce polling; batch reads |
