# HANDOFF — live session-transition state

> **What this is:** the single rolling document any agent reads before starting work. Update at the end of every working session — overwrite stale state in place, don't accumulate timestamped sections. Keep under one page.

---

## TL;DR

**PR [#13](https://github.com/harrylynchh/airtight-container/pull/13) open** (`phase-struct/structural-batch` → `main`): Storage & Handling outbound flow + Pickup Numbers domain + editable quotas + Quote line→container mapping + 50+ smaller polish edits. **Tests: server 226/226, client 50/50, tsc clean both sides.**

**Cutover is mid-run.** Operator was driving `~/airtight-cutover/cutover.sh` against prod tonight; the verifier tripped on a self-inflicted false positive (script's own footer contained the literal word "FAIL"). Both the footer and the orchestrator's grep are fixed. **Re-run `~/airtight-cutover/cutover.sh` end-to-end** to push the new migrations to prod. Steps 1–4 are idempotent against the current local snapshot.

After the cutover finishes, merge PR #13 to ship the code that matches the new schema. (Schema and code can be reversed in order — the migrations are additive — but co-deploying is cleaner.)

---

## Cutover quick reference

Live at `~/airtight-cutover/`:
- `cutover.sh` — staged orchestrator with `y/N` confirm gates per step. Pass `--yes` to skip gates. Tunables (SSH key, paths, creds) at the top.
- `verify-migrations.sql` — schema-only sentinels for migrations 0022/0023 + live-fire trigger probe. Smoke-tested 13/13 PASS on local.
- Older `containers_2.0_POST-AUDIT_*.psql` archives are still here — keep them another week.

Tonight's run reached step 5 successfully (migrations applied, verifier clean). New artifacts in `~/airtight-cutover/`: `prod-20260529-223921.psql` + `local-pre-20260529-223921.psql`. The post-migration local dump will be `local-post-…` once the rerun completes.

Postgres on EC2 can't write into `/home/ubuntu/`; the script now stages dumps/restores through `/tmp/cutover/` and moves artifacts back as ubuntu. Don't change those staging hops without remembering why.

---

## What's new in this PR

| Surface | What changed |
| --- | --- |
| `/pickup-numbers` | New admin page mirroring Releases. Active/Filled tabs, drawer per pickup with assigned-box table, **inline quota edit** (blocks if new < used). |
| `/outbound` | Top-level Sales / Storage & Handling tabs. S&H flow is **single-box**: pick → assign pickup + free-text damage (default `Out good`) → print 80mm thermal receipt. Deep-link via `?sh_inventory_id=X` from Inventory. |
| `/inventory` | Two-tier tabs: Sales (Available/Pending/Sold) and Storage & Handling (On Site/Checked Out). On Site drops the always-empty Checkout column. Checked Out rows get a side-effect-free **Reprint** link to `/sh-pickup-receipt/:id`. |
| `/releases` | **Inline quota edit** mirroring pickup. Same blocking rule. |
| Quote promote | Lines panel with per-line container dropdown. Default mapping is positional; reassign swaps. Promote button gated on 1:1. Server enforces `containers.length == lines.length` (400 `container_count_mismatch`). |
| Receipt template | One timestamp (Printed), customer signature line at the bottom. |
| Schema | **0022** — `pickup_numbers`, `pickup_number_assignments`, `sh_inventory.pickup_damage`. **0023** — trigger that resets `pickup_numbers.is_complete = false` when assignment count drops below quota. |
| Copy | "S&H" → "Storage & Handling" sitewide (narrow tiles abbreviated to "Storage"). |

---

## Post-deploy spot-checks

| Surface | Expected |
| --- | --- |
| `/inventory` | Top-level Sales / Storage & Handling toggle visible; sub-tabs render |
| `/pickup-numbers` | Empty grid (no pickups created yet); + New pickup works |
| `/outbound?sh_inventory_id=X` | S&H tab default, box pre-selected on step 1 |
| `/releases` | Quota edit button on each drawer; 409 if you try to lower below filled |
| `/sh-pickup-receipt/:id` | 80mm preview + AirPrint dialog. One timestamp, signature block |
| `~/airtight-cutover/verify-migrations.sql` on prod | 13/13 PASS + trigger live-fire PASS |

---

## Operator to-do still outstanding from the audit

42 S&H units pre-enrolled in release `AUDIT-5-29-26` waiting for manual `/intake`:

| Category | Units |
| --- | --- |
| sh_dd (3) | XYZU 220000-9, XYZU 220003-5, XYZU 220017-0 |
| flexbox (15) | FAMU 8270 90/91/92/94 + 102 + 149/157/160/161/165/166/175 + FAMU 891888-6 + FXLU 892476-9 + CICU 202837-0 |
| ts_free (10) | TSQU 000000-0 · TRDU 657039-2 · TRDU 666203-5 · UNSU 001361-6/003768-6/004759-7/006245-7/006625-7/009181-4/020983-0 |
| ts_flat_rate (14) | 1, 2, 3, 4, 5, 6, 7, 8, 9, 12, 13, 14, 15, 16 |

Plus invoice **#202605015** (RFCU 217783-4) is still active with `status='awaiting'` — confirm with the customer; the box left the yard 2026-05-27.

---

## Still deferred (not built)

- **S&H invoice editing** — cron-generated invoices remain read-only. No PATCH/DELETE for lines, no UI affordance. When this becomes routine, add PATCH `/api/v2/sh-invoice/:id/lines/:lineId` and an `ShInvoiceDetail` edit mode mirroring `InvoiceEditor` with the limited S&H line shape.
- **Pickup-summary PDF report** — foundation is in (pickup tables, assignment view). Ship when there's demand.
- **Force-detach admin UI** for `DELETE /api/v2/pickup/:id/assignments/:sh_inventory_id` — handler exists, no front-door yet.
- **Site-wide trash-icon adoption** beyond invoice + quote.
- **Junk-row sweep** — 3 historical TEST rows + 1 blank-unit row still in outbound on prod.
- **Modal-backdrop "discard?"** integration on editor modals.

---

## Open threads

- **Twilio A2P 10DLC** — resubmitted 2026-05-29 with sole-prop wording. Outbound stepper hides Driver SMS via `GET /api/v2/report/config/sms` until creds are added. When approved: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_MESSAGING_SERVICE_SID=MGde4ad37ed70fb2bd1bd9330c009ced23` in EC2 `~/airtight-container/.env`.
- **AirPrint `/reports/:id/print` E2E** — operator iPad still recovering from activation-lock. iPhone smoke (2026-05-25) proved iOS → Mango → Star works. Hardware: Star TSP654II + GL.iNet Mango + UniFi U6-Mesh (printer IP `192.168.8.221`). Same path will fire for the new `/sh-pickup-receipt/:id`.
- **Spanish translation review** — first-pass machine output is live in yard flows + Outbound; native review deferred.
- **Phase 8 (QuickBooks)** — deferred.

---

## Conventions

- `main` deploys on push. **Don't push to main without explicit deploy intent.**
- Migrations stay numbered + applied manually via `psql -f` (cutover script handles the loop). Prod is at **0021** until tonight's cutover ships 0022/0023.
- 49 stale `phase-*` branches deleted this session; only `main`, `2.0`, the active PR branch, and 8 `worktree-agent-*` branches remain.
- `sh-outbound.test.ts` carries `retry: 1` on the describe block to absorb a known cross-file pool race with `sh-month-end`. Both tests pass on every retry; this is flake absorption, not a correctness bandaid.

---

## Don't

- **Don't push to `main` without explicit deploy intent.** GHA fires on every `main` push.
- **Don't `--no-verify`** any commit hook.
- **Don't `psql -f <dump>` into an existing populated DB.** Drop + create first. The cutover script handles this; only matters if you're hand-restoring.
- **Don't `pg_dump` as `postgres` directly into `~/airtight-cutover/`** on EC2 — postgres can't write into `/home/ubuntu` (700 perms). Stage via `/tmp/cutover/` as the script does.
- **Don't delete `~/airtight-cutover/` archives** before the new prod state has soaked for a week.
- **Don't touch the `is_complete` flag on `pickup_numbers` from app code.** The outbound endpoint and the 0023 trigger jointly own it; manual writes can drift the flag from the assignment count.

---

## At end of session

Update this file in place. Don't accumulate dated subsections — overwrite stale state.
