# AGENTS.md — Mailflare (yashiels fork)

Self-hosted fork of [`hieunc229/mailflare`](https://github.com/hieunc229/mailflare) —
a webmail app running entirely on Cloudflare Workers (Next.js/OpenNext + D1 + R2 +
Queues + Durable Objects + Email Routing/Sending). This file documents **our**
deployment, customizations, and operations. Upstream docs live in `docs/`.

## Live deployment

- **URL:** https://mailflare.skyner-agents.workers.dev  (webmail = root, admin = `/admin`)
- **Cloudflare account:** Skyner — `e321efa4517397f720d450e7951439ae`
- **Worker name:** `mailflare` (must match `CF_EMAIL_WORKER_NAME` + `WORKER_SELF_REFERENCE`)
- **Resources:** D1 `mailflare`, R2 `mailflare-raw`, Queues `mailflare-inbound`/`mailflare-outbound`,
  DO `RealtimeHub`, Workflow `mailflare-database-backup` (daily 02:00).
- **Plan:** Workers Paid ($5/mo) — required for sending + queues. R2 Paid also active.
- **Connected domains:** `sookdeo.co.za`, `yashiel.dev` (both apex, send+receive live). More to be added.
  - A clean domain with no prior mail (e.g. yashiel.dev) is best-case: Cloudflare auto-adds apex MX,
    apex SPF (`include:_spf.mx.cloudflare.net`), and `_dmarc p=reject` on connect — good deliverability
    from day one. Domains with legacy cPanel records (e.g. sookdeo.co.za) need the old MX deleted and
    SPF repaired manually first.

## Our customizations (vs upstream)

1. **Team plan unlocked** — `src/lib/licenses/service.ts`: `toLicenseStatus` and
   `getLicenseEntitlements` are forced to `team` with all entitlements true (self-hosted,
   no Paymug license). DB row stays `community/inactive`; the app grants Team at runtime.
2. **Apex Email Sending** — `src/lib/domains/provision.ts`: removed the
   `isZoneApex → sendingEnabled=false` branch so apex domains (e.g. `sookdeo.co.za`)
   provision a Cloudflare Email Sending subdomain like any other. Cloudflare accepts the
   apex as a sending identity, which lifts the `send_email` binding's verified-recipient
   restriction. Without this, outbound fails with "destination address is not a verified address".

Both shipped in commit `9a4ff31` (#1).

## Architecture / key facts

- **Inbound:** Cloudflare Email Routing → catch-all + per-address rules → the Worker → D1/R2.
  Free, unlimited domains.
- **Outbound:** `src/lib/email/send.ts` calls `env.EMAIL.send()` (Cloudflare `send_email` binding).
  Requires the sending domain to have a Cloudflare Email Sending subdomain (see customization #2).
- **No IMAP/POP/SMTP.** This is webmail only — no native mail-app (iOS Mail/Outlook) support.
  Phone use = open the URL, Add to Home Screen (PWA).
- **Storage:** messages in D1, raw MIME + attachments in R2.

## Runbook

### Deploy / redeploy (manual)
```bash
# from a local clone of yashiels/mailflare
export CLOUDFLARE_ACCOUNT_ID=e321efa4517397f720d450e7951439ae
npm install
npm run deploy:local        # next build + opennext build + d1 migrate remote + wrangler deploy
```
- `CLOUDFLARE_API_TOKEN` in env must have Workers/D1/R2/Queues write (the broad account token).
- `wrangler.jsonc` needs the D1 `database_id` locally — **do NOT commit it** (per-account; causes D1 error 7404).
  Our D1 id: `93ca251a-7d71-4d8f-bab6-29c5f75e5572`.

### Worker secrets (set once via `wrangler secret put`)
- `CF_TOKEN` — **runtime** email token (NOT the deploy token). Scoped token
  `mailflare-email-runtime` in 1Password Private → "Cloudflare – MailFlare email token".
  Scopes per connected zone: Zone Read, Zone Settings R+W, DNS R+W, Email Routing Rules R+W;
  account: Email Sending R+W, Email Routing Addresses R+W.
- `CF_AID` = account id. `CF_EMAIL_WORKER_NAME` = `mailflare`. `D1_DATABASE_ID` = the D1 id (for backups).

### Add a new domain (send + receive)
1. Widen `CF_TOKEN` zone resources to include the new zone (`PUT /user/tokens/{id}`).
2. In `/admin` → Domains → add the domain (auto-enables routing + sending via our patch).
3. Delete any pre-existing non-Cloudflare MX (Email Routing refuses to write MX otherwise — error 2008).
4. Fix SPF to `v=spf1 include:_spf.mx.cloudflare.net ~all`; keep/repair DKIM; DMARC as desired.
5. Create mailboxes; test in + out.

### "No routing" in the Domains DNS panel — usually a misread
The admin DNS panel shows `Email Routing: { "records": [], "missing": [], "status": "ready" }`.
`records: []` means **nothing outstanding**, not "no routing configured" — the MX/rules are already
in place so they aren't re-listed. `status: "ready"` is the real signal. Confirm via
`GET /zones/{zid}/email/routing` (expect enabled/ready) + the apex MX + rules. The definitive test is
sending a real email to an address on the domain and seeing it in the webmail inbox.

### Common fixes
- **"destination address is not a verified address" (send fails):** the domain has no Email
  Sending subdomain. Ensure our `provision.ts` patch is deployed, or manually
  `POST /zones/{zid}/email/sending/subdomains {name:"<domain>"}` and set
  `domains.sending_enabled=1` + `sending_subdomain_tag` in D1.
- **Routing "misconfigured/locked":** SPF not pointing at Cloudflare, or DNS propagating. Fix SPF.
- **409 code 2008 on connect:** old non-Cloudflare MX present — delete it first.
- **403 code 10000 on email routing:** `CF_TOKEN` missing a scope (needs the full set above,
  incl. Read variants for GET).
- **Lint hook blocks commit:** upstream lint is broken (Next 16 removed `next lint`; eslintrc
  incompatible). `next build` is the real gate. Bypass with `SKIP_YASH_LINT_POLICY=1` until lint is migrated to flat config.

### Update from upstream
Admin → "Update Mailflare" needs `GITHUB_UPDATE_TOKEN`/`GITHUB_UPDATE_REPO`/`GITHUB_UPDATE_REF`
+ repo secrets `CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID`. Or manually merge upstream, re-apply
our two customizations if touched, and `npm run deploy:local`.

## Known follow-ups
- **Deliverability:** new-domain mail lands in spam initially. Tighten DMARC toward
  `p=quarantine`/`p=reject` once alignment is confirmed; warm up sending volume.
- **Font:** Geist web font falls back to serif under OpenNext (cosmetic). Force `font-family` on body.
- **Lint tooling:** migrate eslint to flat config so `npm run lint` works under Next 16.
