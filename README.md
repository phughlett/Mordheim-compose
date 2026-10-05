# Mordheim Roster App


## Getting the code

This repo pulls in the backend and frontend as submodules:

```
git clone --recurse-submodules git@github.com:phughlett/Mordheim-compose.git
```

The project contains a React Router frontend, an Express API using Knex, and PostgreSQL. Docker Compose builds and runs all three services.

## Run with Docker Compose

Start Docker Engine, then run from the repository root:

```sh
docker compose up --build
```

- Frontend: http://localhost:5173
- API health: http://localhost:4000/api/health
- PostgreSQL: localhost:5432

Database migrations run when the backend container starts. Compose keeps database files in the `mordheim-pgdata` named volume. To change the local ports or database credentials, copy `.env.example` to `.env` and edit the values.

API requests are logged as JSON lines in the backend container. View them with `docker compose logs -f backend`.

## Production deployment

The shared production stack for Dawnbreaker and Mordheim is maintained separately
in [hughlett-web-deploy](https://github.com/phughlett/hughlett-web-deploy).
That repository contains one Compose file, one reverse proxy, and a script to
clone or update all four application repositories. This repository remains the
standalone Mordheim development stack.

## API

- `GET /api/health` checks the API and database connection.
- `GET /api/warbands` lists catalogued warbands.
- `GET /api/warbands/:warbandId/warrior-types?category=Hero` lists selectable Hero types, including any `maxCount` and dependent type limits; the `category` parameter also accepts `Henchman` and `Hired Sword`. Add `&includeUnavailable=true` to review prohibited and unverified combinations with their rule source.
- `GET /api/rosters` and `POST /api/rosters`
- `GET`, `PATCH`, and `DELETE /api/rosters/:rosterId`
- `PUT /api/rosters/:rosterId/capacity-modifiers` assigns the roster's capacity items using `{ "modifierIds": [...] }`.
- `POST /api/rosters/:rosterId/members`
- `PATCH` and `DELETE /api/members/:memberId`
- `POST /api/members/:memberId/promote` promotes a Henchman only when its type is marked eligible for Lad's Got Talent.
- `POST /api/members/:memberId/advance-purchases` buys an initial campaign Hero advancement with either `{ "stat": "A" }` or `{ "skillId": "..." }`, using the campaign's configured prices and limits.
- `DELETE /api/members/:memberId/advances/:advanceId` undoes an eligible advancement; purchased awards refund their recorded price and reverse their purchased XP.
- `GET /api/members/:memberId/equipment` returns the warrior's permitted equipment options and owned inventory.
- `PUT /api/members/:memberId/mutations` replaces a Freebuild warrior's ordered mutation selections with `{ "mutationIds": ["..."] }`, charging or refunding the repriced difference. Campaign selections are fixed at recruitment.
- `POST /api/members/:memberId/equipment` buys an item with `{ "equipmentOptionId": "...", "quantity": 1, "modelIndex": -1 }`. A model index of `-1` applies an item to the whole Henchman group; otherwise it targets one model when the source permits individual group gear.
- `DELETE /api/members/:memberId/equipment` removes the listed paid inventory rows with `{ "inventoryItemIds": ["..."] }` and refunds their recorded total cost atomically. Free equipment cannot be sold.
- `DELETE /api/members/:memberId/equipment/:inventoryItemId` removes one paid inventory row and refunds its recorded cost.

The backend migrations create roster tables plus `warbands`, `warrior_types`, and `warband_warrior_types` lookup tables. Warrior types also record promotion eligibility as `eligible`, `ineligible`, `unverified`, or `not_applicable`. The workbook's Hero dropdown whitelist is used for Henchmen; explicit rulebook exclusions override it. Only eligible Henchmen get a promotion action, and the API enforces the same rule. Set `VITE_API_BASE_URL` in `.env` if the API is exposed at a different URL; this value is baked into the frontend image at build time.

Warband limits are stored per warband: 6 Heroes by default, with base model
maximums audited against the local warband PDFs in
[`backend/warband-capacity.json`](backend/warband-capacity.json):

| Base maximum | Warbands |
| --- | --- |
| 12 | Bretonnian, Dark Elves, Dwarf Treasure Hunters, Shadow Warrior, Witch Hunters |
| 20 | Lizardmen, Night Goblins, Orc, Skaven |
| 15 | Amazons, Averlander Mercenaries, Beastmen Raiders, Carnival of Chaos, Cult of the Possessed, Kislevite, Mercenaries, Norse, Ostlanders, Pirate, Pit Fighter, Sisters of Sigmar, Undead, Marauders of Chaos, Battle Monks of Cathay |

The Marauders reference gives Hung tribes a base limit of 12 rather than 15;
tribe selection is not currently supported, so the catalog uses the general
Marauders limit. Referenced encampment increases are not automatically applied.
Halfling Scouts and an assigned Halfling Cookbook each add one warrior slot;
the Cookbook is unavailable to Undead and Carnival of Chaos. Hired Swords do
not count toward the model cap. The API enforces Hero and total-member limits
on recruitment, group resizing, and promotion. The audit migration updates
existing warbands without deleting warriors from rosters above a corrected cap.

Per-type caps are stored on each warband/type association. The sourced rules include fixed limits and dependent limits such as Cave Squigs per Night Goblin and Pirate Swabbies per Crew member. Recruitment and type changes enforce these limits in the API as well as disabling full types in the frontend.

Selected Hired Sword types are fixed, like hired Heroes and Henchmen. The roster
table and details panel show the type instead of a selector. Remove and re-hire
a Hired Sword to choose a different type; names and notes remain editable.

Henchmen are stored as groups of 1–5 models. Each group has one shared stat and experience record; roster capacity and type caps count every model in the group. Lad's Got Talent promotion splits one model into a Hero and leaves the remaining group intact.

Group counts in roster rows are read-only. In member details, **Hire one**
charges the model's hire fee and checks available GC, capacity, and type limits.
**Remove one** refunds the model's hire fee like removing the group and discards
its individual inventory. Campaign hiring/removal permissions apply; the last
model is removed using the roster's Remove action.

Equipment options and warrior-specific list permissions are source-backed in `backend/equipment-catalog.json`. Items marked first-free are added to inventory automatically for existing warriors and at recruitment, one per Henchman model; additional copies are charged. Paid equipment sales refund the recorded purchase cost, while free starter gear cannot be sold. Purchases and sales update the roster treasury transactionally, and group purchases charge per model. Henchman groups share equipment unless their fact sheet explicitly permits individual gear. Existing freeform equipment notes remain available for non-purchasable or campaign-record details.

Cult of the Possessed mutations are tracked as special equipment in warrior
inventory. Campaign Mutants must choose at least one mutation at recruitment;
Possessed may choose mutations optionally. The first selection costs its listed
price and every additional selection costs double, charged together with the
hire fee. Campaign mutations are permanent and cannot be bought later, sold,
or transferred. Freebuild permits later editing, charging or refunding the
difference after repricing the selection. A Mutant without a mutation is flagged
in details; existing campaign warriors are not silently altered or charged.
Mutation effects are displayed, not automatically applied to base characteristics.
Rules and prices are sourced from `2Warbands.pdf`, Cult of the Possessed,
Mutations (page 14).

Campaign creation includes optional paid stat/skill advancements for Heroes
during initial roster creation only (not Freebuild, Henchmen, or Hired Swords).
The creator can customize first/additional prices and a purchased-increase cap
per characteristic; blank caps mean unlimited purchases within racial stat
limits and the 90 XP Hero track. Defaults are M/WS/BS/Ld 15 GC, I 10 GC,
S/A 25 then 35 GC, T 30 then 45 GC, W 20 then 30 GC, and skills 40 GC.
Each purchased stat allows one purchased skill from the Hero's normal skill
lists. Both stat and skill purchases raise experience to the next Hero
advancement threshold and create an already-resolved advancement, so no extra
advance is awarded for the purchased XP. Paid costs and prior XP are recorded.
During initial creation, undoing the latest purchase (or forgetting its skill)
refunds its recorded price and restores its stat/XP effect. Later awards must be
undone first; refunds are blocked after subsequent XP gains or roster creation.
Existing campaigns keep purchases disabled.

Warrior-type lookups include `maxCount`, `maxCountReferenceTypes`, and `maxCountMultiplier`. A null `maxCount` with no reference types means no per-type cap was defined in the catalog; fixed and dependent caps are enforced by both the API and roster selectors.

## Correction submissions

**Report a correction** is available on the sign-in screen and in the roster
sidebar, including for anonymous visitors. A short summary and an explanation
of what is incorrect are required; an HTTP/HTTPS rule-reference URL is optional.
Visitors must consent to publishing the text/link on GitHub. No account details,
roster information, or email addresses are automatically attached.

The backend creates a real issue in `phughlett/Mordheim-compose` and adds it to
[Project 5](https://github.com/users/phughlett/projects/5). Configure the
`CORRECTIONS_*` values in [.env.example](.env.example) before
use. Missing credentials leave the form explicitly unavailable.

- GitHub: a classic PAT needs `public_repo` for public-repository issue writes
  (or `repo` for private repositories), plus `project` for the user-owned Project.
  The repository must have Issues enabled, and the token's user must have
  repository/Project access. Do not paste tokens into chat or commit them.
- Set `CORRECTIONS_RATE_LIMIT_SECRET` to a stable random secret of at least 32
  characters, for example generated with `openssl rand -hex 32`. It is used to
  hash IP-based rate-limit keys; do not commit it. Recreate the backend container
  after changing settings.
- PostgreSQL-backed limits allow five attempts per IP (IPv6 /64) per hour and
  100 total per hour. A hidden spam-trap field is also validated. No CAPTCHA or
  Cloudflare account is required; these measures reduce spam but do not prevent
  all automated abuse. Reference links are stored in the issue, never fetched
  by the API.
- `GET /api/corrections/config` provides availability and destination links.
  `POST /api/corrections` accepts `id` (UUID v4), `title` (5–120 characters),
  `explanation` (20–10,000 characters), optional `referenceUrl`, `consent: true`,
  and the empty spam-trap `website` field.
- A submission receipt persists across API restarts. Browser session storage
  preserves a pending submission and its ID across reopening/reloading the form.
  If issue creation succeeds but Project linking fails, the response explicitly
  reports partial completion and retry links the existing issue. Unknown issue
  creation outcomes are blocked from automatic recreation: search GitHub for the
  displayed submission ID and reconcile the `correction_submissions` record
  before retrying. The local receipt does not store submitted text or raw IPs.

The production deployment repository includes the environment wiring and trusts
one reverse-proxy hop for IP limiting. Local Compose leaves proxy trust disabled;
never trust forwarded IP headers when the API is directly exposed.
Automated tests mock GitHub and create no live issues.

## Sharing and testing

- **Freebuild**: choose "Freebuild (no campaign)" in the campaign menu to build warbands outside any campaign (you enter the starting GC when creating; default 500, treasury stays editable). They can be shared by code like any other warband.
- Freebuild can record mature warriors' experience at any time. Heroes are capped
  at 90 XP and Henchmen at 14 XP in both Freebuild and campaigns, in the table,
  details panel, and API. Experience
  must be a non-negative whole number within database storage limits. Warrior
  types that cannot gain XP, starting-XP floors, and recorded-advance/promotion
  floors still apply. Campaign XP changes are permitted only during post-battle
  step 2, "Allocate experience".
- Campaign members can view each other's warbands read-only. Only the owner can edit.
- An owner can press **Share warband** to get a clickable link and use **Copy link**
  to send it to a friend. Opening the link prompts for sign-in or registration if
  needed, then opens the warband read-only without granting campaign membership.
  Codes still work under "Shared warband code". **Stop sharing** revokes the link,
  code, and redeemed sharing access; campaign members retain campaign viewing.
- Backend integration tests (two or three users against the compose database, test data cleaned up afterwards):
  `docker compose exec backend npm test`
