# INFRAV3.md — mujica.dev migration plan

> Per-project execution plan for **mujica.dev** (the personal landing page)
> under the Mujica v3 infra (see `arcturus/INFRASTRUCTURE.md`). This is the
> simplest project in the portfolio — a static `index.html` + `style.css`.
> The work is mostly DNS + email records, not code.

---

## 1. Current state (as-is)

| Concern | Today |
|---|---|
| Code | Static site: `index.html`, `style.css`, `og.svg`, `.nojekyll`. No build step. |
| Hosting | **GitHub Pages.** `.nojekyll` and `CNAME` (= `www.mujica.dev`) are the giveaways. |
| Domain | `mujica.dev` (apex + `www`). Current registrar unknown. |
| DNS | Wherever the registrar points — likely default registrar DNS. |
| Email | Used as the canonical `dmarc@mujica.dev` reporting address in `INFRASTRUCTURE.md` §9.4. No mailbox exists yet. |
| Secrets | None. |

---

## 2. Target state (per INFRASTRUCTURE.md)

Two locked decisions force change here:

- **§8.1:** every domain moves to Cloudflare DNS.
- **§9:** every domain gets MXroute + Resend records and a DMARC policy, with
  `rene@mujica.dev` as the primary human mailbox.

What does *not* change: the site itself. The smallest correct move is to
**keep static hosting on GitHub Pages, but front it with Cloudflare DNS**
(grey-cloud or orange-cloud — see §3.2 below). Moving the static files onto
the VPS would burn time and a tiny slice of droplet resources for zero gain;
it adds neither features nor isolation. Move only if a future project
consolidates here.

---

## 3. Required changes

### 3.1 Cloudflare onboarding
- Add `mujica.dev` zone to Cloudflare.
- Switch the registrar's nameservers to Cloudflare's two assigned ones (§8.1).
- Consider transferring the registration itself to Cloudflare Registrar
  at-cost — optional and slow (60-day registrar lock from any recent move).

### 3.2 DNS records — site hosting

The clean choice is **keep GitHub Pages but proxy through Cloudflare** so
all traffic for the domain flows the same way as the rest of the portfolio.

```
A     mujica.dev   185.199.108.153    proxied   # GitHub Pages
A     mujica.dev   185.199.109.153    proxied
A     mujica.dev   185.199.110.153    proxied
A     mujica.dev   185.199.111.153    proxied
CNAME www          rene-mujica.github.io.   proxied   # adjust to actual GH user
```

Set **SSL/TLS mode = Full** (not Full strict) for the GitHub Pages origin —
GH issues a Let's Encrypt cert for the custom domain, but the
cert-presented-name semantics differ enough that Full strict is brittle.
Every *other* project in the portfolio uses Full strict per §8.3 because
Traefik on the VPS issues its own LE cert; document this exception on this
page only.

If a future migration moves the static site onto the VPS (Caddy or a tiny
nginx container on Dokploy), flip the A records to the droplet IP and
upgrade to Full strict at that time.

### 3.3 Email records (§9.4 + §9.5)

```
MX    @              <assigned>.mxrouting.net.       priority 10
TXT   @              "v=spf1 include:mxlogin.com -all"
TXT   x._domainkey   <MXroute DKIM value>
TXT   _dmarc         "v=DMARC1; p=none; rua=mailto:dmarc@mujica.dev; pct=100"
```

`dmarc@mujica.dev` is the **rua** mailbox for every domain in the portfolio
per §9.4 — create that mailbox in MXroute DirectAdmin alongside
`rene@mujica.dev`.

`mujica.dev` is not (yet) a sender for application email, so the
`send.mujica.dev` Resend subdomain block from §9.5 is **not required** at
launch. Add it the day this domain sends transactional mail.

### 3.4 Cache & security rules (§8.4, §8.5)

- Cache Rule: `mujica.dev/*` static → Cache Eligible, long Edge TTL (the
  whole page is static).
- Security Level: Medium. Bot Fight Mode on.
- No origin firewall lock (§8.6) because the origin is GitHub Pages, not
  the droplet. The droplet firewall changes from §8.6 don't apply here.

### 3.5 Code changes

None. The repository contains only `index.html`, `style.css`, `og.svg`.
There are no env vars to rotate, no secrets to move, no API contracts to
update.

The single optional change: when DNS moves, double-check the `og:url` and
`twitter:image` absolute URLs in `index.html` still resolve as
`https://www.mujica.dev/og.svg`. They do today; they will still after the
Cloudflare proxy is enabled. Nothing to edit.

---

## 4. Cutover sequence

1. Add zone to Cloudflare; copy the two assigned nameservers.
2. At the current registrar, replace nameservers with Cloudflare's.
3. Wait for propagation (Cloudflare's UI confirms; typically minutes-to-hours).
4. Add the four GitHub Pages `A` records and the `www` `CNAME` (§3.2).
5. Set SSL/TLS mode to Full.
6. Verify `https://mujica.dev` and `https://www.mujica.dev` load and the
   cert chain is Cloudflare's.
7. Add MX / SPF / DKIM / DMARC records (§3.3) once MXroute is provisioned
   and the DKIM value is in hand. These can land any time after step 5.
8. Create the `rene@mujica.dev` and `dmarc@mujica.dev` mailboxes in MXroute.

---

## 5. Project-specific risks

- **GitHub Pages + Cloudflare Full strict is brittle.** Use Full, not Full
  strict — documented exception to §8.3.
- **GitHub Pages requires re-verifying the custom domain** in the repo's
  Pages settings after the nameserver change. The "DNS check" widget on
  the Pages settings page will go red briefly and then green.
- **`dmarc@mujica.dev` is the report sink for the entire portfolio.** Make
  sure that mailbox exists and is actually monitored before tightening any
  domain's DMARC from `p=none` to `p=quarantine` (§9.4). One person should
  actually read the weekly aggregate reports.
- **The site is the lowest-stakes thing in this repo.** If the migration
  hits a snag, leave it on the current setup and come back later — nothing
  downstream depends on it shipping first.

---

## 6. Out of scope

- Rewriting the static site as a framework-built page (Astro, Eleventy,
  etc.).
- Moving hosting onto the VPS. Revisit only if a future project (a blog,
  a writing space) needs a build step.
- Setting up the `send.mujica.dev` Resend subdomain — defer until this
  domain starts sending application email.
