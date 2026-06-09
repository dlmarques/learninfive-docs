# API Reference

The API server is an Express app in `learninfive-server`.

Base URL:

- Local default server port: `http://localhost:8000`
- Client configuration: `VITE_BACKEND_API_URL`

All JSON endpoints expect:

```text
Content-Type: application/json
```

Protected endpoints expect:

```text
Authorization: Bearer <clerk-token>
```

## Cross-Cutting Behavior

### CORS

Allowed origins in `index.ts`:

- `http://localhost:3000`
- `https://learninfive.com`
- `https://www.learninfive.com`

### Rate Limit

Configured in `utils/limiter.ts`:

- 50 requests
- 15-minute window
- response message: `Requests limit are reached.`

### Auth Verification

`/users/*` routes use `verifyTokenMiddleware`, which verifies Clerk tokens with:

- `CLERK_JWT_KEY`
- authorized parties:
  - `http://localhost:3000`
  - `https://www.learninfive.com`
  - `https://learninfive.com`

`/topics/*` routes do not use `verifyTokenMiddleware`. They extract a token if present.

## `GET /topics/get-topic`

Returns today's topic.

Auth:

- Optional.
- Without token: returns public daily topic.
- With token: returns personalized topic for the decoded token subject.

Request:

```http
GET /topics/get-topic
Authorization: Bearer <token> // optional
```

Success response:

```json
{
  "success": true,
  "content": {
    "id": "topic-uuid",
    "concept": "Queues",
    "definition": "...",
    "realWorldAnalogy": "...",
    "examples": [
      {
        "language": "JavaScript",
        "code": "..."
      }
    ],
    "quiz": {
      "id": "quiz-uuid",
      "question": "...",
      "answers": [
        {
          "id": "a",
          "content": "..."
        }
      ],
      "rightAnswer": "a",
      "userAnswer": true
    },
    "date": "2026-06-07T00:00:00.000Z",
    "public": false,
    "userId": "clerk-user-id"
  }
}
```

Possible in-progress response:

```json
{
  "success": false,
  "content": "Topic in progress"
}
```

Notes:

- The client intentionally does not redirect to `/error` when this response content is received.
- React Query retries the topic request.
- Topics receive server-generated topic and quiz IDs.

## `POST /topics/answer-quiz`

Checks a submitted quiz answer.

Auth:

- Optional.
- Without token: checks answer but does not persist server-side.
- With token: checks answer and writes the result into the user's `answeredQuizzes`.

Request:

```http
POST /topics/answer-quiz
Authorization: Bearer <token> // optional
Content-Type: application/json
```

Body:

```json
{
  "topicId": "topic-uuid",
  "answer": "answer-id"
}
```

Validation:

- `topicId` must be a UUID.
- `answer` must be a string.

Success response:

```json
{
  "success": true,
  "content": "Quiz answered correctly",
  "correct": true
}
```

Failure response used by current code:

```json
{
  "success": false,
  "content": "Something went wrong"
}
```

## `GET /users/check-user-profile`

Checks whether the authenticated user has completed a profile.

Auth:

- Required.
- Protected by Clerk verification middleware.

Request:

```http
GET /users/check-user-profile
Authorization: Bearer <token>
```

Profile exists:

```json
{
  "success": true,
  "content": "User already registered.",
  "exists": true
}
```

Profile missing:

```json
{
  "success": true,
  "content": "User not registered.",
  "exists": false
}
```

No token:

```json
{
  "error": "Token not found. User must sign in."
}
```

Invalid token:

```json
{
  "error": "Token not verified."
}
```

## `GET /users/get-user`

Returns the authenticated user's profile document.

Auth:

- Required.
- Protected by Clerk verification middleware.

Request:

```http
GET /users/get-user
Authorization: Bearer <token>
```

Profile found:

```json
{
  "success": true,
  "content": {
    "userId": "clerk-user-id",
    "csLevel": "...",
    "goals": "...",
    "preferences": "...",
    "topicsToAvoid": "...",
    "pastTopics": [],
    "answeredQuizzes": []
  },
  "exists": true
}
```

Profile missing:

```json
{
  "success": true,
  "content": "User not found",
  "exists": false
}
```

## `POST /users/complete-profile`

Creates a user profile.

Auth:

- Required.
- Protected by Clerk verification middleware.

Request:

```http
POST /users/complete-profile
Authorization: Bearer <token>
Content-Type: application/json
```

Body:

```json
{
  "userId": "clerk-user-id",
  "csLevel": "Beginner with JavaScript basics",
  "goals": "Prepare for backend interviews",
  "preferences": "Likes practical examples",
  "topicsToAvoid": "Advanced category theory"
}
```

The `userId` value must match the authenticated Clerk token subject.

Success response:

```json
{
  "success": true,
  "content": "User created."
}
```

Already exists:

```json
{
  "success": false,
  "content": "User already exists."
}
```

User mismatch:

```json
{
  "success": false,
  "content": "Profile userId does not match authenticated user."
}
```

Generic failure:

```json
{
  "success": false,
  "content": "Something went wrong"
}
```

## `PATCH /users/edit-profile`

Updates user profile preferences.

Auth:

- Required.
- Protected by Clerk verification middleware.

Request:

```http
PATCH /users/edit-profile
Authorization: Bearer <token>
Content-Type: application/json
```

Body:

```json
{
  "userId": "clerk-user-id",
  "csLevel": "Intermediate",
  "goals": "Improve distributed systems knowledge",
  "preferences": "TypeScript and diagrams",
  "topicsToAvoid": "CSS basics"
}
```

The `userId` value must match the authenticated Clerk token subject.

Success response:

```json
{
  "success": true,
  "content": "User edited successfully",
  "edited": true
}
```

No modification or failure response:

```json
{
  "success": true,
  "content": "Somethin went wrong",
  "edited": false
}
```
