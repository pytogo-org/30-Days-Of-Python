# Africa Heritage API — 30‑Day Capstone

Design and build a public API that catalogs historic places, museums, landmarks, animals with unique identity, bridges, pyramids and skyscrapers across **Togo** first, then **Africa**.

Learners will create a production‑style REST API with user submissions, an admin review flow, media handling, search, pagination, and documentation.

---

## Product Vision

Preserve and showcase Africa’s cultural & natural heritage through a developer‑friendly API. Anyone can propose a new entry; only admins can approve, edit, or remove entries.

**Primary users**

* **Visitors/Consumers**: read public, approved entries
* **Contributors**: submit new entries and edits (pending review)
* **Admins**: moderate (approve/reject), curate, and manage content

**Example entries**: *Monument de l’Indépendance (Lomé), Place du 2 Février, Maisons Tamberma (Koutammakou), Pyramids (Giza), Éléphant d’Afrique, Colombe de la Paix (Lomé), iconic bridges, skyscrapers, museums…*

---

## Core Features (MVP)

1. **Public catalog** of approved entries with filtering & pagination
2. **Submission system** for anyone (auth optional) to propose entries
3. **Admin moderation**: approve/reject, soft delete, restore
4. **Media**: image URL capture (+ optional upload)
5. **Documentation**: OpenAPI/Swagger at `/docs`
6. **Auth**: JWT‑based with roles (`visitor`, `contributor`, `admin`)
7. **Audit trail**: created/updated timestamps, submitted\_by, reviewed\_by
8. **Search**: by name, country, category, tags

---

## Data Model (Core)

**Entity: `Item`** (single table with categories) – or split into multiple tables if you prefer.

* `id` (UUID)
* `slug` (unique, URL‑safe)
* `name` (string, required) — e.g., "Monument de l’Indépendance"
* `title` (string, optional if different from name)
* `category` (enum): `place | museum | structure | animal | monument | bridge | skyscraper | pyramid | site`
* `country` (ISO code + name) — e.g., `TG`, `Togo`
* `city` (string, optional)
* `location` (geo): `latitude`, `longitude` (optional)
* `year` (integer, optional) — year built/founded/discovered
* `event_date` (date, optional) — key historical date
* `era` (string, optional) — e.g., `Colonial`, `Post‑independence`
* `description` (text, required)
* `abstract` (text, 280–600 chars)
* `tags` (array of strings)
* `image_url` (string, URL)
* `image_credit` (string/URL)
* `official_url` (string, URL, optional)
* `references` (array of URLs)
* `status` (enum): `pending | approved | rejected | archived`
* `submitted_by` (user id or email)
* `reviewed_by` (user id, nullable)
* `created_at`, `updated_at` (timestamps)
* `deleted_at` (nullable; soft delete)

**Entity: `User`**

* `id` (UUID)
* `email` (unique)
* `password_hash`
* `role` (enum): `visitor | contributor | admin`
* `created_at`, `updated_at`

**Entity: `Revision`** (optional but recommended)

* `id`, `item_id`, `diff`, `submitted_by`, `status`, timestamps

**ASCII ERD**

```
User (1) ────< Item (many)
   └────< Revision (many) >──── Item (1)
```

---

## Roles & Permissions

* **Visitor**: `GET /items`, `GET /items/{id}` (approved only)
* **Contributor**: everything Visitor can do + `POST /submissions`, `PATCH /submissions/{id}` (own, while pending)
* **Admin**: all read + `POST /items/approve`, `PATCH /items/{id}`, `DELETE /items/{id}` (soft delete), manage users

**Moderation Rules**

* New submissions default to `pending`
* Admin can `approve` → item becomes public, `rejected` → stays hidden with reason
* Soft delete: mark `deleted_at` instead of hard delete

---

## API Endpoints (Proposed with FastAPIREST semantics)

### Public (no auth)

* `GET /health` → `{status: "ok"}`
* `GET /items` → list approved items with query params:

  * `q` (search name/title/abstract)
  * `category`, `country`, `city`, `tag`
  * `near=lat,lon&radius=km` (optional stretch goal)
  * `page`, `page_size`, `sort` (`name`, `-year`, `created_at`)
* `GET /items/{id_or_slug}` → detail (approved only)
* `GET /stats` → counts by country, category, recent additions

### Auth & Users

* `POST /auth/register` (email, password) → JWT
* `POST /auth/login` → JWT
* `GET /me` → current user & role

### Submissions (Contributor)

* `POST /submissions` → create pending item (same schema as item but `status=pending`)
* `GET /submissions/mine` → list my submissions
* `PATCH /submissions/{id}` → edit while `pending`

### Admin Moderation

* `GET /admin/submissions?status=pending` → queue
* `POST /admin/submissions/{id}/approve` → publish (moves/merges to `items`)
* `POST /admin/submissions/{id}/reject` → with `reason`
* `PATCH /admin/items/{id}` → edit approved item
* `DELETE /admin/items/{id}` → soft delete
* `POST /admin/items/{id}/restore` → undo soft delete

### Documentation

* `GET /docs` (Swagger UI)
* `GET /openapi.json`

---

## Validation & Business Rules

* `name` ≥ 3 chars; `abstract` 280–600 chars; `description` ≥ 200 chars
* `category` in enum (up to you to define)
* `image_url` must be valid, `http(s)` only
* If `country=TG`, `city` is recommended (warn)
* For animals: allow `scientific_name`, `iucn_status` (optional fields)
* Enforce **no duplicates** by `(name, city, country)` or similar unique index (mmmh depends on category but you get the idea yeah?)
* Slug auto‑generated from name; ensure uniqueness with suffix

---

## Tech Stack (recommended)

**Option A — FastAPI** (simple & modern)

* FastAPI + Pydantic + SQLModel (or SQLAlchemy) + Alembic
* PostgreSQL (prod) / SQLite (dev)
* Auth with JWT (PyJWT or fastapi‑users)
* Image upload (stretch): local storage or S3 compatible (MinIO)

**Option B — Django REST Framework** (batteries included)

* DRF + Postgres + django‑filters + djangorestframework‑simplejwt
* Admin site for moderation

Use whichever you want, from the 30days, we've leanrt FastAPI, do so!

---

## Request/Response Examples

### Create a submission (Contributor)

`POST /submissions`

```json
{
  "name": "Monument de l’Indépendance",
  "title": "Monument de l’Indépendance de Lomé",
  "category": "monument",
  "country": "TG",
  "city": "Lomé",
  "year": 1962,
  "abstract": "Symbole de l’indépendance du Togo, situé à Lomé, espace civique majeur.",
  "description": "Érigé après l’indépendance, le monument… (≥200 chars)",
  "image_url": "https://…/monument-independance.jpg",
  "tags": ["indépendance", "histoire", "Lomé"],
  "references": ["https://…", "https://…"]
}
```

### Public list

`GET /items?country=TG&category=monument&page=1&page_size=20`

```json
{
  "count": 42,
  "page": 1,
  "page_size": 20,
  "results": [ { "id": "…", "name": "Monument de l’Indépendance", "slug": "monument-independance-lome", "country": "TG", "city": "Lomé", "category": "monument", "image_url": "…" } ]
}
```

### Approve (Admin)

`POST /admin/submissions/UUID/approve`

```json
{ "approved": true, "item_id": "UUID", "reviewed_by": "admin_id" }
```

---

## Testing Checklist

* Unit tests for schemas & validators
* Integration tests for endpoints
* Auth tests (visitor vs contributor vs admin)
* Duplicate prevention
* Pagination & filters
* Soft delete & restore
* OpenAPI schema generated & valid

---

## Deployment

* `.env` config (DB URL, JWT secret, CORS origins)
* Dockerfile + docker‑compose (API + DB) if applicable
* Seed script with example entries (Togo first)
* CI: run tests & lint on PRs
* Hosting options: Vercel, Netlify, MongoDB for database

---

## Search & Filters (MVP)

* Full‑text search on `name`, `title`, `abstract`
* Facets: `country`, `category`, `tags`, `era`
* Sort: `name`, `-year`, `-created_at`

**Stretch**: Geo queries (`near=lat,lon&radius`), language filter (`lang=fr|en`), multi‑image gallery.

---

## Stretch Goals

* Image upload + thumbnailing
* Rate limiting for public endpoints
* Webhooks or email notifications on approval/rejection
* CSV/JSON export for researchers
* Versioned edits (`Revision` table + diff)
* Admin dashboard (charts: items by country/category)
* i18n content (FR/EN fields) if so desired

---

## Suggested Milestones (10-20 days project window)
For PyCon Togo submission, you can adjust the timeline as needed and submit a working MVP (1,2,4,5,7,8) by the deadline 22nd 23h59 GMT.

**1**: Project scaffolding, DB models, migrations

**2**: Public GET endpoints + pagination + filters

**3**: Auth (JWT) + roles

**4**: Submissions flow (POST/PATCH)

**5**: Admin moderation (approve/reject/delete/restore)

**6**: Validation rules + duplicate prevention + tests

**7**: Swagger docs, examples, seed data (Togo set)

**8**: Dockerize(if so desired), deploy, final polish

---

## Seed Data Ideas (Togo → Africa)

* **Monument de l’Indépendance** (Lomé, TG) — monument
* **Colombe de la Paix** (Lomé, TG) — monument/site
* **Place du 2 Février** (Lomé, TG) — skycraper
* **Maisons Tamberma – Koutammakou** (Kara, TG) — site/UNESCO
* **Musée National du Togo** (Lomé, TG) — museum
* **Éléphant d’Afrique** — animal (add `scientific_name`, `iucn_status`)
* **Monument de l'Amazone** (ex: BJ, Cotonou, monument)
* **Pyramides** (Gizeh, EG) — pyramid
* **Skyscrapers** (ex: The Leonardo, Johannesburg) — skyscraper

---

## Evaluation Rubric (100 pts)

* **API design & documentation** (20)
* **Auth & permissions** (15)
* **Submission & moderation flow** (15)
* **Data validation & integrity** (15)
* **Testing coverage** (15)
* **Search, filters, pagination** (10)
* **DevOps: Docker + deploy + .env** (10) (just deployment is fine without docker, no worries)

Bonus: stretch goals (+10)

---

## Starter Commands (FastAPI example)

* Create app: `uvicorn app.main:app --reload`
* Migrations: `alembic init` → `alembic revision --autogenerate -m "init"` → `alembic upgrade head`
* Run tests: `pytest -q`

**Suggested structure**
Do not mind the structure, you can change it as you want, but here is a suggested structure for your FastAPI app:
```
app/
  main.py
  api/
    routes_public.py
    routes_submissions.py
    routes_admin.py
  core/
    security.py  settings.py
  models/
    item.py  user.py  revision.py
  schemas/
    item.py  user.py
  services/
    moderation.py  search.py
  db.py
tests/
```

---

## Contributor Checklist (what to turn in)

* API base URL + deployed link (if hosted)
* OpenAPI JSON & Postman/Insomnia collection
* DB schema (diagram or SQL)
* 10+ approved entries (min 5 from Togo)
* Admin test credentials
* README with setup, env vars, and usage examples

---

## Resumé

Construire une API publique pour inventorier les sites et monuments du Togo puis d’Afrique. Tout le monde peut **soumettre** une fiche (image, URL, nom, titre, description, année/date, résumé, tags…). **Seuls les admins** peuvent **approuver**, **rejeter**, **supprimer/restaurer**. Inclure l’authentification par rôles, la pagination, la recherche et la documentation Swagger.

Bonne chance !!!
```
