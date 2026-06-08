# Diagrams

These diagrams use Mermaid syntax and can be rendered by GitHub, many Markdown editors, and Mermaid CLI.

## System Context

```mermaid
flowchart LR
  Visitor["Anonymous visitor"] --> Client["React client"]
  User["Signed-in user"] --> Client
  Client --> Clerk["Clerk auth"]
  Client --> API["Express API"]
  API --> Mongo["MongoDB Atlas"]
  API --> OpenAI["OpenAI API"]
  API --> ClerkVerify["Clerk token verification"]
```

## Runtime Architecture

```mermaid
flowchart TB
  subgraph Client["learninfive-client"]
    Main["main.tsx"]
    Router["TanStack Router"]
    Query["React Query"]
    Features["Feature modules"]
    Axios["Axios instance"]
    Theme["Theme store"]
  end

  subgraph Server["learninfive-server"]
    Express["Express app"]
    TopicRoutes["/topics routes"]
    UserRoutes["/users routes"]
    TopicController["Topic controller"]
    UserController["User controller"]
    ModelController["OpenAI model controller"]
    DBUtils["MongoDB utilities"]
  end

  Main --> Router
  Main --> Query
  Main --> Theme
  Router --> Features
  Features --> Axios
  Axios --> Express
  Express --> TopicRoutes
  Express --> UserRoutes
  TopicRoutes --> TopicController
  UserRoutes --> UserController
  TopicController --> ModelController
  TopicController --> DBUtils
  UserController --> DBUtils
  DBUtils --> Mongo["MongoDB"]
  ModelController --> OpenAI["OpenAI"]
```

## Client Route Map

```mermaid
flowchart TB
  Root["Root route App + Outlet"] --> Topic["/ TopicPage"]
  Root --> SignIn["/sign-in RedirectToSignIn"]
  Root --> SignUp["/sign-up RedirectToSignUp"]
  Root --> Complete["/complete-profile"]
  Root --> Edit["/edit-profile-preferences"]
  Root --> About["/about"]
  Root --> License["/license"]
  Root --> Error["/error"]

  Topic --> TopicGuard["If signed in and profile missing, redirect to complete profile"]
  Complete --> CompleteGuard["Requires token and missing profile"]
  Edit --> EditGuard["If signed in and profile missing, redirect to complete profile"]
```

## Topic Request Sequence

```mermaid
sequenceDiagram
  participant Browser
  participant Client as React Client
  participant API as Express API
  participant DB as MongoDB
  participant AI as OpenAI

  Browser->>Client: Open /
  Client->>API: GET /topics/get-topic
  API->>DB: Find today's matching topic
  alt Topic exists
    DB-->>API: Topic
    API-->>Client: Existing topic
  else Topic missing
    API->>DB: Read previous concepts
    API->>AI: Request generated topic JSON
    AI-->>API: Topic JSON
    API->>DB: Insert topic
    API-->>Client: New topic
  end
  Client-->>Browser: Render definition, analogy, examples, quiz
```

## Authenticated Profile Completion

```mermaid
sequenceDiagram
  participant User
  participant Client as React Client
  participant Clerk
  participant API as Express API
  participant DB as MongoDB

  User->>Client: Visit /
  Client->>Clerk: getToken()
  Clerk-->>Client: JWT
  Client->>API: GET /users/check-user-profile
  API->>Clerk: Verify token
  API->>DB: Find user by Clerk sub
  DB-->>API: No profile
  API-->>Client: exists false
  Client-->>User: Redirect /complete-profile
  User->>Client: Submit profile
  Client->>API: POST /users/complete-profile
  API->>Clerk: Verify token
  API->>DB: Insert user profile
  API-->>Client: success
  Client-->>User: Navigate /
```

## Personalized Topic Generation

```mermaid
sequenceDiagram
  participant Client as React Client
  participant API as Express API
  participant Users as users.user
  participant Topics as topics.topic
  participant AI as OpenAI

  Client->>API: GET /topics/get-topic with Bearer token
  API->>API: Decode token sub as userId
  API->>Topics: Find topics by userId
  API->>Users: Find user by userId
  alt Today topic exists
    API->>Users: Check answeredQuizzes
    API-->>Client: Return topic, optionally with quiz.userAnswer
  else Today topic missing
    API->>API: Extract past topic concepts
    API->>AI: Generate personalized topic
    AI-->>API: Topic JSON
    API->>Topics: Insert personalized topic
    API->>Users: Append to pastTopics
    API-->>Client: Return new topic
  end
```

## Quiz Answer Flow

```mermaid
sequenceDiagram
  participant User
  participant Client as React Client
  participant API as Express API
  participant Topics as topics.topic
  participant Users as users.user

  User->>Client: Select answer and submit
  Client->>API: POST /topics/answer-quiz
  API->>Topics: Find topic
  API->>API: Compare answer to quiz.rightAnswer
  alt Authenticated
    API->>Users: Append answeredQuizzes result
    API-->>Client: correct true or false
  else Anonymous
    API-->>Client: correct true or false
    Client->>Client: Store result in localStorage
  end
  Client-->>User: Show result
```

## Data Relationships

```mermaid
erDiagram
  USER {
    string userId
    string csLevel
    string goals
    string preferences
    string topicsToAvoid
  }

  PAST_TOPIC {
    string id
    string concept
  }

  ANSWERED_QUIZ {
    string id
    boolean correctness
  }

  TOPIC {
    string id
    string concept
    string definition
    string realWorldAnalogy
    date date
    boolean public
    string userId
  }

  TOPIC_EXAMPLE {
    string language
    string code
  }

  QUIZ {
    string id
    string question
    string rightAnswer
    boolean userAnswer
  }

  QUIZ_ANSWER {
    string id
    string content
  }

  USER ||--o{ PAST_TOPIC : "stores"
  USER ||--o{ ANSWERED_QUIZ : "stores"
  USER ||--o{ TOPIC : "receives personalized"
  TOPIC ||--o{ TOPIC_EXAMPLE : "has"
  TOPIC ||--|| QUIZ : "has"
  QUIZ ||--o{ QUIZ_ANSWER : "has"
```

## Deployment Shape

```mermaid
flowchart LR
  Browser["Browser"] --> Vercel["Static client hosting"]
  Browser --> APIHost["API host"]
  Vercel --> ClientAssets["Vite build assets"]
  APIHost --> Express["Node Express server"]
  Express --> Mongo["MongoDB Atlas"]
  Express --> OpenAI["OpenAI"]
  Browser --> Clerk["Clerk hosted auth"]
  Express --> ClerkJWT["Clerk JWT verification"]
```

