# AI Usage

Barkada League was built with heavy use of **Claude Code** (Anthropic) as a
pair-programming assistant. It wrote most of the implementation code, working
from a spec and a set of constraints I wrote and from phase-by-phase direction
and review. I'm stating that accurately rather than downplaying it.

Repository: https://github.com/Technix19/Barkada_League

---

## 1. How I used AI

### Entry 1 — Database schema, seed data, and the Express skeleton

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** A PostgreSQL schema for a 1v1 league — players, seasons,
matches — with the rules enforced in the database itself (no tied scores, nobody
playing themselves, the winner has to be one of the two players), seed data to
work against, and a minimal Express app with a health endpoint that connects
through `pg`.

**What came back:** `database/schema.sql` with three tables, foreign keys set to
`ON DELETE RESTRICT`, four `CHECK` constraints, and indexes on the match columns
I'd be filtering by. `database/seed.sql` with 6 players, one active season, and
15 matches. `server/src/db.js` with a shared `Pool`, and `server/src/app.js` with
`GET /api/health` running `SELECT 1`.

**What I kept and changed:** Kept all of it. One thing I noticed and deliberately
left alone for now: the AI set `ssl: { rejectUnauthorized: false }` on the pool.
That encrypts the connection but skips certificate verification. I kept it
because it's what the Supabase pooler connection wanted, but I wrote it up
honestly as a known weakness in `project/SECURITY-CHECKLIST.md` rather than
pretending it's a clean setup.

**Commit:** https://github.com/Technix19/Barkada_League/commit/12d3ee3b9d350e28768da99ed62f06878f77d1fa

---

### Entry 2 — The whole React frontend, built against mock data first

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** All five screens — Leaderboard, Match History, Record
Match, Edit Match, Player Profile — with React Router, built against a fake
in-memory data service so the UI could be finished before the real API existed.

**What came back:** A component tree split into atoms/molecules/organisms, the
five page components, `src/services/mockApi.js` and `src/data/mockData.js`, the
responsive CSS, and client-side form validation.

**What I kept and changed:** Kept the structure. The mock-first approach was on
purpose — I wanted the shape of the data settled before writing the API. It did
backfire in one specific way that I only found later, which I've written up in
section 2 below.

**Commit:** https://github.com/Technix19/Barkada_League/commit/8c03a08bf8dad5337f8b63f9903c36232c4406c8

---

### Entry 3 — Dark theme and a restrained motion pass

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** Replace the original light/green look with a dark, near-black
and neon-blue identity. Keep it restrained — no glow everywhere, no esports
look — and don't break the responsive layout or accessibility.

**What came back:** A rewritten CSS token set in `src/styles/variables.css`, a
`TrophyIcon` component, a subtle highlight treatment for the rank-1 row, and a
`prefers-reduced-motion` block that switches the animations off.

**What I kept and changed:** Kept it, but reviewing the screenshots caught two
bugs it had introduced — the nav showing two items as active at once, and the
winner's name losing its colour on match rows. Both were fixed before I committed.

**Commit:** https://github.com/Technix19/Barkada_League/commit/b4041ec6dd87c348cca9957121a1be708eafdf39

---

### Entry 4 — The read-only Express API over Postgres

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** `GET` endpoints for players, seasons, and matches, reading
from the real database, returning camelCase JSON, with every query parameterised
and route IDs validated before they reach SQL.

**What came back:** `routes/players.js`, `routes/seasons.js`, `routes/matches.js`
and `utils/validation.js` with a `parsePositiveInt` helper. Every query uses
`$1`-style placeholders. Date columns are cast with `::text` in the SQL.

**What I kept and changed:** Kept it. The `::text` cast was the AI's idea and I
asked why — the `pg` driver turns a `DATE` column into a JavaScript `Date` at
local midnight, which shifts the day depending on the server's timezone. Casting
to text in SQL means it comes back as a plain `"YYYY-MM-DD"` string and the
problem can't happen. I accepted it once I understood the reason.

**Commit:** https://github.com/Technix19/Barkada_League/commit/ccdee02519f1445af191cfbfb97019b3c4a03916

---

### Entry 5 — Match write API, with the server deciding the winner

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** `POST`, `PATCH` and `DELETE` for matches. Hard requirement:
the server works out the winner from the scores, and the API refuses a request
that tries to send `winnerId` itself.

**What came back:** The three write routes plus `utils/validateMatch.js`, which
checks types strictly (`typeof value === 'number'`, so a score sent as the string
`"21"` is rejected rather than quietly converted), validates that a date is a real
calendar date, and rejects the whole request with a 400 if `winnerId` appears
anywhere in the body.

**What I kept and changed:** Kept it. The strict number check turned out to matter
more than I expected, because HTML form inputs hand you strings by default — so
the frontend has to convert them before sending, and the backend catches it if it
doesn't.

**Commit:** https://github.com/Technix19/Barkada_League/commit/f64c524ed7431f606fd75ceec0a487bce7fa741c

---

### Entry 6 — Leaderboard calculated from match rows

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** `GET /api/leaderboard?seasonId=` that works out matches
played, wins, losses, win percentage, rank and current streak from the `matches`
table — nothing stored — and still lists players who haven't played yet.

**What came back:** `routes/leaderboard.js` using a `LEFT JOIN` with a `FILTER`
aggregate so zero-match players still appear, the streak worked out in JavaScript
from matches ordered newest-first, and a four-level sort (wins, then win
percentage, then matches played, then name).

**What I kept and changed:** Kept the logic, but the way it was organised caused a
problem as soon as I added the player stats endpoint — see section 2, case 2.

**Commit:** https://github.com/Technix19/Barkada_League/commit/e26b770da73d7c324d6fadf03926556013ade748

---

### Entry 7 — Player stats endpoint and a shared stats module

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** `GET /api/players/:id/stats?seasonId=` for the Player
Profile page, and — specifically — make it impossible for it to disagree with the
leaderboard.

**What came back:** The ranking, streak and win-percentage logic pulled out of the
route and into `server/src/utils/leagueStats.js`, with both the leaderboard route
and the player stats route calling the same `getSeasonStandings()` function.
`leaderboard.js` dropped from 119 lines to 26.

**What I kept and changed:** Kept it. The reason I asked for it this way is that
two separate implementations of "what is this player's rank" can drift apart, and
then the leaderboard and the profile page show different numbers for the same
player. Sharing one function makes that impossible rather than just unlikely.

**Commit:** https://github.com/Technix19/Barkada_League/commit/f105008d7e190930c2d2ebd6eac4c82c13c893cb

---

### Entry 8 — Connecting the React frontend to the real API

**Date:** 2026-09-01
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** Replace the mock service with real `fetch()` calls, and
change as little of the page/component code as possible.

**What came back:** `src/services/api.js` with a shared `request()` helper that
sets JSON headers, returns early on a `204` (so `DELETE` doesn't try to parse an
empty body), and throws an error carrying the backend's own message so the form
can display "Scores cannot be tied" instead of something generic. It also added
small adapter functions that reshape the API's nested response into the flatter
shape the existing components already expected, so the components didn't need
rewriting. The mock files were deleted.

**What I kept and changed:** Kept the adapter approach. It also fixed a crash it
had caused itself — see section 2, case 1.

**Commit:** https://github.com/Technix19/Barkada_League/commit/cbaafdfeca449264909aa87371d56d933992ea1a

---

### Entry 9 — Documentation screenshots

**Date:** 2026-09-03
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** A consistent screenshot set of the finished app — desktop
and 375px mobile, plus close-ups of specific component states — taken against the
real running app and the real database, not mockups.

**What came back:** 22 PNGs in `docs/screenshots/`, captured with headless
Chromium. It used reduced-motion emulation while capturing so the animated
elements sat still and the frames came out clean rather than caught mid-animation.

**What I kept and changed:** Kept all 22. It opened the delete-confirmation dialog
for that screenshot and then cancelled rather than confirming, so no real match
was deleted from the database.

**Commit:** https://github.com/Technix19/Barkada_League/commit/7a242996f03a8c4e284cc1ba9fab28b1491f0988

---

### Entry 10 — Documentation audit and writing

**Date:** 2026-09-27
**Tool:** Claude Code (Claude Sonnet)

**What I asked for:** Audit the repository against the Week 2 requirements, tell
me what's actually missing, and then write the documentation — with an explicit
instruction not to invent anything that the repo and git history don't support.

**What came back:** It found that `README.md` was still the default Vite template,
and that `project/` and `journal/` didn't exist at all. It then wrote the README,
`project/REPORT.md`, `project/SECURITY-CHECKLIST.md` and `journal/week-2.md`.

**What I kept and changed:** Kept them. Worth noting: when I described my week's
work it pushed back on two claims — that responsive design was part of that work
(git shows it was built earlier, in `8c03a08`) and that the documentation was
already done (it wasn't). It also pointed out that the commit history doesn't
support a two-calendar-week timeline and refused to write one.

**Commit:** https://github.com/Technix19/Barkada_League/commit/acf284bc961f1c63400c709c3771f4d09f0f816b

---

### Entry 11 — Active season route (written by me, reviewed with AI)

**Date:** 2026-10-04
**Tool:** Claude Code (Claude Opus / Sonnet)

**What I asked for:** Guidance on how to add `GET /api/seasons/active`, which
returns the season flagged as active, or a 404 if there isn't one.

**What came back:** The AI did not write the route. It gave me the requirements,
where to place it in `server/src/routes/seasons.js`, and the patterns to follow
from the existing routes, then reviewed each version I pasted back.

**What I kept and changed:** I wrote the route myself. The review caught a typo
in the JSON key (`isActiwve`), a path declared as `/` instead of `/active`
(which would have been shadowed by the list route), an empty `catch` block, and
inconsistent error strings. I fixed each one myself.

**Commit:** https://github.com/Technix19/Barkada_League/commit/bd74f7c22d5869e090da755cbbdb37cefbb85a76

---

### Entry 12 — In-memory rate limiter (written by me, reviewed with AI)

**Date:** 2026-10-04
**Tool:** Claude Code (Claude Opus / Sonnet)

**What I asked for:** Guidance on writing a rate limiter with no extra package,
limiting each IP to 100 requests per 15 minutes and returning 429 after that.

**What came back:** The AI did not write the limiter. It explained the Express
middleware shape, what to store per client, and how to wire it into `app.js`,
then reviewed my implementation.

**What I kept and changed:** I wrote `server/src/utils/rateLimit.js` myself. The
review pointed out that the first-request and window-reset branches do the same
thing, and that my comments restated the code. I kept the logic but noted both
points. The limitation that the in-memory `Map` never shrinks is documented in
the README and the security checklist. I tested it against the live database by restarting the
server (to reset the counter) and sending 101 requests to `/api/players`. The
result was 100 `200` responses and 1 `429`.

**Commit:** https://github.com/Technix19/Barkada_League/commit/98235e35cc09bc6553b1dc04c6bcf94eb6eceb79

---

## 2. Where the AI got it wrong

### Case 1 — It wrote two versions of the same thing that disagreed, and the bug hid behind mock data

**What it gave me:** In `LeaderboardRow.jsx` and `LeaderboardTable.jsx` it wrote:

```js
const streakVariant = entry.currentStreak.type === 'W' ? 'win' : ...
```

**What was wrong with it:** The same AI wrote both sides of this contract, and
they didn't match. The mock service returned `{ type: null, count: 0 }` for a
player with no matches:

```js
// src/utils/leagueStats.js (mock version, commit 8c03a08)
if (outcomesNewestFirst.length === 0) {
  return { type: null, count: 0 };
}
```

…but the real backend returns a bare `null`:

```js
// server/src/utils/leagueStats.js
if (!outcomesNewestFirst || outcomesNewestFirst.length === 0) {
  return null;
}
```

So `entry.currentStreak.type` reads `.type` off `null` and the whole leaderboard
page crashes — but only for a player who hasn't played a match yet. Every player
in the mock data had matches, so this sat in the code from 2026-09-01 without
ever showing up. It only became reachable the moment real API data arrived.

**What I did instead:** Changed both to optional chaining so a missing streak
falls through to the neutral badge:

```js
const streakVariant = entry.currentStreak?.type === 'W' ? 'win' : ...
```

**Commits:** the bug is visible in
https://github.com/Technix19/Barkada_League/commit/8c03a08bf8dad5337f8b63f9903c36232c4406c8
and the fix in
https://github.com/Technix19/Barkada_League/commit/cbaafdfeca449264909aa87371d56d933992ea1a

**What I took from it:** Mock data is friendlier than real data. Mine never had a
player with zero matches, so it never produced the one case that broke the page.

---

### Case 2 — It put the stats logic somewhere that would have forced me to duplicate it

**What it gave me:** The first version of `routes/leaderboard.js` (119 lines) had
everything inline in the route handler — the aggregate SQL, the streak
calculation, the win-percentage maths, and the four-level sort.

**What was wrong with it:** It works fine as one endpoint. The problem shows up
with the second one. When I added `GET /api/players/:id/stats` for the Player
Profile page, that same ranking and streak logic would have had to be written a
second time. Two copies of "what is this player's rank" can drift apart, and then
the leaderboard says a player is 3rd while their own profile page says 4th — with
no obvious bug, just two implementations that have quietly diverged.

**What I did instead:** Asked for it to be pulled out into
`server/src/utils/leagueStats.js`, with both routes calling the same
`getSeasonStandings()`. The player stats endpoint now looks itself up in the same
ranked list the leaderboard returns, so the two can't disagree — it's the same
computation, not a matching one. `leaderboard.js` went from 119 lines to 26.

**Commits:** original
https://github.com/Technix19/Barkada_League/commit/e26b770da73d7c324d6fadf03926556013ade748
→ refactor
https://github.com/Technix19/Barkada_League/commit/f105008d7e190930c2d2ebd6eac4c82c13c893cb

---

### Case 3 — A CSS change that silently broke an unrelated feature

**What it gave me:** During the dark theme pass it added an explicit colour to the
player-name link style:

```css
.leaderboard-player-link {
  font-weight: 700;
  text-decoration: none;
  color: var(--color-text); /* <- this line */
}
```

**What was wrong with it:** On a match row, the winning player's name is supposed
to be highlighted — that comes from a colour set on the parent element, which the
link inherits. Setting `color` directly on the link itself beats inheritance, so
every winner's name silently went back to plain white. The page still rendered
and nothing errored; the feature just stopped working. I only caught it by
actually looking at the screenshots and noticing the winners weren't highlighted
any more.

**What I did instead:** Removed the `color` line so the link inherits again, and
left a comment saying why, so it doesn't get "helpfully" added back:

```css
.leaderboard-player-link {
  font-weight: 700;
  text-decoration: none;
  /* No explicit color: inherits from the reset's `a { color: inherit }`
     so a winning player's name still shows in the success color. */
}
```

**Commit:** https://github.com/Technix19/Barkada_League/commit/b4041ec6dd87c348cca9957121a1be708eafdf39

**Honest caveat:** this bug and its fix are both inside that one commit, because I
committed at the end of the phase rather than mid-way — so unlike case 1 you
can't see a before/after diff for it in git. The CSS comment is the evidence
that's actually in the repo.

**What I took from it:** CSS bugs don't throw errors. This one would have shipped
if I'd only checked that the page loaded.

---

## 3. Who wrote what

### Code I wrote myself, reviewed with AI but not written by it.

The pieces below are the part of the Node/Express/Postgres application that I wrote myself. They include complete routes, validation and statistics utilities, middleware, and backend tests rather than only small edits or formatting changes. I listed the exact files and commits for each one so the work can be checked against the repository history.

### Self-written 1 — Rate limiter middleware

- **What I wrote:** `server/src/utils/rateLimit.js`, wired into `server/src/app.js`
  on `/api`. Commit: https://github.com/Technix19/Barkada_League/commit/98235e35cc09bc6553b1dc04c6bcf94eb6eceb79
- **Why it's built that way:** I made it a function that takes the time window and request limit as values so I can easily change them in app.js without editing the actual limiter logic. I used a Map to keep track of each IP address and how many requests it has made. Since this is only a small project running on one server, I thought using memory was enough and using a database just for the rate limiter would be unnecessary.

### Self-written 2 — Active season route

- **What I wrote:** `server/src/routes/seasons.js`, the `GET /active` handler.
  Commit: https://github.com/Technix19/Barkada_League/commit/bd74f7c22d5869e090da755cbbdb37cefbb85a76
- **Why it's built that way:** I made /active a separate route because it has a different purpose from the route that returns all seasons. If there is no active season, I return a 404 because the specific thing being requested does not exist. My first version used /, which would have conflicted with the existing route, so I changed it to /active.

### Self-written 3 — Unit tests for the stats and validation logic

- **What I wrote:** `server/tests/leagueStats.test.js` and
  `server/tests/validateMatch.test.js`, using Node's built-in `node:test` runner
  so no new dependency was needed. I added the `test` script to
  `server/package.json`.
  Commits: https://github.com/Technix19/Barkada_League/commit/b4fa954b032b83d48396dcdf865cdf1f0b3509b1 (tests) and https://github.com/Technix19/Barkada_League/commit/c3116965dfe87f55d7ea6aa769df73d35dd4b89e (script)
- **Why it's built that way:** I focused the tests on functions that can run by themselves without needing the database. This made the tests simpler and safer because they only check the logic and do not change any real data. I also used Node's built-in test tools so I did not have to install another testing library. After the tests were reviewed, I also improved some of the assertions so they checked the exact result more clearly.

### Self-written 4 — Player matches route

- **What I wrote:** the `GET /:id/matches` handler in `server/src/routes/players.js`.
  It returns a player's matches, newest first, with an optional `seasonId` filter.
  Commit: https://github.com/Technix19/Barkada_League/commit/982728fd3c324f489aa96204ecd658c7325ebd9c
- **Why it's built that way:** The route first checks if the player actually exists, so if the ID is invalid or missing in the database it returns a 404 instead of just showing an empty array. I used the same `$1` value to check both `player1_id` and `player2_id` since the player can be on either side of the match. The season condition is only added when a `seasonId` is provided. I also kept the `::text` cast for the date so it stays in `YYYY-MM-DD` format and does not get affected by timezone changes.
- **What I tested:** Against the running server. The matches count for player 1 matched
  the `/stats` count (6 and 6). Unknown player, bad player id, and bad season id all
  returned the expected 404 or 400.
- **Fixed afterwards:** An unknown `seasonId` returned an empty list, while `/stats`
  returned 404. I added the same season check to this route, and it's in
  https://github.com/Technix19/Barkada_League/commit/f2f16209d0b31e41e3c01964c016066459ae850e
  I found a bug in my first version of that check: it ran even without a `seasonId`,
  so every request returned 404. I caught it when I retested and fixed it.

### Self-written 5 — Head-to-head route

- **What I wrote:** the `GET /:id/vs/:opponentId` handler in `server/src/routes/players.js`.
  It returns the matches between two players, newest first, with an optional `seasonId`
  filter and a summary of each player's wins.
  Commit: https://github.com/Technix19/Barkada_League/commit/1edc29282ec8c214827ae3b43fb1b43085d86471
- **Why it's built that way:** The route checks both player IDs first before doing anything with the database. If both IDs are the same, it returns a 400 right away because a player should not be compared against themselves. It also checks that both players actually exist. The match query works even if either player appears as player 1 or player 2, and the win totals are based on `winner_id` so they match the actual recorded winner of each game.
- **What I tested:** Against the running server. `/1/vs/2` and `/2/vs/1` return the same
  two matches, the two win counts add up to the total, and the same-player, unknown
  player, and unknown season cases return the expected 400 and 404.

### Self-written 6 — Match validation rewrite

- **What I wrote:** I rewrote `server/src/utils/validateMatch.js` myself, working from
  the validation rules in my spec and the step-by-step rule list the AI gave me. I
  didn't copy the original file. The AI reviewed my version and confirmed it against
  the rules. Its `checkBodyShape`, `validateCompleteMatch`, and `deriveWinnerId`
  functions are the ones `matches.js` uses. The commit message says "without AI",
  which isn't accurate; the rewrite was done with AI guidance and review.
  Commit: https://github.com/Technix19/Barkada_League/commit/f2c0bc274b8b66be5451e989a65ef133f153ae68
- **Why it's built that way:** The validation rules are checked in a specific order, so the function stops as soon as it finds the first problem. For example, if the same player is selected and the scores are also tied, it will return the player error first. For the date, it checks whether the date stays the same after JavaScript converts it, which helps catch invalid dates like 2026-02-30.
- **What I tested:** `npm test` passes all 16 tests, and the AI checked the file against
  the rule list before I committed it. The original AI-written version is still described in
  Section 1 and Section 2.

### Self-written 7 — Leaderboard and streak logic rewrite

- **What I wrote:** I rewrote `server/src/utils/leagueStats.js` from the ranking and
  streak rules, not by copying the original. It holds `getSeasonStandings`,
  `calculateStreak`, `calculateWinPercentage`, and `seasonExists`, which the leaderboard
  and player stats routes both use.
  Commit: https://github.com/Technix19/Barkada_League/commit/3f51264975005d89dd5271d54c6cdcae4678e00c
- **Why it's built that way:** The ranking is based on wins first, followed by win percentage, then the number of matches played, and finally the player's name if there is still a tie. The streak is based on the player's latest results and counts how many wins or losses in a row they currently have. I used a `LEFT JOIN` so players are still included in the standings even if they have not played any matches yet.
- **What I tested:** `npm test` passes all 16 tests. I also compared the old and new
  `getSeasonStandings` output for season 1, and they matched for all 6 players, including
  ranks and streaks. I checked this with a temporary copy of the old file, which I then
  deleted.
- **A note on the commit message:** It says "not AI". That's not accurate. The AI gave
  me the rules and reviewed the rewrite, the same as with `validateMatch.js`.

### Self-written 8 — Match write handlers

- **What I wrote:** I rewrote the `POST /`, `PATCH /:id`, and `DELETE /:id` handlers in
  `server/src/routes/matches.js`, working from the match rules in my spec and the
  step-by-step guidance the AI gave me. The AI reviewed the rewrite.
  Commits: https://github.com/Technix19/Barkada_League/commit/03dd00de51928c5ae3e16265fe23ff9783fe5faa
  (POST) and https://github.com/Technix19/Barkada_League/commit/120a0c67b104a51bb2f0c5ca266123a245156c02 (PATCH and DELETE)
- **Why it's built that way:** The server always calculates the winner itself whenever a match is created or updated, so the user cannot manually set the winner in the request. For `PATCH`, it only changes the fields that were actually sent and keeps the rest of the existing match data. After that, it validates the full updated match before saving it, so sending something like `null` will cause a validation error instead of removing the value. Before saving any changes, it also checks that the season and both players really exist.
- **What I tested:** Against the running server, I sent invalid requests to all three
  handlers and got the expected 400 and 404 responses. I then ran one create, edit, and
  delete cycle: a 3-1 match was created with player 1 as winner, changing the score to 0-4
  made player 2 the winner under the same id, and the delete returned 204. The match count
  went back to 15.

### Self-written 9 — Create player route

- **What I wrote:** the `POST /` handler in `server/src/routes/players.js`, with its own
  body-shape check. It accepts `name`, an optional `nickname`, and an optional `joinDate`,
  and returns 201 with the new player. I wrote the rules first, then the code, and the AI
  reviewed them. I also exported `isValidDate` from `validateMatch.js` so both routes
  share one date rule.
  Commit: https://github.com/Technix19/Barkada_League/commit/bac94db3961e3c1bdf554a559830e3bd646bd869
- **Why it's built that way:** Before adding the player to the database, I check all the fields first to make sure the values are valid. The name and nickname are trimmed so extra spaces are removed, and if the nickname is left blank, I save it as `null`. I still allow two players to have the same name because each player has their own unique id. After creating the player, I fetch it again using the same query as `GET /api/players/:id` so the returned data uses the same format.
- **What I tested:** Against the running server, six invalid requests returned the expected
  400 messages and the player count stayed at 6. Then I created one test player, which
  returned 201 with the name trimmed and the blank nickname stored as `null`, and it showed up
  in `GET /api/players`. There is no player delete route, so I removed it with a direct SQL
  delete by id, and the count went back to 6.

### Self-written 10 — Longest win streak

- **What I wrote:** `calculateLongestWinStreak` in `server/src/utils/leagueStats.js`, its
  use in `getSeasonStandings`, the `longestWinStreak` field in `GET /api/players/:id/stats`
  in `server/src/routes/players.js`, and the unit tests in `server/tests/leagueStats.test.js`.
  Commit: https://github.com/Technix19/Barkada_League/commit/95e82db6c779865e651b5ab4f98767ffa82092c4
- **Why it's built that way:** I made longestWinStreak a separate function because it measures something different from the current streak. The current streak only checks what is happening right now, while the longest win streak looks through all of the player’s results and finds the highest number of wins in a row. I return null when a player has no matches, but 0 when they have played and never won, because those are two different cases. I also kept it out of the leaderboard sorting so the existing ranking rules would not change.
- **What I tested:** `npm test` passes all 21 tests, including a case where the longer
  streak comes second. The live leaderboard and `/stats` return the field, and the player
  count stayed at 6.

### AI-written piece I understand best

- File: src/styles/variables.css
- Commit: https://github.com/Technix19/Barkada_League/commit/b4041ec6dd87c348cca9957121a1be708eafdf39

The AI-written piece I understand best is src/styles/variables.css. This file contains the main design values used throughout the website, such as the colors, font sizes, spacing, border radius, shadows, and maximum page width. Instead of writing the same color or spacing value repeatedly in different CSS rules, they are stored as CSS variables inside :root.
For example, --color-primary contains the main blue color of the website, while --color-bg and --color-surface control the darker background colors. Other CSS files can then use something like var(--color-primary) instead of writing the hex color again. This makes the design more consistent because changing one variable can update that value everywhere it is being used.
The same idea is used for spacing and font sizes. Variables such as --space-2, --space-4, and --space-8 give the interface a consistent spacing system instead of using random values for every component. The radius and shadow variables work in a similar way for cards, buttons, and other elements.
One thing I would change is some of the naming. There are several variables for similar colors, such as --color-primary, --color-primary-bright, and --color-primary-soft. They make sense after reading the file, but I would probably add short comments explaining where each variation should normally be used. This would make it easier for me to choose the correct variable when adding a new component later.
