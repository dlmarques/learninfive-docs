# Application Flows

This document explains the important runtime flows in the current application.

## 1. Application Boot

1. `src/main.tsx` reads `VITE_CLERK_PUBLISHABLE_KEY`.
2. If the key is missing, the client throws `Missing Publishable Key`.
3. Axios interceptors are registered with `setupInterceptors()`.
4. The React tree is rendered with:
   - Chakra provider
   - theme provider
   - React Query provider
   - Clerk provider
   - TanStack Router provider
   - toaster
5. `WrappedRouter` passes an async `token()` function into router context using Clerk `getToken()`.

## 2. Route Guard Flow

The router is configured in `learninfive-client/src/main.tsx`.

### `/`

1. Router calls `beforeLoad`.
2. It asks Clerk for a token through route context.
3. If a token exists, the client calls `/users/check-user-profile`.
4. If the profile does not exist, the user is redirected to `/complete-profile`.
5. Otherwise the `TopicPage` renders.
6. Anonymous users skip profile checking and can view the public topic.

### `/complete-profile`

1. Router asks Clerk for a token.
2. If no token exists, the user is redirected to `/`.
3. If a token exists, the client calls `/users/check-user-profile`.
4. If a profile already exists, the user is redirected to `/`.
5. Otherwise the completion form renders.

### `/edit-profile-preferences`

1. Router asks Clerk for a token.
2. If a token exists, the client checks whether the profile exists.
3. If the profile does not exist, the user is redirected to `/complete-profile`.
4. Otherwise the edit profile page renders.

## 3. Public Topic Flow

Public topic means no Authorization token is sent with `GET /topics/get-topic`.

1. Client renders `Topic` feature.
2. React Query calls `getTopic(token)`.
3. No token is passed, so the server runs `getPublicTopic`.
4. Server checks whether public generation is already in progress.
5. Server reads all public topics from `topics.topic`.
6. Server finds the first public topic whose `date` is today.
7. If found, it returns the existing topic.
8. If not found, it calls OpenAI through `getPublicModelResponse`.
9. The model response is parsed into a `Topic`.
10. Server adds `id`, `date`, and `public: true`.
11. Server inserts the topic into MongoDB.
12. Server returns the topic to the client.

## 4. Personalized Topic Flow

Personalized topic means an Authorization token is sent with `GET /topics/get-topic`.

1. Client retrieves a Clerk token with `getToken()`.
2. Client sends the token as `Authorization: Bearer <token>`.
3. Server extracts the token.
4. Server decodes the token and reads `sub` as `userId`.
5. Server checks whether generation is already in progress for that user.
6. Server reads topics from `topics.topic` by `userId`.
7. Server reads the user document from `users.user`.
8. Server finds today's topic for that user.
9. If today's topic exists:
   - server checks whether the quiz id exists in `user.answeredQuizzes`
   - if answered, server returns the topic with `quiz.userAnswer`
   - if not answered, server returns the topic as stored
10. If today's topic does not exist:
   - server extracts previous topic concepts from `user.pastTopics`
   - server asks OpenAI for a personalized topic
   - server adds `quiz.id`, topic `id`, `date`, `userId`, and `public: false`
   - server inserts the topic into `topics.topic`
   - server appends `{ id, concept }` to `user.pastTopics`
   - server returns the new topic

## 5. Scheduled Public Topic Flow

Primary code:

- `learninfive-server/controllers/topics.controller.ts`

Behavior:

1. `node-schedule` registers `scheduleJob("0 0 * * *", requestAndSaveNewPublicTopic)`.
2. At midnight according to the server runtime timezone, the server requests a new public topic.
3. It reads previous public concepts.
4. It asks OpenAI for a logically progressive new public concept.
5. It stores the topic as public.

Important note:

- The scheduled job runs in each server process. If multiple server instances run the same code, each process can schedule the job.

## 6. Complete Profile Flow

1. Signed-in user reaches `/complete-profile`.
2. User enters:
   - CS level
   - goals
   - preferences and skills
   - topics to avoid
3. `useCompleteProfile` reads Clerk `userId` and token.
4. Client sends `POST /users/complete-profile`.
5. Server verifies that the submitted `userId` matches the authenticated token subject.
6. Server checks whether the user already exists.
7. If the user does not exist, server inserts the profile into `users.user`.
8. Client shows a success toast and navigates to `/`.

## 7. Edit Profile Flow

1. User opens the Clerk user menu action `Edit profile preferences`.
2. Browser navigates to `/edit-profile-preferences`.
3. Client fetches `GET /users/get-user`.
4. The edit form is prefilled.
5. User submits updated profile fields.
6. Client sends `PATCH /users/edit-profile`.
7. Server verifies that the submitted `userId` matches the authenticated token subject.
8. Server updates the user document.
9. Client shows a success toast and navigates to `/`.

## 8. Quiz Answer Flow

### Anonymous

1. User selects an answer.
2. Client sends `POST /topics/answer-quiz` without Authorization.
3. Server finds the topic by `topicId`.
4. Server compares `answer` with `quiz.rightAnswer`.
5. Server returns `correct: true` or `correct: false`.
6. Client stores the attempt in `localStorage`.
7. Client shows the result.

### Authenticated

1. User selects an answer.
2. Client retrieves a Clerk token.
3. Client sends `POST /topics/answer-quiz` with Authorization.
4. Server decodes the token and reads `sub` as `userId`.
5. Server finds the topic by `topicId` and `userId`.
6. Server compares `answer` with `quiz.rightAnswer`.
7. Server appends `{ id: quiz.id, correctness }` to `user.answeredQuizzes`.
8. Server returns the result.
9. Client shows the result.

## 9. Global Error Flow

1. An Axios response error occurs.
2. The response interceptor maps the status to an error id.
3. The error id is stored in `localStorage`.
4. Unless the server content is `"Topic in progress"`, the browser navigates to `/error`.
5. `/error` renders the user-facing message from `ErrorMap`.

## 10. Theme Flow

1. `ThemeChanger` reads `localStorage.theme`.
2. If a value exists, it is used.
3. Otherwise, `window.matchMedia("(prefers-color-scheme: dark)")` decides the initial theme.
4. The theme is stored in Zustand and persisted in `localStorage`.
5. The HTML root gets `data-theme="dark"` or `data-theme="light"`.
