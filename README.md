# Barkada League

A season leaderboard for a small group of friends who play the same 1v1 game
and want their standings, win percentage, and streaks worked out automatically
from recorded matches instead of by hand.

**Live site:** https://barkada-league.vercel.app
**API:** https://barkada-league-api.onrender.com/api/health
**Demo video:** https://drive.google.com/file/d/1ukJjqWWlt9BiXDnd4RGam--3MfiwoI9Q/view?usp=sharing

> The backend is on Render's free tier, which spins down after inactivity. The
> first request after a quiet period can take 50+ seconds while it wakes back
> up — that's expected, not a bug.

![Leaderboard screenshot](https://github.com/Technix19/barkadaleauge/blob/main/assets/Barkada_League_Design_System.pdf)

## What it does

- Record a match: pick two different players, enter both scores (they can't
  tie), and a date. The server works out the winner from the scores — the
  client is never trusted to say who won.
- Browse the leaderboard: rank, wins, losses, win %, and current streak, all
  calculated fresh from the match history, not stored anywhere.
- Browse match history, newest first, with edit and delete.
- Look up one player's season stats, recent matches, and head-to-head record
  against another player.

## Built with

React and Vite on the front end, Express and PostgreSQL on the back end. The
client is on **Vercel**, the API on **Render**, and the database on
**Supabase** (used only as a Postgres host — no Supabase Auth, Storage, or
client SDK is involved; the frontend only ever talks to the Express API).

Render, not a serverless platform, because the API depends on a process that
stays alive between requests: the rate limiter in
`server/src/utils/rateLimit.js` keeps its counts in an in-memory `Map`, and
the `pg` connection pool in `server/src/db.js` is built to be reused across
requests rather than recreated per call.

## Running it yourself

**Prerequisites:** Node.js 18+, and a PostgreSQL database — a free Supabase
project works, or any Postgres you have a connection string for, since the
backend uses plain `pg` with nothing Supabase-specific.

```bash
# 1. install both the frontend and backend dependencies
npm install
npm run server:install

# 2. set up the environment files (see Environment variables below)
cp .env.example .env                # root — VITE_API_URL
cp server/.env.example server/.env  # server — PORT, DATABASE_URL

# 3. create the schema and seed data, run against your Postgres database
#    (Supabase SQL Editor, psql, or any client) in this order:
#    database/schema.sql, then database/seed.sql
#    — this creates 6 players, 1 season, and 15 matches

# 4. run the backend (terminal 1) and frontend (terminal 2)
npm run dev:server   # http://localhost:3001
npm run dev          # http://localhost:5173
```

Check the API on its own before blaming the frontend:

```bash
curl http://localhost:3001/api/health
# {"status":"ok","database":"connected"} when the server and DB are both reachable
```

## Environment variables

None of these are committed. `.env.example` in both the root and `server/`
lists them with placeholder values.

| Name           | Where               | What it is                                                                                    |
| -------------- | ------------------- | --------------------------------------------------------------------------------------------- |
| `DATABASE_URL` | server              | PostgreSQL connection string. Contains a password — never commit it                           |
| `PORT`         | server              | local dev only. Render sets this itself; don't set it in the Render dashboard                 |
| `VITE_API_URL` | root, at build time | the API's public URL, no trailing slash. Compiled into the built JS and public — not a secret |

## Deploying

**Client, to Vercel.** Import the repo in the Vercel dashboard, Vite preset
(auto-detected from `vite.config.js`), default build command and output
directory. Add `VITE_API_URL` pointed at the deployed API before the first
deploy.

**API, to Render.** Create a web service from the same repo. Build command
`cd server && npm install`, start command `cd server && npm start`. Add
`DATABASE_URL` in the Environment tab — Render supplies `PORT` itself, so
don't set it.

**Database.** Already hosted on Supabase; no separate deploy step. Run
`database/schema.sql` then `database/seed.sql` once against it, the same as
local setup.

## Project structure

```
Barkada_League/
├── src/                    # React frontend (Vite)
│   ├── pages/              # one file per route
│   ├── components/         # atoms/molecules/organisms
│   ├── services/api.js     # all fetch() calls to the Express API live here
│   ├── utils/               # client-side validation + formatting helpers
│   └── styles/
├── server/                 # Express backend
│   └── src/
│       ├── app.js          # express app, middleware, route mounting
│       ├── server.js       # starts the server
│       ├── db.js           # shared pg Pool
│       ├── routes/         # players.js, seasons.js, matches.js, leaderboard.js
│       ├── utils/          # validation + shared leaderboard/streak math
│       └── tests/          # node:test unit tests
├── database/
│   ├── schema.sql          # table definitions + constraints + indexes
│   └── seed.sql            # sample data (6 players, 1 season, 15 matches)
├── docs/screenshots/        # screenshots of the finished app
├── project/                 # weekly report, security checklist, live links
├── journal/                  # weekly reflection
└── AI-USAGE.md
```

## API endpoints

All routes are prefixed with `/api`.

| Method | Route                                       | What it does                                                                                |
| ------ | ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| GET    | `/api/health`                               | Checks the server + database connection                                                     |
| GET    | `/api/players`                              | List all players                                                                            |
| GET    | `/api/players/:id`                          | One player                                                                                  |
| GET    | `/api/players/:id/stats?seasonId=`          | One player's derived stats (wins, losses, win %, streak, longest streak, rank) for a season |
| GET    | `/api/players/:id/matches?seasonId=`        | One player's matches, newest first                                                          |
| GET    | `/api/players/:id/vs/:opponentId?seasonId=` | Head-to-head matches and win totals between two players                                     |
| POST   | `/api/players`                              | Create a player                                                                             |
| GET    | `/api/seasons`                              | List all seasons                                                                            |
| GET    | `/api/seasons/active`                       | The active season (404 if none)                                                             |
| GET    | `/api/matches?seasonId=`                    | List matches for a season, newest first                                                     |
| GET    | `/api/matches/:id`                          | One match                                                                                   |
| POST   | `/api/matches`                              | Create a match (server derives the winner from scores)                                      |
| PATCH  | `/api/matches/:id`                          | Update a match (winner is recalculated)                                                     |
| DELETE | `/api/matches/:id`                          | Delete a match                                                                              |
| GET    | `/api/leaderboard?seasonId=`                | Full season standings, ranked, with streaks                                                 |

A few validation rules worth knowing: scores can't be negative, can't be
equal, both players have to exist and be different from each other, and the
request is not allowed to send `winnerId` — the server always calculates that
itself and rejects the request if you try to send it. `/api` is also rate
limited to 100 requests per IP every 15 minutes.

## Screenshots

[Design System](https://github.com/Technix19/barkadaleauge/blob/main/assets/Barkada_League_Design_System.pdf)

## Architecture

The React client (Vercel) talks only to the Express API (Render) over HTTPS —
it never touches Supabase directly. The API is the sole authority on who won
a match and on every derived number (wins, losses, win %, rank, streak): none
of those are stored, they're computed from the raw `matches` table on every
request, through `server/src/utils/leagueStats.js`, so the leaderboard and a
player's own profile can never disagree with each other.

## What I would do next

- Restrict CORS to the deployed frontend's origin instead of `cors()` with no
  config — fine for a course project, not for anything public long-term.
- Replace the in-memory rate limiter with something that survives a restart
  and doesn't grow unbounded, if this ever saw real traffic.
- Add a UI for managing players and seasons — right now that only happens
  through `seed.sql` or directly against the database, since it wasn't part
  of this project's scope.

See `project/REPORT.md` and `project/SECURITY-CHECKLIST.md` for the full,
checked-against-the-code account of what's built, what's missing, and why.

## Author

Apostol, Lance Jezreel B.
Computer Science — CS402

## AI use

[![Made with AI](https://img.shields.io/badge/Made_with-AI_assistance-blue)](AI-USAGE.md)

This project was built with use of **Claude Code** (Anthropic) as a
pair-programming assistant — it wrote most of the implementation code across
the whole stack (schema, Express routes, React components and styling, and
debugging), working from a spec and constraints I
wrote and reviewing each phase before moving on. Later work — several routes,
two rewritten utility modules, the unit tests, and the longest-win-streak
feature — was written by me.

See **[AI-USAGE.md](AI-USAGE.md)** for the full account: what I asked for and
what came back on each piece of work, cases where the AI got something wrong
and what I did instead, and which parts of the project are my own.
