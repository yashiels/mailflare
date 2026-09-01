# Mailflare API recipes

Copy-paste commands for operating the deployment. Assumes:
```bash
ZID=<zone id for the domain>                 # get via "list zones" below
AID=e321efa4517397f720d450e7951439ae         # Skyner account
SCOPED=$(opd read "op://<Private-vault>/<mailflare-token-item>/credential")   # runtime CF_TOKEN
```
`opd` = desktop 1Password CLI (Private vault). `op-sa` cannot read Private.
Note: `opd`/tmux can clobber PATH mid-script — use absolute `curl`/`python3` if a loop errors.

## List Cloudflare zones (find a zone id)
```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?per_page=50" \
  | python3 -c 'import sys,json;[print(z["name"],z["id"]) for z in json.load(sys.stdin)["result"]]'
```

## Inspect email state for a domain
```bash
# routing status (enabled / status)
curl -s -H "Authorization: Bearer $SCOPED" "https://api.cloudflare.com/client/v4/zones/$ZID/email/routing"
# MX records
curl -s -H "Authorization: Bearer $SCOPED" "https://api.cloudflare.com/client/v4/zones/$ZID/dns_records?type=MX"
# routing rules (address -> worker)
curl -s -H "Authorization: Bearer $SCOPED" "https://api.cloudflare.com/client/v4/zones/$ZID/email/routing/rules?per_page=20"
# sending subdomains (outbound identity)
curl -s -H "Authorization: Bearer $SCOPED" "https://api.cloudflare.com/client/v4/zones/$ZID/email/sending/subdomains"
# verified destination addresses (account level)
curl -s -H "Authorization: Bearer $SCOPED" "https://api.cloudflare.com/client/v4/accounts/$AID/email/routing/addresses"
```

## Mint / widen the scoped CF_TOKEN
Minting needs a token with API Tokens Write (the dash login token in 1Password). Permission
group ids (fetch fresh via `/user/tokens/permission_groups?per_page=300` if they change):

| Group | id | scope |
|---|---|---|
| Zone Read | c8fed203ed3043cba015a93ad1616f1f | zone |
| Zone Settings Read | 517b21aee92c4d89936c976ba6e4be55 | zone |
| Zone Settings Write | 3030687196b94b638145a3953da2b699 | zone |
| DNS Read | 82e64a83756745bbbb1c9c2701bf816b | zone |
| DNS Write | 4755a26eedb94da69e1066d98aa820be | zone |
| Email Routing Rules Read | 1b600d9d8062443e986a973f097e728a | zone |
| Email Routing Rules Write | 79b3ec0d10ce4148a8f8bdc0cc5f97f2 | zone |
| Email Sending Read | 2d5b4b1f6c89487bb7184c2c1dcd3bf1 | account |
| Email Sending Write | 5df633d6b41c42bcaf5b4a62b9d14b64 | account |
| Email Routing Addresses Read | 5272e56105d04b5897466995b9bd4643 | account |
| Email Routing Addresses Write | e4589eb09e63436686cd64252a3aebeb | account |

Body shape (two policies — one zone-scoped, one account-scoped). To ADD a domain, include its
zone id in the zone policy's `resources` (`com.cloudflare.api.account.zone.<ZID>: "*"`) and
`PUT /user/tokens/{token-id}`. Runtime token id: `d06b76065c4d2adbe853ab77eba8e3bf`.
A perms-only edit keeps the same token value, so no need to re-set the CF_TOKEN worker secret.

## Provision Email Sending for a domain (fix send)
```bash
TAG=$(curl -s -X POST -H "Authorization: Bearer $SCOPED" -H "Content-Type: application/json" \
  --data "{\"name\":\"<domain>\"}" \
  "https://api.cloudflare.com/client/v4/zones/$ZID/email/sending/subdomains" \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["result"]["tag"])')
# Cloudflare auto-adds DKIM/SPF/bounce-MX. Then tell Mailflare:
wrangler d1 execute mailflare --remote --command \
  "UPDATE domains SET sending_enabled=1, sending_subdomain_tag='$TAG' WHERE hostname='<domain>'"
```

## MX cutover + SPF (per domain)
```bash
# delete old non-Cloudflare MX (get id from the MX list above)
curl -s -X DELETE -H "Authorization: Bearer $SCOPED" \
  "https://api.cloudflare.com/client/v4/zones/$ZID/dns_records/<MX_RECORD_ID>"
# set SPF to Cloudflare (PATCH the apex TXT SPF record id)
curl -s -X PATCH -H "Authorization: Bearer $SCOPED" -H "Content-Type: application/json" \
  --data '{"content":"v=spf1 include:_spf.mx.cloudflare.net ~all"}' \
  "https://api.cloudflare.com/client/v4/zones/$ZID/dns_records/<SPF_RECORD_ID>"
```

## D1 queries (read state)
```bash
export CLOUDFLARE_ACCOUNT_ID=e321efa4517397f720d450e7951439ae
wrangler d1 execute mailflare --remote --json --command "SELECT hostname,routing_status,sending_enabled,sending_subdomain_tag FROM domains"
wrangler d1 execute mailflare --remote --json --command "SELECT local_part,display_name,type FROM mailboxes"
wrangler d1 execute mailflare --remote --json --command "SELECT direction,from_addr,to_addr,subject,status FROM messages ORDER BY created_at DESC LIMIT 10"
wrangler d1 execute mailflare --remote --json --command "SELECT status,error FROM outbound_jobs ORDER BY created_at DESC LIMIT 5"
```

## Gotchas
- Adding an MX via API may 403 with "zone managed by Email Routing" — Cloudflare auto-manages
  the bounce/routing MX when you enable routing/sending, so you usually don't need to add it.
- Two DKIM TXT records at one selector break verification — keep exactly one.
- `/user/tokens/verify` returns "Invalid API Token" for account-scoped tokens even when valid;
  test capability with a real call instead.
