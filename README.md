# WriteEasy

**Capture, organize, and revisit what you learn.**

WriteEasy is a note-management API for students keeping track of classes, projects, and student life. This repository contains its original **TypeScript, Express, and MongoDB backend**. It provides account authentication, personal notes, user-created categories, and basic search.

The repository was originally named **Documentos**, and the original package metadata refers to **Notely**. WriteEasy is the product's new name; this documentation update leaves the implementation unchanged.

## Current features

- Register and log in with a username, email address, and password.
- Authenticate protected requests using a JWT stored in an HTTP-only cookie.
- View your profile and change your username.
- Create notes with a required title, optional text details, and an optional category.
- List your notes, retrieve an individual note, or search by title, details, and category.
- Update individual note fields and permanently delete notes.
- Create, list, rename, and delete your own categories.
- Preserve notes when deleting a category: their category reference becomes `null`.
- Validate inputs and check ownership of notes and categories.
- Exercise API behavior through Jest/Supertest integration tests.

There is no frontend in this repository. Use an API client such as Bruno or Postman, or scripts, to interact with the backend.

## Technology and architecture

| Component | Technology |
|---|---|
| Language | TypeScript |
| HTTP server | Express |
| Database and models | MongoDB and Mongoose |
| Authentication | JSON Web Tokens and cookies |
| Password hashing | bcrypt |
| Validation | express-validator |
| Tests | Jest, Supertest, mongodb-memory-server |

The application is a monolithic API organized into technical layers:

```text
Request → middleware → route/validation → controller → Mongoose → MongoDB
```

Controllers coordinate database operations and HTTP responses. Some validators also query the database to check resource existence and ownership. User-model middleware hashes passwords before saving them.

```text
src/
├── index.ts                 # Connect to MongoDB, then start listening
├── config/                  # Server assembly, database, environment settings
└── api/
    ├── routes/
    ├── controllers/
    ├── models/
    ├── validation/
    ├── middlewares/
    ├── helpers/
    ├── errors/
    └── interfaces/
tests/                       # API integration tests
```

## Local setup

### Prerequisites

- Node.js and npm. Node.js 20 is a suggested starting point based on the project's Node type dependencies; the package does not declare an enforced runtime version.
- A running MongoDB instance, locally or remotely.

Clone the existing repository:

```sh
git clone git@github.com:ghislainnkundayezu/Documentos.git
cd Documentos
npm ci
cp .env.example .env
```

The clone URL reflects the current repository name. It should be updated when the GitHub repository rename is completed.

Edit `.env` with your local settings:

```dotenv
PORT=5000
DATABASE_COLLECTION=writeeasy-local
DATABASE_URL=mongodb://127.0.0.1:27017/
JWT_SECRET=replace-with-a-generated-secret
```

Generate your own secret, for example:

```sh
node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"
```

Replace the placeholder in `.env` with that output. Do not reuse the example file's signing secret or commit your `.env`.

`DATABASE_COLLECTION` is the historical environment variable name; the database connector uses it as MongoDB's **database name**.

Start the API:

```sh
npm run dev
```

The application connects to MongoDB before opening its HTTP listener. The port defaults to `3000` when `PORT` is not set. The current code binds to `os.hostname()` and does **not** read the `HOST_IP_ADDRESS` variable included in `.env.example`. Use the host reachable on your machine; hostname resolution may need local configuration.

`GET /` and `GET /api` return a welcome response listing the API resources. They are not database health checks.

### Authentication when testing locally

Registration and login set an `auth-token` cookie with a one-hour lifetime. Protected endpoints read this cookie rather than a Bearer API key.

The cookie is configured with `Secure`, `HttpOnly`, and `SameSite=None`. The Express listener itself serves HTTP. To exercise normal secure-cookie behavior, place it behind a local HTTPS proxy with a trusted certificate. For HTTP-only development, an API client can explicitly attach the returned cookie; automatic cookie handling depends on the client and hostname. Do not assume a browser will send a Secure cookie over plain HTTP.

## API reference

All paths below belong to this implementation's `/api` namespace, not the planned Java backend's `/api/v1` contract.

### Authentication and profile

| Method | Path | Request body / purpose |
|---|---|---|
| `POST` | `/api/auth/register` | `username`, `email`, `password`; creates account and sets cookie |
| `POST` | `/api/auth/login` | `username`, `email`, `password`; sets cookie |
| `POST` | `/api/auth/logout` | Clears authentication cookie |
| `GET` | `/api/users` | Returns authenticated user's profile |
| `PATCH` | `/api/users` | `username`; updates username |

Registration and login both require all three fields. Usernames are alphanumeric, at least three characters, and normalized to lowercase. Passwords require at least eight characters. Profile and note/category endpoints require authentication.

Example registration body:

```json
{
  "username": "demostudent",
  "email": "student@example.com",
  "password": "replace-with-your-test-password"
}
```

### Notes

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/notes` | Create a note |
| `GET` | `/api/notes` | List/search your notes |
| `GET` | `/api/notes/:noteId` | Retrieve a note; response data is an array |
| `PATCH` | `/api/notes/:noteId/:fieldToUpdate` | Update one field using `newValue` |
| `DELETE` | `/api/notes/:noteId` | Permanently delete a note |

Example creation body:

```json
{
  "title": "Biology lecture",
  "details": "Review photosynthesis before the next class."
}
```

Optionally supply `categoryId` with the MongoDB ObjectId of one of your categories. Omit it to create an uncategorized note. The owner is taken from authentication, not supplied in the body.

Search parameters can be combined:

```http
GET /api/notes?title=biology
GET /api/notes?details=photosynthesis
GET /api/notes?categoryId=<category-id>
GET /api/notes?title=biology&categoryId=<category-id>
```

Title and details searches use case-insensitive regular expressions. Title search input is restricted to word characters and whitespace. There is no pagination or dedicated uncategorized filter.

Update a title:

```http
PATCH /api/notes/<note-id>/title
Content-Type: application/json

{"newValue": "Biology revision notes"}
```

Update details with the `/details` field path or assign a category with `/category` and a category ObjectId as `newValue`. The update validator also accepts a `status` field, but its values differ from the model; see the limitations below.

Creating notes returns `201` with a confirmation message, not the created note's ID. List notes afterward to obtain IDs. Titles are lowercased; details are escaped and trimmed by request validation. This version does not implement a Markdown editor or guarantee lossless Markdown storage.

### Categories

| Method | Path | Request body / purpose |
|---|---|---|
| `POST` | `/api/categories` | `label`; create category |
| `GET` | `/api/categories` | List your categories |
| `PATCH` | `/api/categories/:categoryId` | `newLabel`; rename category |
| `DELETE` | `/api/categories/:categoryId` | Remove category and uncategorize its notes |

Labels must be alphanumeric and are stored in lowercase. For example, use `biology101` or `capstone`. Categories have no subcategories. Creation returns a confirmation message; list categories to obtain their IDs.

## Example API workflow

Set `API_BASE` to your configured HTTPS endpoint, then use a cookie jar:

```sh
API_BASE=https://localhost:8443

curl -i -c /tmp/writeeasy-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"username":"demostudent","email":"student@example.com","password":"replace-with-your-test-password"}' \
  "$API_BASE/api/auth/register"

curl -i -b /tmp/writeeasy-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"label":"biology101"}' \
  "$API_BASE/api/categories"

curl -i -b /tmp/writeeasy-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"title":"Biology lecture","details":"Review photosynthesis."}' \
  "$API_BASE/api/notes"

curl -i -b /tmp/writeeasy-cookies.txt \
  "$API_BASE/api/notes?title=biology"
```

The HTTPS URL assumes you have configured a local proxy; the repository does not supply one. Treat the cookie jar as a credential and keep it out of version control. In Postman or Bruno, use the same sequence: register/login, retain the cookie, create content, then list/search and update it.

## Response behavior

Successful responses commonly include `success`, `message`, and, for reads, `data`.

Errors use this shape:

```json
{
  "success": false,
  "title": "ValidationError",
  "description": "Invalid Data",
  "details": []
}
```

Validation details depend on the rejected input. A list of notes with no results returns `204 No Content`; a category list with no categories returns `404`. Some errors fall back to `500` rather than a specific application status.

## Tests

```sh
npm test
```

The script runs Jest with coverage and open-handle detection. Tests cover authentication, users, notes, categories, and basic routes. Database-backed suites use `mongodb-memory-server`, which may download a MongoDB binary on the first run. A local `.env` containing a signing secret is needed for authentication tests.

The test suite is part of the existing implementation; its presence is not a claim that every edge case is covered or that it currently passes in every environment.

## Current limitations and planned evolution

This is the original implementation retained as a learning baseline:

- Updating a field to its existing value can produce an error.
- The note model allows statuses `ongoing` and `complete`, while update validation allows `ongoing` and `finished`; status handling is inconsistent.
- Category removal uses separate database writes rather than an atomic transaction.
- Logging out clears the client cookie; there is no server-side token revocation.
- Search is unpaginated, and responses are not fully uniform.
- There is no revision history, conflict detection, idempotent creation, trash/restore, API-key provisioning, or live collaboration.

A separate **`writeeasy-backend`** Java/Spring Boot rebuild is planned with PostgreSQL, JPA/Hibernate, Flyway, API-key authentication, reliable versioned saves, history, and repeatable performance tests. Those capabilities are roadmap items, not features of this repository. A repository link will be added when that backend exists.

## Author and license

Built by Ghislain Nkundayezu. The package declares the ISC license.
