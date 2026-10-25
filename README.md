# Art Studio

A small full-stack website for an art studio. Customers can **buy one-of-a-kind paintings**, **book a wall-painting date**, and **request custom pottery**. The studio (admin) adds paintings and confirms/declines bookings and pottery requests.

- **Backend:** Node.js (plain `http` module, no framework)
- **Database:** SQLite via Node's built-in `node:sqlite`
- **Frontend:** plain HTML + JavaScript (no build step)
- **Dependencies:** none. There is nothing to `npm install`.

See [DESIGN.md](DESIGN.md) for how login/roles work, how double-booking is prevented, and what is and isn't finished.

## Run it on a clean machine

**Requirement:** Node.js **22.13 or newer** (`node -v`). Built and tested on 22.23.

```bash
git clone <this-repo-url> art-studio
cd art-studio

cp .env.example .env
# edit .env: set SESSION_SECRET and ADMIN_PASSWORD (see comments inside)

npm run seed     # creates the database, loads seed data, creates the studio login
npm start        # http://localhost:3000
```

Open <http://localhost:3000>.

| Role | How to get it |
|---|---|
| **Customer** | Click *New customer? Sign up* on the login page. |
| **Studio (admin)** | Log in with `ADMIN_EMAIL` / `ADMIN_PASSWORD` from your `.env`. (There is no studio sign-up, by design.) |

`npm run seed` is safe to run again: it won't duplicate paintings, pottery items, glazes or the admin user.

### Run the tests

```bash
npm test
```

12 API tests run against an in-memory database. They cover 401/403/404 on every endpoint, all validation rules, the status rules, and concurrent buying/booking.

## Using the official seed data

The brief's `seedData.zip` was not available when this was built, so `seed/seedData.json` contains **placeholder values** (a wall rate of 150 per sq ft, five pottery items, five glazes, three sample paintings). **Replace them with the real values** before relying on the numbers:

```json
{
  "wallRatePerSqFt": 150,
  "potteryItems": ["Mug", "Bowl"],
  "glazes": ["Celadon", "Matte White"],
  "paintings": [{ "title": "…", "artist": "…", "description": "…", "price": 12000 }]
}
```

Then `rm -rf data && npm run seed` (or point `SEED_FILE` in `.env` at another file). If the official data has a different shape (e.g. CSV, or a rate that varies), only `src/seed.js` needs to change; the app reads everything from the database.

## Project layout

```
server.js          HTTP server: static files, session check, 401/403 gate, JSON handling
src/routes.js      Every API endpoint and its business rules (RBAC, validation, state changes)
src/db.js          SQLite schema (incl. the unique indexes that stop double-booking) + transaction helper
src/auth.js        Password hashing (scrypt) and signed session tokens (HMAC-SHA256)
src/util.js        Validation helpers, date helpers (IST), HttpError
src/seed.js        Loads seed data and creates the studio account
public/            index.html + app.js (the whole UI)
test/api.test.js   API tests
```

## API summary

All endpoints except sign-up/login/logout need a login (else **401**). Wrong role gives **403**; someone else's record gives **404**; a rule conflict (sold, date taken, too late, wrong state) gives **409**; bad input gives **400**.

| Endpoint | Customer | Admin |
|---|---|---|
| `POST /api/auth/signup`, `/login`, `/logout` · `GET /api/me`, `/api/meta` | ✔ | ✔ (no sign-up) |
| `GET /api/paintings` | ✔ | ✔ |
| `POST /api/paintings` · `PATCH /api/paintings/:id` (only while available) | ✘ 403 | ✔ |
| `POST /api/orders` · `POST /api/orders/:id/cancel` | ✔ own | ✘ 403 |
| `POST /api/bookings` · `POST /api/bookings/:id/cancel` | ✔ own | ✘ 403 |
| `POST /api/bookings/:id/confirm` · `/decline` | ✘ 403 | ✔ |
| `POST /api/pottery` · `PATCH /api/pottery/:id` · `POST /api/pottery/:id/cancel` | ✔ own | ✘ 403 |
| `POST /api/pottery/:id/accept` · `/decline` | ✘ 403 | ✔ |
| `GET /api/orders`, `/bookings`, `/pottery` (and `/:id`) | only own | all |
| `GET /api/bookings/taken-dates` | ✔ | ✔ |
