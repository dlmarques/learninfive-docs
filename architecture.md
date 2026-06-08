# Architecture

LearnInFive is split into a browser client and an API server. The client owns routing, authentication UX, rendering, and frontend state. The server owns content generation, persistence, token verification for protected user operations, and daily topic selection.

## High-Level Components

| Component | Path | Responsibility |
| --- | --- | --- |
| Client app | `learninfive-client/src` | React app, routes, pages, API calls, theme state, user experience. |
| API server | `learninfive-server` | Express routes, controllers, MongoDB access, OpenAI calls, security middleware. |
| Authentication provider | Clerk | Sign-in/sign-up UX on the client and JWT verification on protected server routes. |
| Database | MongoDB Atlas | Stores users and generated topics. |
| AI provider | OpenAI | Generates public and personalized learning topics. |

## Client Architecture

The client is a Vite React TypeScript app.

Important entry points:

- `src/main.tsx`: bootstraps providers, configures TanStack Router, registers Axios interceptors.
- `src/App.tsx`: root route shell with `Outlet`.
- `src/Provider.tsx`: wraps Chakra UI and `next-themes`.
- `src/pages/*`: route-level components, mostly lazy-loading feature modules.
- `src/features/*`: feature implementations.
- `src/shared/*`: shared layout, header, footer, quiz, code examples, form controls.
- `src/utils/interceptors.ts`: central Axios instance and global error redirect behavior.

Runtime providers:

- `ChakraProvider`: UI primitives.
- `ThemeProvider`: theme support.
- `QueryClientProvider`: server-state caching and retries.
- `ClerkProvider`: authentication context.
- `RouterProvider`: route matching, guards, and route-level redirects.
- `Toaster`: user notifications.

## Server Architecture

The server is an Express TypeScript API.

Important entry points:

- `index.ts`: configures CORS, JSON parsing, rate limiting, production Helmet headers, routes, database connection, and server listen.
- `routes/topics.ts`: topic retrieval and quiz-answer endpoints.
- `routes/user.ts`: protected user profile endpoints.
- `controllers/topics.controller.ts`: topic lookup, OpenAI generation, quiz answer persistence.
- `controllers/user.controller.ts`: profile existence, profile creation, profile editing, user retrieval.
- `controllers/model.controller.ts`: OpenAI prompts and response cleanup.
- `utils/dbConnect.ts`: MongoDB client configuration and connection.

## Request Boundaries

The client communicates with the backend through one Axios instance configured in `src/utils/interceptors.ts`.

The base URL is controlled by:

```text
VITE_BACKEND_API_URL
```

Authentication is passed as a Bearer token when available:

```text
Authorization: Bearer <clerk-token>
```

The server applies Clerk verification to `/users/*` routes through `verifyTokenMiddleware`. The `/topics/get-topic` endpoint accepts both anonymous and authenticated requests. When a token is present, topic generation is personalized from the decoded Clerk subject.

## Persistence Boundaries

MongoDB is accessed directly through the native `mongodb` client. The implementation uses two logical databases:

- `users`, collection `user`
- `topics`, collection `topic`

There is no ORM schema layer. TypeScript interfaces in `learninfive-server/types` document the expected document shapes.

## Content Generation

Topic generation happens server-side in `controllers/model.controller.ts`.

There are two prompt families:

- Public prompt: asks for a unique programming/computer science concept, based on previous public concepts when available.
- User prompt: asks for a unique concept tailored to the user's level, preferences, goals, avoided topics, and past topics.

The expected model response is JSON with:

- `concept`
- `definition`
- `realWorldAnalogy`
- `examples`
- `quiz`

The server strips Markdown code fences from model output, parses JSON, enriches it with ids, ownership, `date`, and `public`, then writes it to MongoDB.

## Daily Topic Strategy

The current behavior is "one topic per day" by date comparison.

Public topics:

- The server searches existing public topics.
- If a topic exists for today, it returns that topic.
- Otherwise, it generates and stores a new public topic.
- A scheduled job also generates a public topic at midnight using `node-schedule`.

User topics:

- The server searches topics for the Clerk `sub` user id.
- If today's personalized topic exists, it returns it.
- If the user already answered that topic's quiz, the returned quiz includes `userAnswer`.
- Otherwise, it generates a new personalized topic and appends it to the user's `pastTopics`.

## Concurrency Guard

`topics.controller.ts` uses in-memory guards while generation is in progress:

- `inProgress`: tracks active generation by `"public"` or `userId`.
- `publicTopicInProgress`: stores an in-flight public topic once generated.
- `usersTopicsInProgress`: stores in-flight personalized topics by user id.

This prevents duplicate generation inside a single server process. Because the guard is in memory, it does not coordinate across multiple server instances.

## Security Controls

Current controls:

- CORS allowlist for local development and production domains.
- Express JSON parsing.
- Global rate limiting: 50 requests per 15 minutes.
- Clerk token verification for `/users/*`.
- Production-only Helmet headers and content security policy.
- MongoDB credentials and OpenAI/Clerk secrets loaded from environment variables.

Important nuance:

- `/topics/get-topic` does not verify the token with Clerk. It extracts and decodes the token when present to find the subject. Protected user profile routes do verify the token.

## Architecture Decisions Reflected in Code

- The public learning experience does not require sign-in.
- Personalization starts only after the user completes a profile.
- Topic generation is backend-only so OpenAI credentials stay off the client.
- React Query owns topic and profile fetch state.
- TanStack Router owns route-level profile completion guards.
- The backend stores denormalized learning history on the user document for prompt generation.

## Known Implementation Notes

These notes document the current code, not desired behavior:

- The `completeProfileValidation` and `editProfileValidation` middleware arrays exist but are not applied in `routes/user.ts`.
- The topic generation guard is process-local and will not prevent duplicate generation across horizontally scaled server instances.
- The OpenAI JSON response is parsed directly. Malformed or schema-incompatible JSON will fail at runtime unless handled upstream.
