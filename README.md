# Medin

An invite-only professional network where doctors share clinical cases, discuss them, and follow colleagues.

- **Frontend:** Next.js 15 (App Router), React 18, TypeScript, Tailwind CSS v4, TanStack Query, TipTap editor
- **Backend:** FastAPI, SQLAlchemy 2, PostgreSQL, Alembic migrations, pytest
- **Auth:** email + password, bcrypt hashes, JWT in an httpOnly cookie
- **Images:** Cloudinary (optional for local development)

```
medin/
├── backend/            FastAPI app
│   ├── app/
│   │   ├── main.py           creates the app, registers routers
│   │   ├── config.py         settings from environment variables
│   │   ├── database.py       SQLAlchemy engine + get_db()
│   │   ├── models.py         database tables
│   │   ├── schemas.py        request/response shapes (Pydantic)
│   │   ├── security.py       password hashing + JWT
│   │   ├── dependencies.py   get_current_user()
│   │   ├── post_helpers.py   who can see which post, building post responses
│   │   ├── pagination.py     cursor pagination
│   │   ├── invites.py        invite links
│   │   ├── uploads.py        image uploads
│   │   └── routers/          one file per area: auth, posts, comments, users, …
│   ├── migrations/     Alembic migrations (database schema history)
│   ├── scripts/        seed.py (demo data), create_invite.py
│   └── tests/          pytest tests against a real Postgres database
├── frontend/           Next.js app
│   ├── app/            pages (folders = URLs)
│   ├── components/     ui/ is the design system; the rest are feature components
│   ├── hooks/          all data fetching (React Query)
│   ├── lib/            api() helper, types, formatting, dropdown lists
│   ├── config/         fonts
│   └── middleware.ts   sends logged-out visitors to /signin
└── docker-compose.yml  optional local Postgres
```

## Running it locally

You need **Python 3.11+**, **Node 20+** and **PostgreSQL 16**. For Postgres, either install it (on a Mac: `brew install postgresql@16 && brew services start postgresql@16`) or use Docker (step 2).

**1. Get the code**

```bash
git clone git@github.com:uddiptagogoi2000/medin.git
cd medin
```

**2. Start a database**: pick one

```bash
docker compose up -d     # Docker: starts Postgres with a "medin" and a "medin_test" database
# or
createdb medin           # Postgres installed on your machine
```

**3. Backend** (terminal 1)

```bash
cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

Open `backend/.env` and set:

| Setting        | Value                                                                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `DATABASE_URL` | Docker: keep `postgresql://postgres:postgres@localhost:5432/medin`<br>Postgres on a Mac (Homebrew): `postgresql://localhost:5432/medin`           |
| `SECRET_KEY`   | Paste the output of `python -c "import secrets; print(secrets.token_urlsafe(32))"`. Keep it secret; anyone who has it can log in as any user. |
| `CLOUDINARY_*` | Optional. Leave empty and everything works except image uploads. For uploads, use the keys from a free [cloudinary.com](https://cloudinary.com) account. |

Then:

```bash
alembic upgrade head            # creates the tables
python -m scripts.seed          # demo doctors and posts; prints a login and an invite link
uvicorn app.main:app --reload   # API on http://localhost:8000 (http://localhost:8000/docs lists every endpoint)
```

**4. Frontend** (terminal 2)

```bash
cd frontend
npm install
cp .env.example .env.local      # already points at http://localhost:8000; no edits needed
npm run dev                     # http://localhost:3000
```

**5. Try it**

Open http://localhost:3000 and log in as **asha.rao@example.com / password123** (every demo doctor uses `password123`). To try signing up, open the invite link the seed script printed. Signing up needs an invite: create more with `python -m scripts.create_invite`, or from inside the app (avatar menu → *Invite a colleague*).

To start again with fresh demo data, empty the database (`dropdb medin && createdb medin`, or `docker compose down -v && docker compose up -d`), then run `alembic upgrade head` and `python -m scripts.seed` again.

**Tests**

The tests use a separate `medin_test` database (they wipe it between tests).

```bash
cd backend
createdb medin_test             # skip with Docker; it's created for you
pytest

# With Docker, tell pytest the username and password:
TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/medin_test pytest
```

## How it works

### One origin: the frontend proxies the API

The browser only ever talks to the Next.js app. Requests to `/api/...` are forwarded to FastAPI by a rewrite in `frontend/next.config.js`:

```
browser ──/api/posts──▶ Next.js (:3000) ──/posts──▶ FastAPI (:8000) ──▶ Postgres
```

Because of this, the login cookie is an ordinary first-party cookie, and no CORS setup is needed.

### Authentication

1. **Sign up** (`POST /auth/signup`) needs a valid invite token. The password is hashed with **bcrypt** (salted and deliberately slow), so the database never stores it.
2. On sign up or log in, the backend creates a **JWT** containing the user id and an expiry (7 days), signed with `SECRET_KEY`.
3. The JWT is stored in an **httpOnly cookie**. JavaScript can't read it, so an XSS bug can't steal it. `SameSite=Lax` stops other sites from sending it with form posts (CSRF), and `Secure` (in production) means HTTPS only.
4. Every protected endpoint depends on `get_current_user()`. It reads the cookie, checks the signature and expiry, and loads the user, or responds with **401**.
5. `middleware.ts` only checks that the cookie *exists* to decide which pages to show. The real check happens on every API call. If one returns 401, the frontend signs out and goes to `/signin`.

Login failures always return the same message and run a bcrypt check even for unknown emails, so attackers can't tell which emails have accounts.

### Invites

Members create single-use links (`POST /invites`) that expire after 14 days. Only a SHA-256 **hash** of the token is stored, so a database leak doesn't leak usable links. When someone signs up, the invite row is locked (`SELECT … FOR UPDATE`) and creating the user and using up the invite happen in **one transaction**, so the same link can't be used twice at the same moment.

### Who can see a post

| Visibility    | Who sees it                   |
| ------------- | ----------------------------- |
| `public`      | every logged-in doctor        |
| `connections` | the author and their followers |
| `private`     | only the author (a draft)     |

The rule is written once, in `post_helpers.visible_to()`, and every endpoint that returns posts uses it: feed, post page, profile, search, comments, likes. A post you can't see returns **404**, the same as one that doesn't exist, so private posts can't be found by guessing ids.

**Anonymous posts** never send the author to the browser (`author: null`). They don't appear on the author's profile and aren't matched by searching the author's name. The author still sees `is_mine: true`, so they can edit or delete them.

### Avoiding N+1 queries

Showing a page of posts needs, for each post, its author, like/comment/repost counts, and whether *you* liked, reposted or follow the author. Querying that per post would be 1 + 6×N queries. Instead, `serialize_posts()` runs a fixed handful of queries for the whole page (authors via a JOIN, counts via `GROUP BY`). The test `test_feed_query_count_does_not_grow_with_page_size` proves the count is the same for 2 posts and 12.

### Cursor pagination

Lists use `?before_id=<last id you saw>` instead of `?offset=20`. With offsets, a new post arriving while you scroll shifts every page, so you see duplicates. A cursor always continues from the same place, and the database jumps straight to it using the primary-key index. Each response includes `next_cursor` (null on the last page).

### Idempotent likes and follows

Like = `POST /posts/{id}/like`, unlike = `DELETE`. Sending the same request twice gives the same result, so a double click can't undo a like. The join tables (`likes`, `follows`, `reposts`) use a **composite primary key**, so the database itself prevents duplicates, and inserts use `ON CONFLICT DO NOTHING`.

### Frontend data flow

All server data goes through React Query hooks in `frontend/hooks/`. The same post can be on screen in several places (feed, profile, search), so mutations update every cached copy with `updateCachedPosts()`. A like shows up everywhere at once. Likes and reposts are **optimistic**: the button changes immediately and rolls back if the request fails.

The rich-text editor is large, so it's loaded only when you open the composer or a comment box (`next/dynamic`). Saved posts are displayed by `RichTextView`, which turns the editor's JSON into React elements. No HTML strings are injected, so a post can't run scripts in someone else's browser.

### Design system

`frontend/components/ui/` holds every visual building block: Button, fields, Avatar, Tag/Badge, Modal (built on the native `<dialog>`), Menu, Toast, Tabs, empty/error states and skeletons. Colours, fonts and shadows are tokens in `frontend/styles/globals.css`. Pages are composed only from these, so the whole app stays consistent.

## Database migrations

The schema lives in `backend/app/models.py`. After changing a model:

```bash
alembic revision --autogenerate -m "describe the change"   # writes a migration file
alembic upgrade head                                       # applies it
```

## Deploying

- **Backend:** any Python host. Set `DATABASE_URL`, `SECRET_KEY` (long and random), `COOKIE_SECURE=true`, `FRONTEND_URL`, and optionally the Cloudinary variables. Run `alembic upgrade head` on deploy.
- **Frontend:** e.g. Vercel with the root directory set to `frontend/`. Set `API_URL` to the backend's URL. It's read when the app is **built**, so redeploy after changing it.

## Known limitations (and what would come next)

- No email verification or password reset, which would need an email service.
- Sessions can't be revoked before they expire, because JWTs are stateless. A token version on the user, or short-lived tokens with refresh tokens, would fix that.
- No rate limiting on login.
- Followers/following lists show the newest 100.
- Search uses `ILIKE`. PostgreSQL full-text search would scale better.
