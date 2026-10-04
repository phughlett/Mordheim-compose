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
- `GET /api/members/:memberId/equipment` returns the warrior's permitted equipment options and owned inventory.
- `POST /api/members/:memberId/equipment` buys an item with `{ "equipmentOptionId": "...", "quantity": 1, "modelIndex": -1 }`. A model index of `-1` applies an item to the whole Henchman group; otherwise it targets one model when the source permits individual group gear.
- `DELETE /api/members/:memberId/equipment` removes the listed paid inventory rows with `{ "inventoryItemIds": ["..."] }` and refunds their recorded total cost atomically. Free equipment cannot be sold.
- `DELETE /api/members/:memberId/equipment/:inventoryItemId` removes one paid inventory row and refunds its recorded cost.

The backend migrations create roster tables plus `warbands`, `warrior_types`, and `warband_warrior_types` lookup tables. Warrior types also record promotion eligibility as `eligible`, `ineligible`, `unverified`, or `not_applicable`. The workbook's Hero dropdown whitelist is used for Henchmen; explicit rulebook exclusions override it. Only eligible Henchmen get a promotion action, and the API enforces the same rule. Set `VITE_API_BASE_URL` in `.env` if the API is exposed at a different URL; this value is baked into the frontend image at build time.

Warband limits are stored per warband: 6 Heroes and 15 warriors by default, with cited overrides such as Dwarf Treasure Hunters/Shadow Warriors (12) and Skaven/Orcs/Night Goblins (20). Halfling Scouts and an assigned Halfling Cookbook each add one warrior slot; the Cookbook is unavailable to Undead and Carnival of Chaos. The API enforces Hero and total-member limits on both recruitment and promotion.

Per-type caps are stored on each warband/type association. The sourced rules include fixed limits and dependent limits such as Cave Squigs per Night Goblin and Pirate Swabbies per Crew member. Recruitment and type changes enforce these limits in the API as well as disabling full types in the frontend.

Selected Hired Sword types are fixed, like hired Heroes and Henchmen. The roster
table and details panel show the type instead of a selector. Remove and re-hire
a Hired Sword to choose a different type; names and notes remain editable.

Henchmen are stored as groups of 1–5 models. Each group has one shared stat and experience record; roster capacity and type caps count every model in the group. Lad's Got Talent promotion splits one model into a Hero and leaves the remaining group intact.

Equipment options and warrior-specific list permissions are source-backed in `backend/equipment-catalog.json`. Items marked first-free are added to inventory automatically for existing warriors and at recruitment, one per Henchman model; additional copies are charged. Paid equipment sales refund the recorded purchase cost, while free starter gear cannot be sold. Purchases and sales update the roster treasury transactionally, and group purchases charge per model. Henchman groups share equipment unless their fact sheet explicitly permits individual gear. Existing freeform equipment notes remain available for non-purchasable or campaign-record details.

Warrior-type lookups include `maxCount`, `maxCountReferenceTypes`, and `maxCountMultiplier`. A null `maxCount` with no reference types means no per-type cap was defined in the catalog; fixed and dependent caps are enforced by both the API and roster selectors.

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
- An owner can press **Share warband** to get a code; any signed-in user can enter it under "Shared warband code" for read-only access. **Stop sharing** revokes it.
- Backend integration tests (two or three users against the compose database, test data cleaned up afterwards):
  `docker compose exec backend npm test`
