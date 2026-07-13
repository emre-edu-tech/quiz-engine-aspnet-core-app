# Quiz Engine (ASP.NET Core + React)

## 1. Overview
A lightweight, self-hosted quiz engine built with ASP.NET Core 8 Web API (backend) and **React 18 + Vite** (frontend).
- The **Admin Panel** is password‑protected (JWT authentication).
- **Save** button stores the draft; **Publish** button generates a unique public link (slug) that students can visit without authentication.

## 2. Core Logic
| Requirement | Implementation |
|---|---|
| **Quiz has NO subject** | Subject is strictly a property of a `Question`. The `Quiz` entity has no `SubjectId` field. |
| **Subject is optional** | `Question.SubjectId` is nullable (`int?`). If null, the question simply belongs to no category. |
| **No filtering during creation** | The admin does **not** pick a subject for the whole quiz. The dropdown for subject only exists on the individual question form. |
| **Delete Category constraint** | If a `Subject` has at least one `Question` referencing it, the API **must** return a `409 Conflict` error and block the deletion. |
| **Save, Publish and Unpublish** | - **Save**: Persists all question/choice data to the database. `IsPublished` property/field remains false. <br> - **Publish**: Generates a unique `Slug` (GUID), sets `IsPublished = true`, and returns the full public URL. <br> - **Unpublish**: Sets `IsPublished = false`, and removes the public URL. |
| **Database** | **SQLite** for development/production (file‑based, zero config). Leave EF Core migrations ready to switch to **MariaDB 10.11** by changing the provider and connection string. |

## 3. Technology Stack
| Layer | Technology | Justification |
|---|---|---|
| **Backend** | ASP.NET Core 8 Web API (Controllers) | Mature, high‑performance, native JSON serialization. Built‑in Dependency Injection. |
| **ORM** | Entity Framework Core 8 | Code‑first approach. Works identically for SQLite and MariaDB. |
| **Auth** | ASP.NET Core Identity + JWT Bearer | Secure, handles user/password hashing, easy to integrate. |
| **Frontend** | React 18 + Vite | Vite is faster than CRA (create-react-app). No Next.js lock‑in. |
| **Styling** | Tailwind CSS v3 | Utility‑first, fast UI development. |
| **HTTP Client** | Native `fetch()` function | No third‑party library. |
| **Image Storage** | Local file system (`wwwroot/media/questions/`) | Served via static files middleware. No cloud storage. |
| **State Management** | React Context API + `useReducer` | Simple, sufficient for a single admin flow. |
| **Deployment** | Plesk (Linux) + Native ASP.NET hosting | Activate the ASP.NET Core app hosting |

## 4. Data Models (C# / EF Core)
```csharp
// Models/Subject.cs
public class Subject {
    public int Id { get; set; }
    
    [Required(ErrorMessage = "Subject name is required.")]
    [StringLength(100, MinimumLength = 1, ErrorMessage = "Subject name must be between 1 and 100 characters.")]
    public string Name { get; set; } // No default value; must be provided by the admin

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation Property
    public ICollection<Question> Questions { get; set; }

    // Constructor to initialize the Questions collection
    public Subject() {
        Questions = new List<Question>();
    }
}

// Models/Quiz.cs
public class Quiz {
    public int Id { get; set; }

    [Required(ErrorMessage = "Quiz title is required.")]
    [StringLength(200, MinimumLength = 1, ErrorMessage = "Quiz title must be between 1 and 200 characters.")]
    public string Title { get; set; }   // No default value; must be provided by the admin  

    public string? Slug { get; set; }   // Nullable - null until published
    public bool IsPublished { get; set; } = false;  // Explicit default value for clarity, a Quiz is not published by default
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;

    // The admin user who owns the Quiz (ASP.NET Core Identity)
    [Required(ErrorMessage = "Creator ID is required.")]
    // Foreign Key
    public string CreatedById { get; set; }; // No default value, must be set when creating a Quiz
    
    // Navigation Property - EF Core will populate this
    public IdentityUser CreatedBy { get; set; }

    // Navigation Property
    public ICollection<Question> Questions { get; set; }

    // Constructor to initialize the Questions collection
    public Quiz() {
        Questions = new List<Question>();
    }
}

// Models/Question.cs
public class Question {
   public int Id { get; set; }

   [Required(ErrorMessage = "Question text is required.")]
   public string Text { get; set; } // No default value; must be provided by the admin

   public string? ImagePath { get; set; }  // Optional image for the questions - Relative path: "/media/questions/xyz.jpg"
   public int Order { get; set; } = 0;  // For manual ordering (drag/drop later' just int for now and default to 0)

   // Foreign Keys
   public int QuizId { get; set; }  // Required - must belong to a Quiz
   // Navigation Property - EF Core will populate this
   public Quiz Quiz { get; set; }

   // Foreign Keys
   // Subject is nullable since not every question must have a subject
   // Not every question must have a subject.
   public int? SubjectId { get; set; }
   public Subject? Subject { get; set; }  // Navigation Property - nullable and can be null if the question has no subject

   public ICollection<Choice> Choices { get; set; }

   // Constructor to initialize the Choices collection
   public Question() {
       Choices = new List<Choice>();
   }
}

// Models/Choice.cs
public class Choice {
    public int Id { get; set; }

    [Required(ErrorMessage = "Choice text is required.")]
    [StringLength(200, MinimumLength = 1, ErrorMessage = "Choice text must be between 1 and 200 characters.")]
    public string Text { get; set; };   // No default value; must be provided by the admin

    public bool IsCorrect { get; set; } = false; // Explicit default value for clarity, a Choice is not correct by default

    // Foreign Key - Required
    public int QuestionId { get; set; } // Must belong to a Question

    // Navigation Property - EF Core will populate this
    public Question Question { get; set; }
}
```

## 5. API Endpoints

### 5.1 Authentication (`api/auth`)

| **Method** | **Endpoint** | **Description** | **Auth** |
|---|---|---|---|
| `POST` | `/api/auth/login` | Login → returns `{ accessToken, username }` | No |
| `POST` | `/api/auth/register` | (Optional) Create admin user<br>(will just be used for initial admin user creation; later, only login will be needed) | No |

### 5.2 Subjects (`api/subjects`) - *[Authorize]*

| **Method** | **Endpoint** | **Description** | **Details** |
|---|---|---|---|
| `GET` | `/api/subjects` | Lists all subjects | Subjects may be listed while creating a question to set its subject. |
| `POST` | `/api/subjects` | Create a new subject | Create { body: `{name}` } |
| `PUT` | `/api/subjects/{id}` | - **Update** a subject name<br>- { body: `{ "name": "Physics and Mechanics" }` } | 1. Checks for duplicate names, ensure name is unique (case-insensitive check)<br>2. Return `404` if subject not found.<br>3. Return `409 Conflict` if subject name already exists. |
| `DELETE` | `/api/subjects/{id}` | Delete if no questions reference to that subject | 1. Check `db.Questions.Any(q => q.SubjectId == id)`.<br>2. If true -> return `409 Conflict` with `{"error": "Cannot delete subject. It is assigned to existing questions."}`.<br>3. Else, delete the subject. |

### 5.3 Quizzes (`api/quizzes`) - *[Authorize]*

| **Method** | **Endpoint** | **Description** | **Details** |
|---|---|---|---|
| `GET` | `/api/quizzes` | List all quizzes owned by the current user | Quiz fields -> Title, IsPublished, Slug, UpdatedAt |
| `POST` | `/api/quizzes` | Creates a new empty quiz (just a title). | Returns `{id, title}`. |
| `GET` | `/api/quizzes/{id}` | Returns the full quiz. | **With all nested questions and choices of that quiz** (including `IsCorrect` flag for editing) |
| `PUT` | `/api/quizzes/{id}` | Update quiz title { body: `{ "title": "New Quiz Title" }` } | 1. Check ownership (quiz.CreatedById == currentUserId). Only the quiz owner can update.<br>2. Only `Title` can be updated via this endpoint (other fields are managed elsewhere). |
| `DELETE` | `/api/quizzes/{id}` | Deletes the quiz and all its questions/choices (cascade). | No question and its choices will be hanging around when a quiz is removed. |

### 5.4 Quiz Actions (`/api/quizzes/{id}/...`) - Save/Publish/Unpublish  - *[Authorize]*

| **Method** | **Endpoint** | **Description** | **Details** |
|---|---|---|---|
| `POST` | `/api/quizzes/{id}/save` | Saves updated quiz information and updates `UpdatedAt` timestamp. | Returns the updated quiz object (No slug changes). |
| `POST` | `/api/quizzes/{id}/publish` | Generate slug (if null), set IsPublished = true | - If `IsPublished == false` **and** `Slug == null` -> generate new GUID, set `IsPublished = true`.<br>- If `IsPublished == false` **and** `Slug != null` (Unpublished but had a slug) -> set `IsPublished = true`, keep the existing `Slug`.<br>- If `IsPublished == true` -> just return the existing `Slug` (no changes). |
| `POST` | `/api/quizzes/{id}/unpublish` | Sets `IsPublished = false`. The `Slug` is preserved (kept in the database) | Admin can re-publish later with the `same link`. |

### 5.5 Question & Choices (`api/quizzes/{quizId}/questions`) - *[Authorize]*

| **Method** | **Endpoint** | **Description** | **Details** |
|---|---|---|---|
| POST | /api/quizzes/{quizId}/questions | Expects `{text, image (file, optional), subjectId (optional), choices: [{text, isCorrect}, ...]}`. | - Saves the image to `wwwroot/media/questions/` if provided.<br>- Ensures *exactly one* choice has `isCorrect = true` (validation). |
| PUT | /api/questions/{id} | Expects `{text, image (file, optional), subjectId (optional), choices: [{text, isCorrect}, ...]}`. | Updates text, subject, choices, and optionally (if provided) replaces the image. |
| DELETE | /api/questions/{id} | Expects `questionId` to perform the deletion. | Deletes the question and all its choices (cascade). |

### 5.6 Public Interface (No Auth) - `/api/public`
- `GET /api/public/quizzes/{slug}` -> Returns the quiz **but strips the `IsCorrect` flag** from choices. Includes question text, image (if available), and choice IDs/text.
- `POST /api/public/quizzes/{slug}/submit` -> Body: `{ answers: {"questionId": "choiceId", ...} }`.
    1. The total number of questions is calculated as `db.Questions.Count(q => q.QuizId == quizId)`.
    2. The submitted answers are iterated. For each `(questionId, choiceId)` pair:
        - We fetch the choice and check `IsCorrect`.
        - If correct, we increment the `correctCount`.
    3. The response returns:
        - `correctCount` (number of correct answers)
        - `totalQuestions` (the total number of questions in that quiz)
        - `scorePercentage` (rounded to 2 decimals) - backend or frontend can compute this.
