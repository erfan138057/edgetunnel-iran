# edgetunnel-iran

VLESS / Trojan / Shadowsocks over a Cloudflare Worker, tuned for Iranian lines, with the
2026 TLS bypass recipe baked into every subscription node.

> **Before you do anything else:** **set your own password**, and never publish your worker URL, tokens or subscription link.
> The `ADMIN` token, the `KEY` and the subscription token are the passwords of your deployment.
> Full rules: [Privacy and security](#8-privacy-and-security-required).

## 1. What you get

- VLESS over WebSocket served by a Cloudflare Worker (`_worker.js`).
- Server-side config stored in KV (`config.json`), editable through the web admin panel.
- Subscription endpoint `/sub?token=...` exporting share links.
- This build forces the bypass recipe on every node:
  - `fp=unsafe` (TLS fingerprint)
  - `alpn=http/1.1`
  - `cs=` the 13-suite cipher list
  - `fm=` two-stage finalmask JSON

## 2. Requirements

- A Cloudflare account (free plan works).
- Node.js 18+ and wrangler (`npm i -g wrangler`).
- A client that can apply the recipe (see section 6).

## 3. Deploy

```powershell
wrangler login
wrangler kv namespace create CONFIG      # paste the returned id into wrangler.toml
wrangler deploy
```

- Keep `keep_vars = true` in `wrangler.toml`: runtime config lives in KV, so redeploys never wipe your settings.
- Worker vars available for tuning: `PRELOAD_RACE_DIAL`, `PROXY_CONCURRENT_DIAL`.

## 4. Configure

1. Set your password secrets in the Cloudflare dashboard (Worker -> Settings -> Variables and Secrets): `ADMIN` and `KEY`. Use long random values; see section 8.
2. Open `https://<your-worker>.workers.dev/admin?token=<ADMIN>`.
3. Edit `config.json` in KV. Fields that matter for the bypass:
   - `UUID` - client id; regenerate it, do not reuse sample values
   - `Fingerprint` - keep `unsafe`
   - `ALPN` - keep `http/1.1`
   - `CipherSuites` - the colon-joined suite list
   - `FinalMask` - the two-stage fragment JSON
   - `preferredSubGen` - IP pool; `randomIp` resamples addresses on every fetch
   - `ECH` - off by default; turn on only if your client supports ECH
4. Save. Takes effect on the next request; no redeploy needed.

## 5. Subscription

- `https://<your-worker>.workers.dev/sub?token=<SUB_TOKEN>` returns plain-text `vless://` lines when the User-Agent contains "mozilla"; other agents get base64 (`?b64=1` forces it).
- Every fetch re-samples Cloudflare IPs and path values. If a list works well, save a copy; that file is your fixed snapshot.

## 6. Client requirements (read this)

The recipe only works in clients that understand `fm=` and `cs=`:

| Client | Result |
|---|---|
| PattN / PattNG | Full support. Import links or the sub directly. |
| Xray-core >= 26.6.27 | Full support when the link fields are passed through. |
| v2rayN (stock) | Imports `fm`, `fp`, `alpn` but DROPS `cs`. Its bundled core must be >= 26.6.27, otherwise every node fails with `LengthMin can't be 0`. |
| Apps that ignore `fm=` | Send a plain TLS hello. On the filtered line this fails; no config value can compensate for a client that never applies the recipe. |

**Fixing v2rayN:** download the official `Xray-windows-64.zip` (>= 26.6.27), rename `bin\xray\xray.exe` to `xray-old.bak.exe`, copy the new `xray.exe` in, restart v2rayN.

## 7. Why it fails otherwise (background)

The line applies two rules: a TLS ClientHello fingerprint rule (blocked stacks are silently dropped even when curl passes the same IP and SNI) and a random 6-packet upload cap on IPv4. The finalmask splits the hello into the permitted shape; `fp=unsafe` plus the cipher list complete the disguise. Use unpublished or clean Cloudflare IPs (the random pool already avoids published ones).

## 8. Privacy and security (required)

**Everyone must set a password.** A fresh deployment is not safe to share until you do.

| Credential | What it protects | Where it lives |
|---|---|---|
| `ADMIN` | the admin panel (full control of the config) | Worker secret, Cloudflare dashboard |
| `KEY` | UUID derivation and the quick link | Worker secret, Cloudflare dashboard |
| Subscription token | your tunnel: whoever has `/sub?token=...` spends your traffic | KV config, set in the admin panel |
| `UUID` | client identity | KV config, regenerate per deployment |

Rules:

1. Keep your deployment private: never publish your worker URL, tokens or subscription links - not in a public repository, fork, issue, pull request, screenshot or video. This repository ships with placeholders only; keep it that way in anything you publish.
2. Treat every token like a password: long, random, unique per deployment, and rotated the moment one appears in a screenshot, chat or log.
3. Never commit secrets to git. Secrets live in the Cloudflare dashboard or in KV; neither is part of this repository.
4. If a token leaks, rotate it immediately - `ADMIN`/`KEY` in the dashboard, subscription token in the admin panel. A leaked link cannot be recalled after the fact.

## 9. CI / GitHub Actions

- `verify.yml` - runs on every push: syntax check, English-only check, settings sanity.
- `deploy.yml` - manual deploy; requires the repository secrets `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` (your own account id).
- `health.yml` - scheduled probe; set the repository variable `WORKER_URL` to your worker. Until that variable exists, the schedule stays dormant.
- Repository checks enforce English-only text: no CJK, Arabic, Persian or Cyrillic characters in tracked files.

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `LengthMin can't be 0` | Core older than 26.6.27 | Section 6 core swap |
| Test passes, browsing fails | Client not applying `fm=` | Use PattN/PattNG or fix the core |
| All nodes die over time | IPs blocked | Re-fetch the sub for fresh IPs or pin a clean address |
| Ping tests show nothing | Ping is meaningless for these configs | Load a real page instead |

## 11. License

Copyright (C) 2026 edgetunnel-iran contributors.

GNU General Public License, version 2 - see [LICENSE](LICENSE).

## 12. Credits

Special thanks to the original creators this project builds on:

- **[zizifn](https://github.com/zizifn)** - the original **EdgeTunnel** ([zizifn/edgetunnel](https://github.com/zizifn/edgetunnel), 2020): running V2ray inside the Cloudflare edge. Everything in this repository descends from that work.
- **[patterniha](https://github.com/patterniha)** - the creator of **PattNG** ([patterniha/PattNG](https://github.com/patterniha/PattNG)) and PattN, the clients whose finalmask / fingerprint / cipher-suite recipe makes the 2026 bypass possible, and whose share-link format this subscription speaks.

Also thanks to the maintainers of the extended edgetunnel line (`cmliu/edgetunnel`) and the v2rayNG / v2rayN upstreams (`2dust`), and to the core this worker runs on: `XTLS/Xray-core`.
