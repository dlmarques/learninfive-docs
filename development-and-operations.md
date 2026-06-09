# Development and Operations

This document covers how to run, test, configure, and operate the current project.

## Prerequisites

- Node.js 20 is used by the client CI workflow.
- npm is supported by both projects through `package-lock.json`.
- The client also has a `pnpm-lock.yaml`, but the configured CI workflow uses npm.
- MongoDB Atlas credentials are required for the server.
- Clerk application credentials are required for authentication.
- OpenAI API credentials are required for topic generation.

## Local Setup

Install client dependencies:

```bash
cd learninfive-client
npm install
```

Install server dependencies:

```bash
cd learninfive-server
npm install
```

Run the server:

```bash
cd learninfive-server
npm run dev
```

Run the client:

```bash
cd learninfive-client
npm run start
```

Default local URLs:

- Client: `http://localhost:3000`
- Server: `http://localhost:8000`

## Environment Variables

### Client

Used in `learninfive-client`:

| Variable | Purpose |
| --- | --- |
| `VITE_BACKEND_API_URL` | Base URL used by Axios for API requests. |
| `VITE_CLERK_PUBLISHABLE_KEY` | Clerk publishable key used by `ClerkProvider`. |

### Server

Used in `learninfive-server`:

| Variable | Purpose |
| --- | --- |
| `PORT` | Optional server port. Defaults to `8000`. |
| `NODE_ENV` | Enables production Helmet headers when set to `production`. |
| `MONGO_DB_URI` | Optional full MongoDB connection string override. Useful for tests or non-Atlas deployments. |
| `MONGO_DB_USER` | MongoDB Atlas username. |
| `MONGO_DB_PASSWORD` | MongoDB Atlas password. |
| `TOPIC_GENERATION_LOCK_TTL_MS` | Optional topic generation lease duration. Defaults to 120000 milliseconds. |
| `OPEN_AI_API_KEY` | OpenAI API key. |
| `CLERK_JWT_KEY` | Clerk JWT verification key. |

## Client Scripts

From `learninfive-client/package.json`:

| Script | Command | Purpose |
| --- | --- | --- |
| `start` | `vite --port 3000` | Start local Vite dev server. |
| `build` | `vite build && tsc` | Build static assets and run TypeScript check. |
| `serve` | `vite preview` | Preview production build locally. |
| `test` | `vitest run` | Run client tests. |

## Server Scripts

From `learninfive-server/package.json`:

| Script | Command | Purpose |
| --- | --- | --- |
| `dev` | `tsx index.ts` | Run API server in development. |
| `test` | `vitest run` | Run backend tests. |
| `build` | `tsc` | Compile TypeScript into `dist`. |
| `start` | `node dist/index.js` | Run compiled server. |

## Testing

Client tests exist under:

```text
learninfive-client/src/**/__tests__
```

Current coverage areas:

- About page render behavior.
- Error-code mapping.
- User profile completion helper.
- Theme utilities.
- Axios interceptor behavior.

Run client tests:

```bash
cd learninfive-client
npm run test
```

Backend tests exist under:

```text
learninfive-server/__tests__
```

Current backend coverage includes MongoDB-backed topic generation locking, duplicate insert fallback, and UTC day-key behavior.

Run backend tests:

```bash
cd learninfive-server
npm run test
```

## CI

Client CI exists at:

```text
learninfive-client/.github/workflows/ci.yml
```

It runs on pull requests to `main`:

1. Checkout.
2. Setup Node.js 20.
3. `npm ci`.
4. `npm run test`.
5. `npm run build`.

No server CI workflow was found in the server project.

## Deployment Notes

Client:

- `vercel.json` rewrites all routes to `index.html`, which supports client-side routing.
- Production client domain is included in server CORS allowlist.

Server:

- Production mode enables Helmet security headers.
- Server CORS allowlist must include the deployed client origin.
- Server must be able to reach MongoDB Atlas and OpenAI.

## Operational Behavior

### Topic generation latency

OpenAI generation can take long enough that the client treats `"Topic in progress"` as retryable. The topic query retries 24 times with a 5-second delay.

### Rate limiting

The limiter is applied globally in `index.ts`. It currently appears twice:

```ts
app.use(limiter);
...
app.use(limiter);
```

This should be reviewed before changing rate-limit behavior.

### Daily schedule

The public topic schedule is:

```text
0 0 * * *
```

That means midnight in the server process timezone. Topic uniqueness is still keyed by UTC `dayKey`, so all server instances agree on the persisted daily topic.

### Topic generation locking

Topic generation uses MongoDB-backed lease documents in `topics.topicGenerationLocks`. If another instance is already generating the same public or user daily topic, the API keeps returning the retry-compatible `"Topic in progress"` response.

Topic uniqueness is enforced by MongoDB indexes:

- public topics: one topic per `dayKey`
- personalized topics: one topic per `userId + dayKey`

## Troubleshooting

### Client throws `Missing Publishable Key`

Check:

```text
VITE_CLERK_PUBLISHABLE_KEY
```

### Client cannot reach API

Check:

```text
VITE_BACKEND_API_URL
```

Also verify the API origin is included in CORS when running a nonstandard local port or domain.

### Protected user routes return 401

Check:

- Clerk token is being sent as `Authorization: Bearer <token>`.
- `CLERK_JWT_KEY` is configured on the server.
- The client origin is included in the middleware `authorizedParties`.

### Topic requests fail or hang

Check:

- MongoDB credentials.
- OpenAI API key.
- Server logs around JSON parsing.
- Whether rate limiting has been hit.
- Whether a topic generation request is returning `"Topic in progress"` repeatedly.

### Production page refresh returns 404

Check the static host rewrite configuration. The client includes a Vercel rewrite from every path to `/index.html`.

## Maintenance Priorities

These are the highest-leverage follow-ups visible from the current code:

1. Add backend tests for topic generation, profile creation/editing, token verification, and quiz answer persistence.
2. Add runtime validation for OpenAI JSON before writing to MongoDB.
3. Consider indexes or unique constraints for `userId` and topic `id`.
