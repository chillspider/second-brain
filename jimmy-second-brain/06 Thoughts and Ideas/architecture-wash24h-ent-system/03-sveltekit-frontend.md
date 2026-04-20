# 03 — SvelteKit Frontend

> SvelteKit as a Backend-for-Frontend (BFF). Browser never sees tokens. OIDC Authorization Code flow + PKCE between SvelteKit server and Keycloak.

---

## 1. Why BFF, not SPA-holds-token

The naive approach stores access/refresh tokens in browser `localStorage` or JS memory. This exposes them to any XSS vuln in your app or its dependencies.

The BFF approach:

- SvelteKit server (running in Node) handles the OIDC flow with Keycloak.
- Tokens live server-side, in an encrypted session cookie (httpOnly, Secure, SameSite=Strict).
- Browser only ever sees a session cookie that's meaningless if stolen outside the domain.
- API calls go browser → SvelteKit server → .NET API, with SvelteKit attaching `Authorization: Bearer <access_token>`.

This is what Keycloak docs now recommend for confidential JS clients.

---

## 2. Initialize project

```bash
cd frontend
npm create svelte@latest .         # SvelteKit + TypeScript + ESLint + Prettier
npm install
npm i -D @sveltejs/adapter-node    # for standalone Node deployment
npm i openid-client jose cookie
```

**`svelte.config.js`:**

```js
import adapter from '@sveltejs/adapter-node';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

export default {
  preprocess: vitePreprocess(),
  kit: {
    adapter: adapter(),
    csrf: { checkOrigin: true },
  },
};
```

---

## 3. Environment variables

**`.env` (dev, gitignored):**

```
KEYCLOAK_ISSUER=https://auth.wash24h.com/realms/jetx
KEYCLOAK_CLIENT_ID=jetx-web
KEYCLOAK_CLIENT_SECRET=<from keycloak>
KEYCLOAK_REDIRECT_URI=http://localhost:5173/auth/callback
KEYCLOAK_POST_LOGOUT_REDIRECT_URI=http://localhost:5173/

API_BASE_URL=http://localhost:5000

SESSION_SECRET=<32+ random bytes, base64>
PUBLIC_APP_NAME=JetX
```

Anything prefixed `PUBLIC_` is exposed to the browser. Everything else is server-only.

---

## 4. OIDC helper

**`src/lib/server/auth/oidc.ts`:**

```ts
import { Issuer, generators, type Client } from 'openid-client';
import {
  KEYCLOAK_ISSUER,
  KEYCLOAK_CLIENT_ID,
  KEYCLOAK_CLIENT_SECRET,
  KEYCLOAK_REDIRECT_URI,
  KEYCLOAK_POST_LOGOUT_REDIRECT_URI,
} from '$env/static/private';

let clientPromise: Promise<Client> | null = null;

export function getOidcClient(): Promise<Client> {
  if (!clientPromise) {
    clientPromise = Issuer.discover(KEYCLOAK_ISSUER).then(
      (issuer) =>
        new issuer.Client({
          client_id: KEYCLOAK_CLIENT_ID,
          client_secret: KEYCLOAK_CLIENT_SECRET,
          redirect_uris: [KEYCLOAK_REDIRECT_URI],
          post_logout_redirect_uris: [KEYCLOAK_POST_LOGOUT_REDIRECT_URI],
          response_types: ['code'],
        }),
    );
  }
  return clientPromise;
}

export function makePkce() {
  const code_verifier = generators.codeVerifier();
  const code_challenge = generators.codeChallenge(code_verifier);
  const state = generators.state();
  const nonce = generators.nonce();
  return { code_verifier, code_challenge, state, nonce };
}
```

---

## 5. Session storage (encrypted cookie)

For small deployments, an encrypted cookie is simpler than Redis. At scale, move to Redis.

**`src/lib/server/auth/session.ts`:**

```ts
import { EncryptJWT, jwtDecrypt } from 'jose';
import { SESSION_SECRET } from '$env/static/private';
import { createHash } from 'node:crypto';

const key = createHash('sha256').update(SESSION_SECRET).digest(); // 32-byte key

export type Session = {
  access_token: string;
  refresh_token: string;
  id_token: string;
  expires_at: number; // epoch seconds
  user: {
    sub: string;
    username: string;
    email: string;
    name: string;
    groups: string[];
    roles: string[];
  };
};

export async function encryptSession(s: Session): Promise<string> {
  return await new EncryptJWT(s as any)
    .setProtectedHeader({ alg: 'dir', enc: 'A256GCM' })
    .setIssuedAt()
    .setExpirationTime('12h')
    .encrypt(key);
}

export async function decryptSession(token: string): Promise<Session | null> {
  try {
    const { payload } = await jwtDecrypt(token, key);
    return payload as unknown as Session;
  } catch {
    return null;
  }
}
```

---

## 6. Routes: login, callback, logout

**`src/routes/auth/login/+server.ts`:**

```ts
import type { RequestHandler } from './$types';
import { redirect } from '@sveltejs/kit';
import { getOidcClient, makePkce } from '$lib/server/auth/oidc';

export const GET: RequestHandler = async ({ cookies, url }) => {
  const client = await getOidcClient();
  const { code_verifier, code_challenge, state, nonce } = makePkce();

  cookies.set('oidc_flow', JSON.stringify({ code_verifier, state, nonce, returnTo: url.searchParams.get('returnTo') ?? '/' }), {
    path: '/',
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 300,
  });

  const authUrl = client.authorizationUrl({
    scope: 'openid profile email',
    code_challenge,
    code_challenge_method: 'S256',
    state,
    nonce,
  });

  throw redirect(302, authUrl);
};
```

**`src/routes/auth/callback/+server.ts`:**

```ts
import type { RequestHandler } from './$types';
import { redirect, error } from '@sveltejs/kit';
import { getOidcClient } from '$lib/server/auth/oidc';
import { encryptSession, type Session } from '$lib/server/auth/session';
import { KEYCLOAK_REDIRECT_URI } from '$env/static/private';

export const GET: RequestHandler = async ({ url, cookies }) => {
  const raw = cookies.get('oidc_flow');
  if (!raw) throw error(400, 'Missing flow cookie');
  const { code_verifier, state, nonce, returnTo } = JSON.parse(raw);
  cookies.delete('oidc_flow', { path: '/' });

  const client = await getOidcClient();
  const params = client.callbackParams(url.toString());
  const tokenSet = await client.callback(KEYCLOAK_REDIRECT_URI, params, { code_verifier, state, nonce });

  const claims = tokenSet.claims();
  const session: Session = {
    access_token: tokenSet.access_token!,
    refresh_token: tokenSet.refresh_token!,
    id_token: tokenSet.id_token!,
    expires_at: tokenSet.expires_at!,
    user: {
      sub: claims.sub,
      username: (claims.preferred_username as string) ?? '',
      email: (claims.email as string) ?? '',
      name: (claims.name as string) ?? '',
      groups: ((claims as any).groups as string[]) ?? [],
      roles: ((claims as any).realm_access?.roles as string[]) ?? [],
    },
  };

  cookies.set('session', await encryptSession(session), {
    path: '/',
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 12,
  });

  throw redirect(302, returnTo || '/');
};
```

**`src/routes/auth/logout/+server.ts`:**

```ts
import type { RequestHandler } from './$types';
import { redirect } from '@sveltejs/kit';
import { getOidcClient } from '$lib/server/auth/oidc';
import { decryptSession } from '$lib/server/auth/session';
import { KEYCLOAK_POST_LOGOUT_REDIRECT_URI } from '$env/static/private';

export const GET: RequestHandler = async ({ cookies }) => {
  const raw = cookies.get('session');
  cookies.delete('session', { path: '/' });

  const client = await getOidcClient();
  let url = KEYCLOAK_POST_LOGOUT_REDIRECT_URI;

  if (raw) {
    const s = await decryptSession(raw);
    if (s) {
      url = client.endSessionUrl({
        id_token_hint: s.id_token,
        post_logout_redirect_uri: KEYCLOAK_POST_LOGOUT_REDIRECT_URI,
      });
    }
  }

  throw redirect(302, url);
};
```

---

## 7. `hooks.server.ts` — session guard + token refresh

```ts
import type { Handle } from '@sveltejs/kit';
import { decryptSession, encryptSession } from '$lib/server/auth/session';
import { getOidcClient } from '$lib/server/auth/oidc';

const PUBLIC_ROUTES = ['/', '/auth/login', '/auth/callback', '/auth/logout'];

export const handle: Handle = async ({ event, resolve }) => {
  const raw = event.cookies.get('session');
  if (raw) {
    let session = await decryptSession(raw);

    // Refresh if within 60 seconds of expiry
    if (session && session.expires_at - 60 < Math.floor(Date.now() / 1000)) {
      try {
        const client = await getOidcClient();
        const refreshed = await client.refresh(session.refresh_token);
        session = {
          ...session,
          access_token: refreshed.access_token!,
          refresh_token: refreshed.refresh_token ?? session.refresh_token,
          id_token: refreshed.id_token ?? session.id_token,
          expires_at: refreshed.expires_at!,
        };
        event.cookies.set('session', await encryptSession(session), {
          path: '/',
          httpOnly: true,
          secure: true,
          sameSite: 'strict',
          maxAge: 60 * 60 * 12,
        });
      } catch {
        session = null;
        event.cookies.delete('session', { path: '/' });
      }
    }

    if (session) {
      event.locals.session = session;
      event.locals.user = session.user;
    }
  }

  const needsAuth = !PUBLIC_ROUTES.some((p) => event.url.pathname === p || event.url.pathname.startsWith(p + '/'));
  if (needsAuth && !event.locals.session) {
    return new Response(null, { status: 302, headers: { location: `/auth/login?returnTo=${encodeURIComponent(event.url.pathname)}` } });
  }

  return resolve(event);
};
```

**`src/app.d.ts`:**

```ts
import type { Session } from '$lib/server/auth/session';

declare global {
  namespace App {
    interface Locals {
      session?: Session;
      user?: Session['user'];
    }
  }
}
export {};
```

---

## 8. Calling the .NET API from SvelteKit

Two patterns — pick one per endpoint:

### Pattern A: Browser → SvelteKit server endpoint → .NET (recommended default)

`src/routes/api/sites/+server.ts`:

```ts
import type { RequestHandler } from './$types';
import { json, error } from '@sveltejs/kit';
import { API_BASE_URL } from '$env/static/private';

export const GET: RequestHandler = async ({ locals, fetch }) => {
  if (!locals.session) throw error(401);
  const r = await fetch(`${API_BASE_URL}/api/sites`, {
    headers: { Authorization: `Bearer ${locals.session.access_token}` },
  });
  if (!r.ok) throw error(r.status);
  return json(await r.json());
};
```

Browser calls `/api/sites` on the SvelteKit app. Token never leaves the server.

### Pattern B: `load()` functions fetching directly

`src/routes/sites/+page.server.ts`:

```ts
import type { PageServerLoad } from './$types';
import { API_BASE_URL } from '$env/static/private';

export const load: PageServerLoad = async ({ locals, fetch }) => {
  const r = await fetch(`${API_BASE_URL}/api/sites`, {
    headers: { Authorization: `Bearer ${locals.session!.access_token}` },
  });
  return { sites: await r.json() };
};
```

SSR uses the session directly. Client gets HTML with data already in place.

---

## 9. Typed API client from OpenAPI

The .NET API emits OpenAPI at `/openapi/v1.json`. Generate a TypeScript client on frontend build:

```bash
npm i -D openapi-typescript
```

**`package.json` scripts:**

```json
"scripts": {
  "gen:api": "openapi-typescript http://localhost:5000/openapi/v1.json -o src/lib/api/schema.ts",
  "dev": "npm run gen:api && vite dev",
  "build": "npm run gen:api && vite build"
}
```

Then use `openapi-fetch`:

```ts
import createClient from 'openapi-fetch';
import type { paths } from './schema';

export const api = createClient<paths>({ baseUrl: API_BASE_URL });
```

---

## 10. UI toolkit suggestion

For an internal operational tool, skip heavy component libraries. `shadcn-svelte` (now stable) gives you copy-in components built on Tailwind + Bits UI primitives, which matches your existing shadcn experience from the FinOps platform.

```bash
npx shadcn-svelte@latest init
npx shadcn-svelte@latest add button card table input
```

For data-dense views (Grafana-like dashboards over Pi fleet, camera uptime, etc.): `@tanstack/svelte-table` for tables, `layerchart` or Chart.js for charts.

---

## 11. Protecting UI by role

Create a `CanSee` component that reads from `$page.data.user`:

```svelte
<!-- src/lib/components/CanSee.svelte -->
<script lang="ts">
  import { page } from '$app/stores';
  export let role: string | string[] | undefined = undefined;
  export let group: string | undefined = undefined;

  $: user = $page.data.user;
  $: roles = Array.isArray(role) ? role : role ? [role] : [];
  $: allowed = !!user && (
    (roles.length === 0 || roles.some(r => user.roles.includes(r))) &&
    (!group || user.groups.includes(group))
  );
</script>

{#if allowed}<slot />{/if}
```

Expose user to all pages in the root `+layout.server.ts`:

```ts
export const load = ({ locals }) => ({ user: locals.user });
```

**Authorization is still enforced by the .NET API.** Frontend checks are UX only — never trust them.

---

## 12. Things to avoid

- **Don't put `KEYCLOAK_CLIENT_SECRET` in any `PUBLIC_` env var.** If it ever touches the browser, rotate immediately.
- **Don't use `localStorage` for tokens.** The whole point of the BFF is to not do that.
- **Don't skip `state` and `nonce` validation.** `openid-client` does it for you if you pass them in.
- **Don't trust role claims on the frontend for security decisions.** Rendering vs. enforcing are different things.
- **Don't proxy file uploads through SvelteKit** for large files (NVR segments, etc.). Use a presigned URL from the API directly to MinIO.
