---
name: mailflare-ops
description: >-
  Operate the yashiels/mailflare self-hosted email deployment on Cloudflare Workers
  (mailflare.skyner-agents.workers.dev, Skyner account). Use this whenever the task
  involves this Mailflare instance — deploying or redeploying it, adding a new email
  domain (send + receive), creating mailboxes, rotating/scoping the CF_TOKEN, or
  fixing outbound send errors, Email Routing "misconfigured", MX cutover conflicts,
  token 403s, spam/deliverability, or the blocked lint hook on commit. Trigger even
  when the user just says "mailflare", "my email server", "the webmail", "add <domain>
  to mailflare", or describes a symptom (e.g. "sending fails / not a verified address")
  without naming the tool.
---

# Mailflare Ops

Self-hosted fork of `hieunc229/mailflare` — a **webmail app on Cloudflare Workers**
(Next.js/OpenNext + D1 + R2 + Queues + Durable Objects + Email Routing/Sending).
Repo: `yashiels/mailflare`. This skill is the operator runbook. The repo `AGENTS.md`
has the same facts in prose; read it for background.

## Deployment facts (memorize)

| Thing | Value |
|---|---|
| Live URL | https://mailflare.skyner-agents.workers.dev (webmail = `/`, admin = `/admin`) |
| CF account | Skyner — `e321efa4517397f720d450e7951439ae` |
| Worker name | `mailflare` (must equal `CF_EMAIL_WORKER_NAME` + `WORKER_SELF_REFERENCE`) |
| D1 | `mailflare` id `93ca251a-7d71-4d8f-bab6-29c5f75e5572` |
| R2 | `mailflare-raw` |
| Queues | `mailflare-inbound`, `mailflare-outbound` |
| Plan | Workers Paid ($5/mo) — required for sending + queues |
| Domains | sookdeo.co.za, yashiel.dev (both apex, send+receive live) |
| Runtime token | `CF_TOKEN` secret = 1Password Private "Cloudflare – MailFlare email token" |
| Deploy token | env `CLOUDFLARE_API_TOKEN` (broad account token; Workers/D1/R2/Queues) |

## How it works (so fixes make sense)

- **Inbound:** Cloudflare Email Routing (per domain) → catch-all + per-address rules → the
  Worker → stored in D1 (metadata) + R2 (raw MIME/attachments). Free, unlimited domains.
- **Outbound:** `src/lib/email/send.ts` → `env.EMAIL.send()` (Cloudflare `send_email` binding).
  The binding only sends to **verified** recipients UNLESS the sending domain has a Cloudflare
  **Email Sending subdomain** provisioned. Our `provision.ts` patch provisions one for every
  domain (apex included) so sending to any recipient works with From = the real address.
- **No IMAP/POP/SMTP.** Webmail only. Phone = open URL → Add to Home Screen (PWA). If a user
  needs native Mail-app support, Mailflare can't do it (different host required).

## Our customizations vs upstream (re-apply if upstream update clobbers them)

1. `src/lib/licenses/service.ts` — `toLicenseStatus` + `getLicenseEntitlements` forced to
   `team` with all entitlements (self-hosted, no Paymug license).
2. `src/lib/domains/provision.ts` — removed the `isZoneApex → sendingEnabled=false` branch so
   apex domains provision an Email Sending subdomain like subdomains do.

## Runbook

### Deploy / redeploy
```bash
# in a clone of yashiels/mailflare (deploy from a linked worktree, not a canonical checkout)
export CLOUDFLARE_ACCOUNT_ID=e321efa4517397f720d450e7951439ae
export WRANGLER_SEND_METRICS=false OPEN_NEXT_DEPLOY=true
npm install
npm run deploy:local     # next build + opennext build + d1 migrate remote + wrangler deploy
```
- Put the D1 `database_id` in local `wrangler.jsonc` but **never commit it** (per-account; D1 error 7404).
- Verify after: `curl -s -o /dev/null -w '%{http_code}' https://mailflare.skyner-agents.workers.dev/`

### First-time secrets (once per Worker)
```bash
printf '%s' "<scoped-token>" | wrangler secret put CF_TOKEN
printf '%s' "e321efa4517397f720d450e7951439ae" | wrangler secret put CF_AID
printf '%s' "mailflare" | wrangler secret put CF_EMAIL_WORKER_NAME
printf '%s' "93ca251a-7d71-4d8f-bab6-29c5f75e5572" | wrangler secret put D1_DATABASE_ID
```

### Add a new domain (send + receive)
1. **Widen `CF_TOKEN`** to include the new zone: `PUT /user/tokens/{id}` adding the zone to the
   zone-scoped policy resources. Needed perms per zone: Zone Read, Zone Settings R+W, DNS R+W,
   Email Routing Rules R+W; account: Email Sending R+W, Email Routing Addresses R+W.
2. **Admin → Domains → add domain.** Our patch auto-enables Routing + Sending (apex OK).
3. **Delete the old non-Cloudflare MX** on that zone first, or connecting fails with 409 code 2008.
4. **Fix SPF** to `v=spf1 include:_spf.mx.cloudflare.net ~all`. Keep/repair DKIM.
5. Create mailboxes; send a test in + out.

See `references/api-recipes.md` for the exact curl commands (list zones, mint/widen token,
provision sending subdomain, MX/SPF edits, D1 queries).

## Fixes (symptom → cause → action)

- **Send fails: "destination address is not a verified address"** → domain has no Email Sending
  subdomain (or sending disabled). Ensure the `provision.ts` patch is deployed. To fix a domain
  by hand: `POST /zones/{zid}/email/sending/subdomains {name:"<domain>"}`, then in D1
  `UPDATE domains SET sending_enabled=1, sending_subdomain_tag='<tag>' WHERE hostname='<domain>'`.
  Cloudflare auto-adds the DKIM/SPF/bounce-MX DNS. Dedupe DKIM if you also added it manually.
- **Admin Domains panel looks like "no routing"** → misread. `Email Routing: {records:[], missing:[], status:"ready"}`
  means nothing outstanding (MX/rules already in place), not unconfigured. `status:"ready"` is the signal;
  confirm with `GET /zones/{zid}/email/routing` + apex MX + rules; test by receiving a real email.
- **Email Routing "misconfigured/locked"** → SPF not pointing at Cloudflare, or DNS still
  propagating. Set SPF to `v=spf1 include:_spf.mx.cloudflare.net ~all`; recheck `GET /zones/{zid}/email/routing`.
- **Connect domain → 409 code 2008 "Non-Cloudflare MX records exist"** → delete the old cPanel/host MX first.
- **Any email-routing call → 403 code 10000** → `CF_TOKEN` missing a scope. It needs the full set
  in step 1 above, including the Read variants (Cloudflare requires explicit Read groups for GET).
- **Mail lands in spam** → new-domain warmup. Tighten DMARC `p=none` → `p=quarantine` → `p=reject`
  once alignment holds; send real content; ramp volume. SPF/DKIM already aligned.
- **Commit blocked by `[yash lint]`** → upstream lint is broken (Next 16 removed `next lint`;
  eslintrc incompatible). `next build` is the real gate and it passes. Commit with
  `SKIP_YASH_LINT_POLICY=1`. Proper fix: migrate eslint to flat config.
- **Font renders serif** → Geist web font not applied under OpenNext (cosmetic). Force
  `font-family` on `body` in globals.css.

## Inspecting live state (read-only)
Token: `opd read "op://<Private-vault>/<mailflare-token-item>/credential"`. Then hit the CF API
(routing status, MX, rules, destinations, sending subdomains) or query D1 with
`wrangler d1 execute mailflare --remote --json --command "SELECT ..."`. Recipes in
`references/api-recipes.md`.

## Commit policy (this repo)
Fork under `yashiels/*` → direct push to `main` allowed. Every commit needs an issue ref
(`#N`); issues were enabled on the fork. Message: `type(scope): summary #N`. GPG-signed
automatically; no AI attribution. Close the issue when the change is pushed.
