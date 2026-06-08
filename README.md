# LearnInFive Documentation

Last reviewed: 2026-06-07

LearnInFive is an AI-assisted learning app that generates one short programming or computer science topic per day. Anonymous visitors receive the public daily topic. Signed-in users receive topics personalized from their profile, past topics, goals, preferences, and avoided topics.

This folder documents the current implementation across both projects:

- `learninfive-client`: Vite, React, TypeScript, TanStack Router, React Query, Chakra UI, Clerk.
- `learninfive-server`: Express, TypeScript, MongoDB, Clerk token verification, OpenAI.

## Documentation Map

- [Architecture](./architecture.md): system boundaries, runtime components, and responsibilities.
- [Features](./features.md): product behavior by user-facing feature and screen.
- [Application Flows](./flows.md): how routing, topic generation, profile completion, and quiz answering work.
- [Data Model](./data-model.md): persisted entities, API DTOs, and data ownership.
- [API Reference](./api.md): server endpoints, auth behavior, request/response shapes.
- [Diagrams](./diagrams.md): Mermaid diagrams for architecture, routing, sequence flows, and data relationships.
- [Development and Operations](./development-and-operations.md): local setup, scripts, environment variables, testing, deployment, and known gaps.

## Project Shape

```text
learninfive/
  docs/
  learninfive-client/
    src/
      features/
      pages/
      shared/
      service/
      store/
      types/
      utils/
  learninfive-server/
    controllers/
    middlewares/
    routes/
    types/
    utils/
```

## System Summary

The app has one primary loop:

1. The browser loads `/`.
2. If the user is signed in, the client checks whether their learning profile exists.
3. The client asks the server for today's topic.
4. The server returns an existing daily topic or generates one with OpenAI.
5. The client renders the concept, definition, analogy, language examples, and quiz.
6. Quiz answers are checked by the server. Authenticated answers are persisted to the user's profile.

The app deliberately supports both anonymous and authenticated use:

- Anonymous users can read the public topic and answer its quiz locally.
- Authenticated users get personalized topics and persisted quiz history.

