# Week 2 Security Checklist

Note: I didn't have an exact checklist template from the professor to
fill in row-by-row, so this covers the standard categories that make
sense for a small full-stack project like this one (secrets, injection,
input validation, error handling, transport security, dependencies). If
the actual assignment gives specific row wording, the answers below
should map onto it pretty directly.

Every answer here is based on actually looking at the code and git
history, not just what "should" be true.

## Secrets and credentials

**Is the database connection string kept out of the codebase?**
Yes. `DATABASE_URL` and the Supabase password only ever live in
`server/.env`, which is listed in `.gitignore`. I checked the full git
history (`git log -p --all`), not just the current files, and confirmed
`server/.env` and the root `.env` were never committed at any point.

**Were any real secrets ever accidentally committed?**
No, but it was close. While setting up the backend I twice pasted the
real Supabase connection string into `server/.env.example` (the
tracked template file) instead of `server/.env` (the gitignored real
one). Both times I caught it and rewrote the example file with a
placeholder before staging/committing anything. Verified this is true by
searching the entire git history for the password and for the Supabase
project ref — zero matches anywhere.

**Does `.env.example` contain placeholder values only?**
Yes. Both `.env.example` files have obviously fake placeholder values
(`http://localhost:3001` and
`postgresql://YOUR_SUPABASE_CONNECTION_STRING`), no real credentials.

**Is the frontend free of any database credentials or Supabase secret
keys?**
Yes. The frontend only knows `VITE_API_URL`, which is just the Express
server's address, not a secret. Grepped the frontend source and the
built production bundle for `DATABASE_URL`, the actual database
password, and the word "supabase" — no matches in either.

## SQL injection / database safety

**Are all SQL queries parameterized?**
Yes. Every query in `server/src/routes/*.js` uses `$1`, `$2`, etc.
placeholders through the `pg` library instead of building query strings
by hand. Checked with a grep for any query that concatenates
`req.body`/`req.params`/`req.query` directly into a SQL string — found
none.

**Are database-level constraints used as a backup to application
validation (defense in depth)?**
Yes. `database/schema.sql` has `CHECK` constraints for non-negative
scores, no tied scores, no playing against yourself, and the winner
having to be one of the two players — on top of the same checks already
happening in the Express validation code. So even if the application
validation had a bug, the database itself won't accept bad data.

## Input validation

**Does the API validate input types strictly (not just loosely coercing
strings to numbers)?**
Yes. `server/src/utils/validateMatch.js` checks `typeof value ===
'number'` for scores and player/season IDs — a JSON body with
`"player1Score": "21"` (a string) gets rejected with a 400, not silently
converted.

**Can a client send a trusted `winnerId` and have it accepted?**
No — and this one was checked carefully since it matters a lot for this
project. `POST /api/matches` and `PATCH /api/matches/:id` both reject
the request with a 400 if `winnerId` shows up anywhere in the body. The
server always calculates the winner itself from the scores.

**Are route parameters (like `:id`) validated before being used?**
Yes. Every `:id` param is checked to be a positive integer
(`utils/validation.js`) before it's used in a query. Non-numeric,
negative, zero, or decimal IDs return a 400 instead of reaching the
database.

## Error handling / information disclosure

**Do error responses avoid leaking stack traces or raw database
errors?**
Yes. The Express error handler in `app.js` logs the actual error to the
server console but only ever sends the client a generic
`{"error": "Internal server error"}` on unexpected failures. Expected
errors (like "Player not found" or "Scores cannot be tied") get a plain,
specific message with no internal details attached.

## Transport / connection security

**Is the connection to the database encrypted?**
Yes. `server/src/db.js` sets `ssl: { rejectUnauthorized: false }` on the
`pg` Pool, which encrypts the connection to Supabase.

**Is the database certificate actually verified (not just encrypted)?**
No. `rejectUnauthorized: false` means the connection is encrypted but
doesn't verify the server's TLS certificate, so it's not fully protected
against a man-in-the-middle in theory. This is a common shortcut for
Supabase pooler connections and isn't a real practical risk on a
managed connection like this, but it's not the strictest possible
setting, so I'm not marking it a clean "yes."

## CORS

**Is the API's CORS policy restricted to known origins?**
No. `app.use(cors())` in `app.js` is called with no configuration, which
allows requests from any origin. That's fine for local development on a
course project, but it's not something I'd want to ship as-is if this
were a real production API.

## Authentication / authorization

**Does the app have user accounts, login, or authentication?**
N/A. This was an explicit non-goal of the project from the start (see
`BARKADA_LEAGUE_PROJECT_SPEC.md`) — it's a small private league tracker
for one group of friends, not a multi-user system. There's nothing to
authenticate against.

## Dependencies

**Are there any known-risky or unnecessary dependencies?**
Yes/No mix, explained: the dependency list is intentionally small —
`express`, `pg`, `cors`, `dotenv` on the backend, `react`,
`react-router-dom` on the frontend. No ORM, no auth library, no
Supabase SDK on the frontend (checked `package.json` directly to
confirm `@supabase/supabase-js` isn't installed anywhere). I did not run
a vulnerability scanner (like `npm audit`) as part of this checklist, so
I can't say for certain there are zero known CVEs in the dependency
tree — that's a real gap, not something I'm claiming is clean.

## Rate limiting / abuse protection

**Is there any rate limiting or request throttling on the API?**
Yes. I wrote a small rate limiter myself in `server/src/utils/rateLimit.js`
(no extra package) and it's registered in `server/src/app.js` on the `/api`
path. Each IP gets 100 requests per 15-minute window; the 101st returns a `429`
with a JSON error. `/api/health` is registered before the limiter, so it is not
limited. The counts live in memory, so they reset when the server restarts, and
the limiter doesn't clean up entries for IPs that stop calling. I tested it by
sending 101 requests to an unknown `/api` path: 100 returned `404` and the last
one returned `429`.

## Git hygiene

**Is `node_modules` excluded from git?**
Yes. Confirmed with `git ls-files | grep node_modules` — no results.

**Is the build output (`dist/`) excluded from git?**
Yes. Confirmed with `git ls-files | grep ^dist/` — no results.
