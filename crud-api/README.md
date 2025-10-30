# Blog Platform API — CRUD + Auth

Title: Simple Blog Platform

## Overview
This is a small example API for a blog platform demonstrating authentication (signup, login, forgot/reset password) and CRUD operations for posts. The spec below lists the base URL, all endpoints, and a detailed mapping for each endpoint including positive and negative request/response examples and status codes.

## Base API URL
- Local (example): `http://localhost:8000/api/v1`
- Production (example): `https://api.example.com/v1`

## Error response format
Errors in responses use the following shape:

```json
{
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": null
}
```

## Authentication
- Auth uses JSON Web Tokens (JWT) sent in the Authorization header: `Authorization: Bearer <access_token>`.
- Refresh tokens may be used for long-lived sessions via `POST /auth/refresh` (not required for this spec but recommended).

## Top-level endpoints

Auth
- [POST /auth/signup/](#signup-auth)
- [POST /auth/login/](#login-auth)
- [POST /auth/forgot-password/](#forgot-password-auth)
- [POST /auth/reset-password/](#reset-password-auth)
- [GET /auth/profile/](#get-profile-auth)

Posts (CRUD)
- [GET /posts/](#list-posts-posts)
- [POST /posts/](#create-post-posts)
- [GET /posts/{id}/](#get-post-posts)
- [PUT /posts/{id}/](#update-post-posts)
- [PATCH /posts/{id}/](#patch-post-posts)
- [DELETE /posts/{id}/](#delete-post-posts)

---

## Endpoint reference (detailed)

All request and response bodies are JSON unless otherwise stated. For each endpoint below you'll find:
- API path
- Short one-line description
- Positive and negative cases for: Request type, **Request body**, **Response body**, Status code

Model (Post)
- id: string (uuid)
- author_id: string (user id)
- title: string (required)
- body: string (markdown/html)
- tags: array[string]
- is_published: boolean
- created_at, updated_at

### Signup (auth)
- **URL:** `/auth/signup/`
- **Method:** `POST`
- **Description:** Create a new user account. Positive case returns 201 with user data. Negative cases: duplicate email (400), validation error like missing password (400).
- **Request**:
```json
{
    "name": "Alice Example",
    "email": "alice@example.com",
    "password": "P@ssw0rd!"
}
```
- **Response** (201 Created for success; 400 for errors):
```json
{
    "id": "user_123",
    "name": "Alice Example",
    "email": "alice@example.com",
    "created_at": "2025-10-30T10:00:00Z"
}
```
For duplicate email:
```json
{
    "code": "EMAIL_ALREADY_EXISTS",
    "message": "A user with this email already exists.",
    "details": null
}
```
For validation error (missing password):
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "password": "Password is required" }
}
```

### Login (auth)
- **URL:** `/auth/login/`
- **Method:** `POST`
- **Description:** Authenticate and receive an access token. Positive case returns 200 with token. Negative cases: invalid credentials (401), missing fields (400).
- **Request**:
```json
{
    "email": "alice@example.com",
    "password": "P@ssw0rd!"
}
```
- **Response** (200 OK for success; 401/400 for errors):
```json
{
    "access_token": "eyJ...",
    "token_type": "Bearer",
    "expires_in": 3600
}
```
For invalid credentials:
```json
{
    "code": "INVALID_CREDENTIALS",
    "message": "Email or password is incorrect.",
    "details": null
}
```
For missing fields:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "password": "Password is required" }
}
```

### Forgot Password (auth)
- **URL:** `/auth/forgot-password/`
- **Method:** `POST`
- **Description:** Send a password reset email/token to the user if the email exists. Positive case returns 200 with message. Negative case: invalid email format (400).
- **Request**:
```json
{
    "email": "alice@example.com"
}
```
- **Response** (200 OK for success; 400 for error):
```json
{
    "message": "If an account with that email exists, a password reset link has been sent."
}
```
For invalid email:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "email": "Invalid email format" }
}
```

### Reset Password (auth)
- **URL:** `/auth/reset-password/`
- **Method:** `POST`
- **Description:** Reset the user's password using a reset token from email. Positive case returns 200 with message. Negative cases: invalid/expired token (400), weak password (400).
- **Request**:
```json
{
    "token": "reset-token-abc",
    "password": "N3wP@ssw0rd!"
}
```
- **Response** (200 OK for success; 400 for errors):
```json
{
    "message": "Password has been reset successfully."
}
```
For invalid token:
```json
{
    "code": "INVALID_TOKEN",
    "message": "Reset token is invalid or has expired.",
    "details": null
}
```
For weak password:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "password": "Password too weak" }
}
```

### Get Profile (auth)
- **URL:** `/auth/profile/`
- **Method:** `GET`
- **Description:** Return the authenticated user's profile. Requires Authorization header. Positive case returns 200 with user data. Negative case: missing/invalid token (401).
- **Response** (200 OK for success; 401 for error):
```json
{
    "id": "user_123",
    "name": "Alice Example",
    "email": "alice@example.com",
    "created_at": "2025-10-30T10:00:00Z"
}
```
For unauthorized:
```json
{
    "code": "UNAUTHORIZED",
    "message": "Missing or invalid authentication token.",
    "details": null
}
```

### List Posts (posts)
- **URL:** `/posts/`
- **Method:** `GET`
- **Description:** List posts (public posts + optional authenticated user's drafts). Supports pagination and filters. Positive case returns 200 with data array. Negative case: invalid pagination (400).
- **Response** (200 OK for success; 400 for error):
```json
{
    "data": [
        {
            "id": "post_1",
            "author_id": "user_123",
            "title": "First post",
            "excerpt": "Short excerpt...",
            "tags": ["intro","news"],
            "is_published": true,
            "created_at": "2025-10-29T12:00:00Z"
        }
    ],
    "meta": { "page": 1, "per_page": 10, "total": 1 }
}
```
For invalid pagination:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "per_page": "Must be positive" }
}
```

### Create Post (posts)
- **URL:** `/posts/`
- **Method:** `POST`
- **Description:** Create a new post. Authentication required. Positive case returns 201 with post data. Negative cases: missing auth (401), validation error like missing title (400).
- **Request**:
```json
{
    "title": "My New Post",
    "body": "This is the content of my post.",
    "tags": ["example"],
    "is_published": false
}
```
- **Response** (201 Created for success; 401/400 for errors):
```json
{
    "id": "post_456",
    "author_id": "user_123",
    "title": "My New Post",
    "body": "This is the content of my post.",
    "tags": ["example"],
    "is_published": false,
    "created_at": "2025-10-30T11:00:00Z",
    "updated_at": "2025-10-30T11:00:00Z"
}
```
For missing auth:
```json
{
    "code": "UNAUTHORIZED",
    "message": "Missing or invalid authentication token.",
    "details": null
}
```
For validation error:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "title": "Title is required" }
}
```

### Get Post (posts)
- **URL:** `/posts/{id}/`
- **Method:** `GET`
- **Description:** Retrieve a single post by id. Public posts available to anyone; drafts only to author. Positive case returns 200 with post data. Negative cases: not found (404), draft by non-author (403).
- **Response** (200 OK for success; 404/403 for errors):
```json
{
    "id": "post_1",
    "author_id": "user_123",
    "title": "First post",
    "body": "Full content...",
    "tags": ["intro"],
    "is_published": true,
    "created_at": "2025-10-29T12:00:00Z"
}
```
For not found:
```json
{ "code": "NOT_FOUND", "message": "Post not found", "details": null }
```
For forbidden:
```json
{ "code": "FORBIDDEN", "message": "You do not have access to this resource", "details": null }
```

### Update Post (posts)
- **URL:** `/posts/{id}/`
- **Method:** `PUT`
- **Description:** Replace a post. Auth required and must be author. Positive case returns 200 with updated post. Negative cases: not author (403), validation error (400).
- **Request**:
```json
{
    "title": "Updated Title",
    "body": "Updated content",
    "tags": ["updated"],
    "is_published": true
}
```
- **Response** (200 OK for success; 403/400 for errors):
```json
{
    "id": "post_1",
    "author_id": "user_123",
    "title": "Updated Title",
    "body": "Updated content",
    "tags": ["updated"],
    "is_published": true,
    "created_at": "2025-10-29T12:00:00Z",
    "updated_at": "2025-10-30T12:00:00Z"
}
```
For not author:
```json
{ "code": "FORBIDDEN", "message": "You do not have access to this resource", "details": null }
```
For validation:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "body": "Body is required" }
}
```

### Patch Post (posts)
- **URL:** `/posts/{id}/`
- **Method:** `PATCH`
- **Description:** Partially update a post (e.g., change title or publish flag). Auth required and must be author. Positive case returns 200 with updated post. Negative case: invalid field (400).
- **Request**:
```json
{ "is_published": true }
```
- **Response** (200 OK for success; 400 for error):
```json
{
    "id": "post_1",
    "author_id": "user_123",
    "title": "First post",
    "body": "Full content...",
    "tags": ["intro"],
    "is_published": true,
    "created_at": "2025-10-29T12:00:00Z",
    "updated_at": "2025-10-30T12:30:00Z"
}
```
For invalid field:
```json
{
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": { "is_published": "Must be a boolean" }
}
```

### Delete Post (posts)
- **URL:** `/posts/{id}/`
- **Method:** `DELETE`
- **Description:** Delete a post. Auth required and must be author. Positive case returns 204. Negative cases: not author (403), not found (404).
- **Response** (204 No Content for success; 403/404 for errors):
```json
{ }
```
For not author:
```json
{ "code": "FORBIDDEN", "message": "You do not have access to this resource", "details": null }
```
For not found:
```json
{ "code": "NOT_FOUND", "message": "Post not found", "details": null }
```

