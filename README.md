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
- `PUT /api/rosters/:rosterId/capacity-modifiers` is retired and returns 409; capacity item bonuses are derived from the leader's inventory.
- `POST /api/rosters/:rosterId/members`
- `PATCH` and `DELETE /api/members/:memberId`
- `POST /api/members/:memberId/promote` promotes a Henchman only when its type is marked eligible for Lad's Got Talent.
- `POST /api/members/:memberId/advance-purchases` buys an initial campaign Hero advancement with either `{ "stat": "A" }` or `{ "skillId": "..." }`, using the campaign's configured prices and limits.
- `DELETE /api/members/:memberId/advances/:advanceId` undoes an eligible advancement; purchased awards refund their recorded price and reverse their purchased XP.
- `GET /api/members/:memberId/equipment` returns the warrior's permitted equipment options and owned inventory.
- `PUT /api/members/:memberId/mutations` replaces a Freebuild warrior's ordered mutation selections with `{ "mutationIds": ["..."] }`, charging or refunding the repriced difference. Campaign selections are fixed at recruitment.
- `POST /api/members/:memberId/equipment` buys an item with `{ "equipmentOptionId": "...", "quantity": 1, "modelIndex": -1 }`. A model index of `-1` applies an item to the whole Henchman group; otherwise it targets one model when the source permits individual group gear.
- `DELETE /api/members/:memberId/equipment` removes inventory rows with `{ "inventoryItemIds": ["..."] }` atomically. During creation it refunds recorded cost; after creation it pays half the listed base price, rounded down per copy. Free starter gear, bound items and permanent poisoned weapons cannot be sold.
- `DELETE /api/members/:memberId/equipment/:inventoryItemId` applies the same refund/resale rules to one inventory row; shared Henchman gear must be sold together for every model.

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
Halfling Scouts add one warrior slot. A Halfling Cookbook adds one slot only
while the current leader carries it in their inventory; stash copies, copies
on other warriors, and multiple cookbooks do not grant extra slots.
The Cookbook is unavailable to Undead and Carnival of Chaos. Hired Swords do
not count toward the model cap. The API enforces Hero and total-member limits
on recruitment, group resizing, and promotion. The audit migration updates
existing warbands without deleting warriors from rosters above a corrected cap.

The standard leader type leads whenever present (for example, Mercenary
Captain). Otherwise the Hero with the highest current Leadership leads; ties
use recruitment time, then warrior ID, not roster display order. Only Heroes
can lead. The current leader automatically displays the **Leader** ability in
their skills and printed roster: nearby members within 6 inches may use the
leader's Leadership for Leadership tests. It costs no XP or gold and cannot be
forgotten or purchased; it disappears when another Hero becomes leader.
Leadership and cookbook capacity are recalculated after recruitment, changes
to Leadership, transfers, and deaths. Losing the bonus does not delete existing
warriors or block returning the item; an over-capacity warning is shown and
further recruitment remains blocked until the roster fits its current limit.
Old manually selected cookbook bonuses no longer apply.
If the leader dies carrying the cookbook, it is lost with their other carried
items. The successor receives the Leader ability, not the dead leader's
equipment; a replacement cookbook must be purchased and assigned to the new
leader to restore the slot.

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

Equipment options and warrior-specific list permissions are source-backed in `backend/equipment-catalog.json`. Items marked first-free are added to inventory automatically for existing warriors and at recruitment, one per Henchman model; additional copies are charged. Creation refunds return recorded purchase cost; later sales pay half the listed base price. Free starter gear cannot be sold. Purchases and sales update the roster treasury transactionally, and group purchases charge per model. Henchman groups share equipment unless their fact sheet explicitly permits individual gear. Existing freeform equipment notes remain available for non-purchasable or campaign-record details.

Pistols, duelling pistols, and warplock pistols can be bought singly or as a
named **Brace** option wherever the single pistol is permitted. A brace costs
twice the list's single-pistol price and is recorded as one inventory unit
containing two pistols, counting as one missile weapon toward the two-weapon
carrying limit. The existing quantity field counts singles or braces according
to the selected item; buying two singles does not automatically convert them.

**Export PDF / Print** includes a writable experience track for each warrior
that can gain XP: 90 boxes for Heroes and 14 for Henchmen/Hired Swords.
Current XP is marked with an X; double-bordered boxes indicate advancement
thresholds. Henchman tracks record shared XP per model, not group-total XP.
The boxes use visible borders and text, so background printing is not required.

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
lists when **Allow skill purchases** is checked. Uncheck that option during
campaign creation to allow only purchased characteristics. The API stores
this as `advancePurchaseRules.skillsEnabled` and rejects skill purchases when
false; older campaigns without the setting retain their previous behavior.
Earned skill advances are unaffected. Both stat and skill purchases raise experience to the next Hero
advancement threshold and create an already-resolved advancement, so no extra
advance is awarded for the purchased XP. Paid costs and prior XP are recorded.
During initial creation, undoing the latest purchase (or forgetting its skill)
refunds its recorded price and restores its stat/XP effect. Later awards must be
undone first; refunds are blocked after subsequent XP gains or roster creation.
Existing campaigns keep purchases disabled.

Warrior-type lookups include `maxCount`, `maxCountReferenceTypes`, and `maxCountMultiplier`. A null `maxCount` with no reference types means no per-type cap was defined in the catalog; fixed and dependent caps are enforced by both the API and roster selectors.

## Warband stash and trading

Each warband has a separate **Warband Stash**. Gear in the stash is safe when
a warrior dies; carried gear is lost. Use **Record death** during post-battle
injuries rather than removing a hire: deaths never refund hire fees or equipment.
For Henchmen, choose the dead model; only that model and its gear are removed.

The **Trading shop** includes every **Core, 1a and 1b** row from the
[New Mordheimer Trading Post](https://mordheimer.net/docs/trading-post):
178 source rows represented by 203 purchasable entries, including brace,
weapon-material and Dark Elf Blade variants. Filter by grade and by close
combat, missile, blackpowder, armour, miscellaneous equipment, or animal bestiary.
Prices and rarity retain faction/type exceptions, and each entry links to its
source rules. Buying places equipment in the
stash and deducts the exact recorded cost. A brace is one item containing two
pistols. Recruitment-list prices remain unchanged during initial roster setup;
later campaign equipment and Freebuild equipment after the first battle must
be bought through the shop.

- **Freebuild:** edit **Battles fought** in Warband Information; counts persist
  and appear in print/PDF exports. At **0**, only warrior-specific recruitment
  shops are open, with full-cost refunds for paid creation equipment.
  At **1 or more**, those shops close, including for new recruits: buy available
  Mordheim shop items into the stash without rarity searches, then transfer
  them to eligible warriors. Roll variable
  prices in the app or enter physical D6 results.
  Alternatively, **Add as combat spoils** adds the selected item and quantity
  to the stash at zero cost, without price rolls or rarity searches. Normal
  item availability and warrior equipment restrictions still apply.
  Rituals and permanent weapon upgrades must use their special paid actions,
  not combat spoils. Creation-only items remain locked after the first battle.
  Manual combat spoils are unavailable in campaigns; scenario and administrator
  reward awards are planned for a later update.
- **Campaigns:** search at post-battle step 6; purchase at steps 6–8. Each Hero
  gets one search per battle, whether successful or not. Heroes marked out of
  action cannot search. A 2D6 total meeting the item's rarity permits one copy;
  Streetwise adds +2. Search results and price quotes persist across reloads,
  and requesting a quote again does not reroll an unpaid offer.
- **Selling:** after the first Freebuild battle, or at campaign purchasing
  steps 6–8, sell stash quantities or carried equipment. Each copy pays half
  its listed Mordheim shop **base cost**, rounded down to whole GC; variable
  price dice and amounts paid are ignored. This includes ordinary combat
  spoils. Campaign price overrides/custom prices apply; items not listed in
  the Mordheim shop use their warband-list price. Free starter equipment,
  bound Familiars and permanently poisoned weapons cannot be sold.
  Shared Henchman gear must be sold from every model together; sell or return
  barding before selling its mount. Sales remain possible when buying is
  unaffordable and update treasury and inventory atomically.
- Mark Heroes **out of action** during battle or injuries. Availability resets
  with the next battle's record, not by reloading the page.
- Transfer gear between stash and members in Freebuild, initial setup,
  pre-battle, or post-battle **Reallocate equipment** (step 9). Transfers do not
  charge/refund GC. Battle/injury transfers are blocked to prevent rescuing gear
  from a dead warrior.
- Fixed-price shop items display their price and can be bought directly;
  only variable-price items show price-roll controls.
  Skink Heroes can be selected as the buyer for their Common, fixed-price
  Black Lotus and Dark Venom exceptions. Campaign price overrides take
  precedence, and variable prices support up to ten D6.
- **Special purchases:** Dark Elf Blades are new swords or daggers with the
  upgrade included in the price. A Familiar summoning attempt charges its
  quoted cost even if the rarity roll fails, uses that Hero's campaign search,
  and excludes Prayer users. A summoned Familiar is bound to its caster,
  including while stashed, and is lost if that caster dies.
  Poisoned Weapon permanently upgrades a carried weapon; identical-gear
  Henchmen upgrade one matching weapon per model and pay for every copy.
  Such weapons cannot be traded, sold or returned to stash.
  The Standard of Nagarythe is available only before the first battle and,
  in campaigns, during initial setup.
- Swivel Guns are limited to one per warband, including recruitment-origin
  gear. Peg Legs and Bota Bags are limited to one per model. Barding requires
  the recipient's carried warhorse; return barding before returning the mount.
  Giant Spiders and Giant Wolves cannot coexist in a warband, and species
  and skill restrictions still apply to their riders.
- Each stash item has a recipient selector containing only eligible warriors.
  Choose a recipient and quantity there; Henchman quantities are per model
  unless their equipment rules permit selecting an individual model.
  Warriors return items using **Return to stash** in their character inventory.
  Shared Henchman equipment returns the same quantity from every model.
- **Assigned equipment** groups matching item names and categories into total
  quantities (for example, **Dagger x 7 - Weapon**), followed by the names of
  the carrying warriors or Henchman groups. Model numbers are hidden in this
  summary; individual-model inventory tracking and transfer controls are unchanged.
- Recipients must meet weapon/armour list, warband, warrior-type, Hero-only and
  skill restrictions. Weapons Training and Weapons Expert permit their
  respective weapon classes but do not bypass hard restrictions. Identical-gear
  Henchmen take one copy per model; individual gear requires an explicit list
  permission. A transferred free starter item is not regenerated.
- Campaign creation supports item disables, fixed or variable price overrides,
  rarity overrides, and custom items. Custom items require a name, description,
  cost and rarity (Common or a target), with optional category, allowed/excluded
  warbands/types, Hero-only and required-skill restrictions. They belong only to
  that campaign; existing campaigns use the default shop.

Miscellaneous effects are shown for players to apply; this feature does not
automatically resolve all item effects or injuries. The leader-carried
Halfling Cookbook capacity bonus is automated. Existing Tome of Magic
consumption works with tomes bought from the shop and assigned to an eligible
Hero. Shop gear cannot use the old full-cost recruitment refund action.
Voluntarily reducing a Henchman group's size returns the removed models'
shop gear to the stash rather than refunding its purchase price. After
creation, removed models' recruitment-list gear also returns to the stash
without a full-cost equipment refund.
Permanently poisoned weapons are lost instead; bound-item metadata is
preserved during inventory transfers and promotions.
Animal entries are inventory, not automatically recruited/fielded models.
Entries for unsupported warbands remain listed but unavailable rather than
silently granting faction-specific equipment to other warbands.
Haggle discounts are not yet automated, and the generic Mercenaries roster
does not distinguish Marienburg for its rare-item search bonus.
For the same reason, faction-specific Rapier and Middenheim Wolfcloak
eligibility is not inferred from a generic Mercenaries roster.

### Trading API

- `GET /api/shop`: base shop catalog for campaign customization.
- `GET /api/rosters/:rosterId/trading`: stash, carried gear, shop, Hero status,
  current-battle searches and action permissions.
- `POST .../trading/search`: `{heroId, itemId, mode, dice?, quoteId?}`.
  Familiar summoning requires the caster's `quoteId`; the response includes
  the paid attempt result, including unsuccessful attempts.
- `POST .../trading/quote`: `{itemId, mode, dice?, buyerId?}` returns a persisted
  quote scoped to that buyer (required for Familiar rituals).
- `POST .../trading/purchase`: `{quoteId, quantity, searchId?}`.
- `POST .../trading/spoils`: `{itemId, quantity?}`, Freebuild only.
- `POST .../trading/sell`: `{source: "stash", inventoryIds: [id], quantity}`
  sells copies from one stash stack; `{source: "member", inventoryIds: [id, ...]}`
  sells complete carried rows. Returns refreshed trading data and `saleAmount`.
- `POST /api/rosters` and `PATCH /api/rosters/:rosterId` accept
  `battlesFought` (0–2147483647) for Freebuild only. Campaign counts are managed
  by the battle sequence. Quotes from an earlier battle count expire.
- `POST .../trading/upgrade`: `{itemId, inventoryId, quoteId}` permanently
  upgrades a carried weapon, charging every model of an identical-gear group.
- `POST .../trading/transfer`: `{direction, inventoryId, memberId, quantity,
  modelIndex}`. Direction is `to_member` or `to_stash`; model index `-1`
  equips all models with `quantity` copies each.
- `POST .../trading/hero-status`: `{heroId, outOfAction}`.
- `POST /api/rosters/:rosterId/casualties`: `{memberId, modelIndex}`.
- `POST /api/campaigns` accepts `tradingRules: {overrides, customItems}`.

Dice mode is `simulated` or `manual`; manual input must have exactly the
required number of integers from 1 to 6. Treasury changes, stock
movement, quote consumption and rare-offer consumption remain transactional.

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

- **Freebuild**: choose "Freebuild (no campaign)" in the campaign menu to build warbands outside any campaign (you enter the starting Gold Crowns when creating; default 500). Gold Crowns and Wyrdstone are editable, saved balances; Wyrdstone starts at zero. Both are included in print/PDF exports. Warbands can be shared by code like any other warband.
- **Campaign currency**: Gold Crowns and Wyrdstone are tracked in the Warband Stash
  header (editable in Freebuild, read-only in Campaign mode)
  and cannot be edited directly through the API. Starting Gold Crowns still come
  from the campaign's limit, and hiring, purchases, and equipment sales continue
  to adjust gold normally. Wyrdstone sales and campaign earnings tracking are
  planned, not implemented yet.
- Editable Freebuild currency balances have bordered input boxes and an
  automatic-save hint; these editing cues are not shown on read-only warbands.
- The editable warband name uses a larger title font beside the limits and roster
  statistics in the warband information row, stacking on narrow screens. Stash currency
  balances align with the stash title, with the Freebuild save hint beneath.
  A crown and **Leader** badge identify the current leader in the Hero list and
  their details header; the badge follows automatic leadership succession.
- PDF/print exports include **Total Fielded** (including every Henchman model
  and Hired Sword) and **Rout Test At**, rounded up to 25% out of action.
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
