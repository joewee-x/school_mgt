# School Management System - REST API

A backend-only REST API for managing a school: students, teachers, classes, subjects and
student results, protected by JWT authentication and role based access control.

Built with **Node.js + Express 5**. There is **no frontend and no real database** - all data
lives in in-memory arrays that are re-seeded every time the server starts.

---

## Table of contents

1. [Features](#features)
2. [Tech stack](#tech-stack)
3. [Setup](#setup)
4. [Environment variables](#environment-variables)
5. [Demo accounts](#demo-accounts)
6. [Running the server](#running-the-server)
7. [API conventions](#api-conventions)
8. [Endpoints](#endpoints)
9. [Authorization matrix](#authorization-matrix)
10. [Data model](#data-model)
11. [Project structure](#project-structure)
12. [Security notes](#security-notes)
13. [Testing](#testing)

---

## Features

- **Authentication** - register and log in, `bcrypt` password hashing, JWT access tokens
- **Authentication middleware** - verifies the token and attaches the caller to `req.user`
- **Authorization middleware** - role based access control for `admin`, `teacher`, `student`
- **CRUD** for students, teachers, classes, subjects and results
- **Ownership rules** - a student can only read their own profile, class and results
- **Validation** - every body, route param and query string is validated with `zod`
- **Global error handling** - one consistent error envelope, no stack traces leaked
- **Request logger** - method, URL, status code, timestamp, user and role
- **Rate limiting** - strict limits on `/api/auth/register` and `/api/auth/login`
- **Relational integrity** - foreign keys are checked on write and protected on delete
- **Seed data** - 6 accounts (1 admin, 2 teachers, 3 students), 2 classes, 4 subjects,
  2 teacher records and 6 results on boot

---

## Tech stack

| Package | Purpose |
| --- | --- |
| `express` | HTTP server and routing |
| `bcrypt` | Password hashing |
| `jsonwebtoken` | Signing and verifying tokens |
| `zod` | Request validation schemas |
| `morgan` | HTTP request logging |
| `express-rate-limit` | Rate limiting |
| `dotenv` | Environment variables |
| `nodemon` | Auto-restart in development |

---

## Setup

```bash
# 1. install dependencies
npm install

# 2. create your environment file
cp .env.example .env

# 3. generate a strong JWT secret and paste it into .env
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"

# 4. start the server
npm run dev
```

The server starts on <http://localhost:8080> and prints the seeded demo accounts to the console.

> `JWT_SECRET` is mandatory. If it is missing the server refuses to boot rather than signing
> tokens with an undefined secret.

---

## Environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `PORT` | `8080` | Port the server listens on |
| `NODE_ENV` | `development` | Environment name |
| `JWT_SECRET` | - | **Required.** Secret used to sign/verify tokens |
| `JWT_EXPIRES_IN` | `1h` | Token lifetime |
| `SALT_ROUNDS` | `12` | bcrypt cost factor |
| `ALLOW_PUBLIC_ADMIN_REGISTRATION` | `true` | Set to `false` so only an existing admin can create admin accounts |
| `AUTH_RATE_LIMIT_WINDOW_MS` | `900000` | Window for the auth limiter (15 min) |
| `AUTH_RATE_LIMIT_MAX` | `10` | Max register/login requests per window |
| `API_RATE_LIMIT_WINDOW_MS` | `900000` | Window for the global limiter (15 min) |
| `API_RATE_LIMIT_MAX` | `100` | Max requests per window for every other endpoint |
| `SEED_DEMO_DATA` | `true` | Set to `false` to start with an empty database |

`.env` is gitignored. Only `.env.example` is committed.

---

## Demo accounts

Seeded automatically on boot (see `src/data/seed.js`):

| Role | Email | Password |
| --- | --- | --- |
| admin | `admin@school.test` | `Admin@123` |
| teacher | `grace.nwosu@school.test` | `Teach@123` |
| teacher | `ibrahim.musa@school.test` | `Teach@123` |
| student | `ada.obi@school.test` | `Stu@123` |
| student | `chidi.eze@school.test` | `Stu@123` |
| student | `fatima.bello@school.test` | `Stu@123` |

The student and teacher accounts are already linked to their student/teacher records, so they
can immediately read their own data.

---

## Running the server

```bash
npm run dev     # development, restarts on file changes (nodemon)
npm start       # production style, plain node
```

Useful URLs:

- `GET /` - service banner
- `GET /api/health` - health check

---

## API conventions

**Successful response**

```json
{
  "success": true,
  "message": "Student retrieved",
  "data": { "id": "stu_...", "name": "Ada Obi", "...": "..." }
}
```

List endpoints add a `meta` block:

```json
{
  "success": true,
  "message": "Students retrieved",
  "data": [],
  "meta": { "total": 3, "page": 1, "limit": 20, "totalPages": 1 }
}
```

**Error response**

```json
{
  "success": false,
  "message": "Student not found"
}
```

Validation errors add a `details` array:

```json
{
  "success": false,
  "message": "Validation failed for \"body\"",
  "details": [{ "field": "email", "message": "Email must be a valid email address" }]
}
```

**Status codes**

| Code | Used for |
| --- | --- |
| `200` | Successful read / update / delete |
| `201` | Resource created |
| `400` | Validation failed, malformed JSON, broken foreign key |
| `401` | Missing, invalid or expired token, wrong credentials |
| `403` | Authenticated but not allowed |
| `404` | Record or route not found |
| `409` | Conflict (duplicate email, delete blocked by dependencies) |
| `429` | Rate limit exceeded |
| `500` | Unexpected server error (message is generic, details stay on the server) |

**Conventions**

- `PUT` is a **partial update**: send only the fields you want to change. An empty body is a `400`.
  Because the validator drops unknown fields, a controller can hand the validated body straight to
  `store.update`, which ignores any field left out.
- IDs are prefixed UUIDs: `usr_`, `stu_`, `tch_`, `cls_`, `sub_`, `res_`.
- Authenticated requests need the header `Authorization: Bearer <token>`.
- `POST /api/auth/register` returns the created user but **no token** - log in to get one.
- All requests and responses use JSON.

---

## Endpoints

### Authentication

| Method | Endpoint | Access | Description |
| --- | --- | --- | --- |
| `POST` | `/api/auth/register` | public | Create an account |
| `POST` | `/api/auth/login` | public | Log in, returns a JWT |
| `GET` | `/api/auth/me` | any role | The caller's account and linked profile |
| `PUT` | `/api/auth/me` | any role | Update own name/phone/password |

### Students - `/api/students`

| Method | Endpoint | Access |
| --- | --- | --- |
| `POST` | `/api/students` | admin |
| `GET` | `/api/students` | admin, teacher |
| `GET` | `/api/students/:id` | admin, teacher, student (own record only) |
| `PUT` | `/api/students/:id` | admin |
| `DELETE` | `/api/students/:id` | admin |

Query parameters for `GET /api/students`: `page`, `limit`, `search`, `classId`.

### Teachers - `/api/teachers`

| Method | Endpoint | Access |
| --- | --- | --- |
| `POST` | `/api/teachers` | admin |
| `GET` | `/api/teachers` | admin |
| `GET` | `/api/teachers/:id` | admin |
| `PUT` | `/api/teachers/:id` | admin |
| `DELETE` | `/api/teachers/:id` | admin |

Query parameters for `GET /api/teachers`: `page`, `limit`, `search`, `subjectId`.

### Classes - `/api/classes`

| Method | Endpoint | Access |
| --- | --- | --- |
| `POST` | `/api/classes` | admin |
| `GET` | `/api/classes` | admin, teacher |
| `GET` | `/api/classes/:id` | admin, teacher, student (own class only) |
| `PUT` | `/api/classes/:id` | admin |
| `DELETE` | `/api/classes/:id` | admin |

Query parameters for `GET /api/classes`: `page`, `limit`, `search`, `teacherId`.

### Subjects - `/api/subjects`

| Method | Endpoint | Access |
| --- | --- | --- |
| `POST` | `/api/subjects` | admin |
| `GET` | `/api/subjects` | admin, teacher, student |
| `GET` | `/api/subjects/:id` | admin, teacher, student |
| `PUT` | `/api/subjects/:id` | admin |
| `DELETE` | `/api/subjects/:id` | admin |

Query parameters for `GET /api/subjects`: `page`, `limit`, `search`.

### Results - `/api/results`

| Method | Endpoint | Access |
| --- | --- | --- |
| `POST` | `/api/results` | admin, teacher |
| `GET` | `/api/results` | admin, teacher (all), student (own only) |
| `GET` | `/api/results/:id` | admin, teacher, student (own only) |
| `PUT` | `/api/results/:id` | admin, teacher |
| `DELETE` | `/api/results/:id` | admin |

Query parameters for `GET /api/results`: `page`, `limit`, `studentId`, `subjectId`, `classId`,
`term`, `session`. A student's filters are ignored - it always sees only its own results.

### Grading

`grade` is optional on write. When omitted it is derived from the score and recomputed whenever
the score changes:

| Score | Grade |
| --- | --- |
| 70 - 100 | A |
| 60 - 69 | B |
| 50 - 59 | C |
| 40 - 49 | D |
| 0 - 39 | F |

---

## Authorization matrix

| Action | admin | teacher | student |
| --- | :---: | :---: | :---: |
| Register / login | yes | yes | yes |
| View own profile | yes | yes | yes |
| Create / update / delete students | yes | no | no |
| View all students | yes | yes | no |
| View own student record | yes | yes | yes |
| Create / update / delete teachers | yes | no | no |
| Create / update / delete classes | yes | no | no |
| View classes | yes | yes | own class only |
| Create / update / delete subjects | yes | no | no |
| View subjects | yes | yes | yes |
| Create results | yes | yes | no |
| View results | yes | yes | own only |
| Update results | yes | yes | no |
| Delete results | yes | no | no |
| Update own account (`/api/auth/me`) | yes | yes | yes |
| Change anyone's role | no | no | no |

`401` means "I could not identify you" (no/invalid/expired token). `403` means "I know who you
are, but you are not allowed".

No endpoint can change a role or a password on someone else's behalf: `/api/auth/me` takes no
id, and the admin CRUD routes do not touch user accounts. To make a teacher an admin, seed or
register the account with that role.

---

## Data model

All records are linked by id, exactly as a real database would:

```
User (usr_) 1---1 Student (stu_) / Teacher (tch_)

Subject (sub_) 1---* Teacher (tch_)      teacher.subjectId  -> subject.id
Teacher (tch_) 1---* Class   (cls_)      class.teacherId    -> teacher.id
Class   (cls_) 1---* Student (stu_)      student.classId    -> class.id
Student (stu_) 1---* Result  (res_)      result.studentId   -> student.id
Subject (sub_) 1---* Result  (res_)      result.subjectId   -> subject.id
```

The API returns both the foreign key and a resolved name, for example:

```json
{
  "id": "stu_1b9d6c2e-....",
  "name": "Ada Obi",
  "email": "ada.obi@school.test",
  "phone": "+2348020000001",
  "classId": "cls_9f2b....",
  "class": "JSS 1 - Gold",
  "userId": "usr_3c1a....",
  "createdAt": "2026-09-27T10:30:00.000Z"
}
```

A student/teacher account is linked to its record by matching email, or explicitly with the
optional `profileId` field at registration. Foreign keys are validated on create/update, and
deletes that would orphan related records are rejected with `409`:

- a student with results cannot be deleted
- a class that still has students cannot be deleted
- a subject referenced by results or teachers cannot be deleted
- a teacher who still runs a class cannot be deleted

---

## Project structure

```
src/
├── controllers/          # request handlers
│   ├── auth.controller.js
│   ├── student.controller.js
│   ├── teacher.controller.js
│   ├── class.controller.js
│   ├── subject.controller.js
│   ├── result.controller.js
│   └── helpers.js        # shared controller helpers (404, FK/unique checks,
│                         #   filtering, search, pagination)
├── routes/               # URL -> middleware chain -> controller
│   ├── crudRouter.js     # builds the 5 standard routes from handlers + roles
│   ├── auth.route.js
│   ├── student.route.js
│   ├── teacher.route.js
│   ├── class.route.js
│   ├── subject.route.js
│   └── result.route.js
├── middleware/
│   ├── authentication.js # verifies the JWT, sets req.user
│   ├── authorization.js  # adminOnly / staffOnly / allRoles role gates
│   ├── validator.js      # zod validation for body / params / query
│   ├── errorHandler.js   # 404 handler + global error handler
│   ├── logger.js         # morgan request logger
│   └── rateLimiter.js    # auth limiter + global limiter
├── validators/           # zod schemas per resource
│   ├── common.validator.js
│   ├── auth.validator.js
│   ├── student.validator.js
│   ├── teacher.validator.js
│   ├── class.validator.js
│   ├── subject.validator.js
│   └── result.validator.js
├── utils/                # ApiError, asyncHandler, env, jwt, password, ids,
│                         #   grading, response envelope
├── data/                 # the "database"
│   ├── store.js          # in-memory collections + relation helpers
│   ├── presenters.js     # response shaping (password stripping)
│   └── seed.js           # demo data
├── app.js                # express app, middleware order, route mounting
└── server.js             # boot: env check, seed, listen
```

Request flow:

```
request
  -> security headers + body parser
  -> logger
  -> global rate limiter
  -> route
       -> authenticate        (401 when the token is missing/invalid)
       -> authorize(...)      (403 when the role is not allowed)
       -> validate(schema)    (400 on bad body/params/query)
       -> controller          (404 when the record is missing, 409 on conflicts)
  -> notFoundHandler         (404 for unknown routes)
  -> errorHandler            (one consistent error shape)
```

---

## Security notes

- Passwords are hashed with `bcrypt` using `SALT_ROUNDS` from the environment. Plain text
  passwords are never stored.
- The `password` field never leaves the server: responses go through the presenters in
  `src/data/presenters.js`, and only the registered account owner can change it.
- `JWT_SECRET` lives in `.env` (gitignored) and the server refuses to start without it.
- The role is re-read from the store on every request, so editing the `role` claim inside a
  token does not grant anything.
- Login failures return the same message for an unknown email and a wrong password, so the
  endpoint cannot be used to discover which emails are registered.
- Unknown body fields are rejected, so a typo or an attempt to smuggle in `isAdmin` fails
  validation.
- Bodies are limited to 100kb, `X-Powered-By` is disabled, and `nosniff` / `DENY` /
  `no-referrer` headers are set.
- Unknown errors are logged on the server and reported to the client as a generic `500`; stack
  traces are never sent.
- Registration and login are rate limited (10 per 15 minutes by default) to slow down brute
  force and mass account creation.
- Set `ALLOW_PUBLIC_ADMIN_REGISTRATION=false` outside local development so nobody can
  self-register as an admin. The server logs a warning while it is enabled.

---

## Testing

`postman_collection.json` contains 59 requests with assertions, grouped into health checks,
authentication, subjects, teachers, classes, students, results and error handling.

- **Full walkthrough with expected responses:** [TESTING.md](./TESTING.md) - 78 numbered steps
  covering every endpoint, role rule, validation error, the `429` rate limit and the log format.
- **Import:** Postman -> `Import` -> select `postman_collection.json` -> run the collection
  from top to bottom. Tokens and ids are stored in collection variables automatically.

> The guide deliberately tests the auth rate limit near the end. If you do that early, the next
> 10 login attempts will return `429` - restart the server (or raise `AUTH_RATE_LIMIT_MAX`) to
> reset the in-memory counter.

---
# school_mgt
