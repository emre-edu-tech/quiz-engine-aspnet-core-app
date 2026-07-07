# Quiz Engine (ASP.NET Core + React)

## 1. Overview
A lightweight, self-hosted quiz engine built with ASP.NET Core 8 Web API (backend) and **React 18 + Vite** (frontend).
- The **Admin Panel** is password‑protected (JWT authentication).
- **Save** button stores the draft; **Publish** button generates a unique public link (slug) that students can visit without authentication.

## 2. Core Logic
| Requirement | Implementation |
|---|---|
| Quiz has NO subject | Subject is strictly a property of a Question. The Quiz entity has no SubjectId field. |
| Subject is optional | Question.SubjectId is nullable (int?). If null, the question simply belongs to no category. |