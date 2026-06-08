# Features

This document describes what the application currently does from a product and user-experience perspective.

## 1. Daily Learning Topic

Route:

```text
/
```

Primary code:

- `learninfive-client/src/features/topic/index.tsx`
- `learninfive-client/src/features/topic/services/getTopic.ts`
- `learninfive-server/controllers/topics.controller.ts`

Behavior:

- Shows a single programming or computer science concept for the day.
- The page title is rendered as `Learn {concept} in 5 minutes`.
- Content is split into:
  - Definition
  - Real-world analogy
  - Code examples
  - Quiz
- The client fetches the topic with React Query.
- The query retries up to 24 times with a 5-second delay. This supports the current server behavior where topic generation can temporarily return `"Topic in progress"`.

Anonymous user behavior:

- Receives the public daily topic.
- Can answer the quiz.
- Quiz result is stored locally in browser `localStorage`.

Authenticated user behavior:

- Receives a personalized daily topic after profile completion.
- Topic generation uses profile data and past topics.
- Quiz answer is persisted to the user document.
- Returning to an already answered quiz should show the prior result when `quiz.userAnswer` is returned by the server.

## 2. AI-Generated Topic Content

Primary code:

- `learninfive-server/controllers/model.controller.ts`
- `learninfive-server/controllers/topics.controller.ts`

Generated content includes:

- `concept`
- `definition`
- `realWorldAnalogy`
- code `examples`
- `quiz.question`
- `quiz.answers`
- `quiz.rightAnswer`

Required example languages requested in prompts:

- JavaScript
- Python
- Java
- C#
- C++
- TypeScript
- PHP

Public topics avoid previously generated public concepts when possible. Personalized topics avoid the user's prior topics and optional avoided topics.

## 3. Code Examples by Language

Primary code:

- `learninfive-client/src/features/topic/components/Examples.tsx`
- `learninfive-client/src/shared/components/code-example/index.tsx`

Behavior:

- Renders examples inside Chakra tabs.
- Displays language icons from `react-icons`.
- Normalizes model-provided language names into canonical language names.
- Uses `react-code-block` for syntax-highlighted code.
- Hides the whole code example component if none of the provided languages can be normalized.

Supported language aliases:

| Canonical | Accepted aliases |
| --- | --- |
| JavaScript | `javascript`, `node.js`, `nodejs`, `node` |
| TypeScript | `typescript` |
| Python | `python` |
| Java | `java` |
| C# | `c#`, `csharp`, `.net`, `dotnet` |
| C++ | `c++`, `cplusplus`, `cpp` |
| PHP | `php` |

## 4. Quiz

Primary code:

- `learninfive-client/src/features/topic/components/Quiz.tsx`
- `learninfive-client/src/shared/components/quiz/index.tsx`
- `learninfive-client/src/shared/components/quiz/components/Result.tsx`
- `learninfive-server/controllers/topics.controller.ts`

Behavior:

- Displays one multiple-choice quiz for the topic.
- User selects an answer with radio controls.
- User submits with a button.
- The server compares the selected answer id with `quiz.rightAnswer`.
- The UI displays either:
  - `Correct!`
  - `Wrong!` and the correct answer text

Authenticated persistence:

- The server writes `{ id: quizId, correctness }` into `user.answeredQuizzes`.
- On future loads, the server attaches `quiz.userAnswer` when today's quiz has already been answered.

Anonymous persistence:

- The client stores quiz attempts in `localStorage` under `pastPlayedQuizzes`.
- Stored guest attempts are matched by quiz id, with the topic id used as a fallback for older public topics that do not include a quiz id.

## 5. Authentication

Primary code:

- `learninfive-client/src/pages/SignIn.tsx`
- `learninfive-client/src/pages/SignUp.tsx`
- `learninfive-client/src/shared/components/header/index.tsx`
- `learninfive-server/middlewares/verifyToken.ts`

Behavior:

- Clerk owns sign-in and sign-up.
- `/sign-in` redirects to Clerk sign-in.
- `/sign-up` redirects to Clerk sign-up.
- After sign-in/sign-up, the configured redirect target is `/complete-profile`.
- The header shows:
  - a Clerk `UserButton` for signed-in users
  - a `Sign in` button for signed-out users
- The user button includes an `Edit profile preferences` action.

Server-side protected routes:

- `/users/check-user-profile`
- `/users/get-user`
- `/users/edit-profile`
- `/users/complete-profile`

## 6. Profile Completion

Route:

```text
/complete-profile
```

Primary code:

- `learninfive-client/src/features/complete-profile/index.tsx`
- `learninfive-client/src/features/complete-profile/components/ProfileCompleteForm.tsx`
- `learninfive-client/src/features/complete-profile/hooks/useCompleteProfile.ts`
- `learninfive-server/controllers/user.controller.ts`

Behavior:

- Signed-in users who do not have a profile are redirected here before using personalized learning.
- The form collects:
  - Computer Science level
  - Computer Science goals
  - Preferences and fluent skills
  - Topics to avoid
- On submit, the client sends the profile plus Clerk `userId` to the server.
- On success, a toast is shown and the user is navigated to `/`.

Route guard behavior:

- If no token is available, `/complete-profile` redirects to `/`.
- If a profile already exists, `/complete-profile` redirects to `/`.

## 7. Edit Profile Preferences

Route:

```text
/edit-profile-preferences
```

Primary code:

- `learninfive-client/src/features/edit-profile/index.tsx`
- `learninfive-client/src/features/edit-profile/components/EditProfileForm.tsx`
- `learninfive-client/src/features/edit-profile/hooks/useEditProfile.ts`
- `learninfive-server/controllers/user.controller.ts`

Behavior:

- Fetches the existing user profile.
- Pre-fills the profile form.
- Allows editing level, goals, preferences, and avoided topics.
- On success, shows a toast and navigates back to `/`.

Access behavior:

- Signed-in users with incomplete profiles are redirected to `/complete-profile`.
- Anonymous users are not redirected away by the current route guard because the guard only checks profile completion when a token exists.

## 8. Theme Switching

Primary code:

- `learninfive-client/src/shared/components/theme-changer/index.tsx`
- `learninfive-client/src/store/theme.store.ts`
- `learninfive-client/src/utils/getSystemTheme.ts`
- `learninfive-client/src/styles/styles.css`

Behavior:

- Detects system dark mode on first load.
- Persists selected theme in `localStorage` under `theme`.
- Stores the current theme in Zustand.
- Applies the theme through `html[data-theme="dark"]` or `html[data-theme="light"]`.
- Header icon toggles between light and dark.

## 9. Error Handling Page

Route:

```text
/error
```

Primary code:

- `learninfive-client/src/utils/interceptors.ts`
- `learninfive-client/src/utils/pathByError.ts`
- `learninfive-client/src/features/error/index.tsx`
- `learninfive-client/src/features/error/types/Error.ts`

Behavior:

- Axios response errors are mapped to a short error id.
- The id is written to `localStorage` under `error`.
- The browser is redirected to `/error`.
- The error page reads the id and displays the matching user-facing message.
- The interceptor does not redirect when the server response content is `"Topic in progress"`, allowing React Query retries to continue.

Mapped statuses:

| HTTP status | Error id | Message category |
| --- | --- | --- |
| 401 | `unauthorized` | Permission/auth problem |
| 404 | `not-found` | Missing resource |
| 429 | `too-many-requests` | Rate limit |
| 500 | `unexpected` | Generic failure |
| other | `unexpected` | Generic failure |

## 10. About Page

Route:

```text
/about
```

Primary code:

- `learninfive-client/src/features/about/index.tsx`

Behavior:

- Explains the purpose of LearnInFive.
- Shows a disclaimer about possible loading delays due to free-tier services.
- Describes the five-minute learning use case.

## 11. License Page

Route:

```text
/license
```

Primary code:

- `learninfive-client/src/features/license/index.tsx`
- `learninfive-client/LICENSE`
- `learninfive-server/LICENSE`

Behavior:

- Explains that the project is licensed under GNU AGPL v3.
- Links to the official AGPL v3 license.
- Links to the source code repository referenced by the page.
- Includes contact links.

## 12. Layout and Navigation

Primary code:

- `learninfive-client/src/shared/layouts/Base.tsx`
- `learninfive-client/src/shared/components/header/index.tsx`
- `learninfive-client/src/shared/components/footer/index.tsx`
- `learninfive-client/src/shared/hooks/useFooterLinks.tsx`

Behavior:

- Most pages use `BaseLayout`.
- Header contains theme switching and auth controls.
- Footer renders contextual links depending on the current route.
- Main content is centered and scrollable within the viewport.

## 13. Rate Limiting and Loading Behavior

Primary code:

- `learninfive-server/utils/limiter.ts`
- `learninfive-client/src/features/topic/index.tsx`

Behavior:

- Server limits clients to 50 requests per 15-minute window.
- Topic loading shows a spinner and a message while fetching or retrying.
- The topic query retries over roughly two minutes when the backend reports generation is in progress.
