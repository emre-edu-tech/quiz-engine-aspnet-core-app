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