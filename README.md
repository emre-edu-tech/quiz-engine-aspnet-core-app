# Quiz Engine (ASP.NET Core + React)

## Overview
A lightweight, self-hosted quiz engine built with ASP.NET Core 8 Web API (backend) and **React 18 + Vite** (frontend).
- The **Admin Panel** is password‑protected (JWT authentication).
- **Save** button stores the draft.
- **Publish** button generates a `unique public link (slug)` that students can visit **without authentication**.
- **Unpublish** button sets the `IsPublished` property to `false` but preserves the unique public link.