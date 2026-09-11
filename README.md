# Meta Ads API notes

Field notes from setting up a **server-to-server ads uploader**: one Meta app, system users, App Review, Full Access, and the rate-limit mess.

This is the guide we send people who ask “how did you wire the system user?” Official docs exist. They are badly maintained, names keep changing, and they do not match the headers you actually get. Use the official links at the bottom. Trust live headers and exact error codes more than the prose.

---

## TL;DR

1. Create a **Business** app owned by a **spend-free toolmaker Business Manager**.
2. Add the **Marketing API** product. You start on **Limited** access (headers still say `development_access`).
3. Create an **Admin system user** in each ad BM. Assign the app with **full control**, then the ad accounts and Pages. Mint a token **after** that.
4. Do **not** fight the Limited / “dev” tier in production. The **60 max score** is the thing that ruins you.
5. Build it as a **server-to-server app**, then **do App Review** and upgrade **Marketing API Access Tier** to **Full Access**. For S2S apps this is a form, not a screen recording. Ours took about **two hours**.
6. Full Access is the rate-limit upgrade. Headers still call it `standard_access`. Yes, that is stupid.

If you only remember one thing: **do the review. The Full Access quotas are worth it.**

---

## The naming mess (read this first)

Meta reused the same English words for three independent controls. One does not grant the others.

| What people say | What it actually is | Where you set it |
|---|---|---|
| **Full control** | Asset assignment on the system user (the app, ad accounts, Pages) | Business Settings → System users |
| **Full Access** | Marketing API **capacity tier** (rate limits, system-user caps) | App Dashboard → App Review → **Marketing API Access Tier** |
| **Advanced Access** | Per-permission / per-feature review state | App Dashboard → App Review → that permission |

Then they **renamed the tier** and did not update every page:

| UI now | Older UI / some pages | Header / API value |
|---|---|---|
| **Limited Access** (default) | Development access / “Standard Access” to the old *Ads Management Standard Access* feature | `development_access` |
| **Full Access** (after review) | “Advanced Access” to that same feature | `standard_access` |

Live proof of your tier is the header field `ads_api_access_tier`, not the dashboard copy.

Official rename notice: [Marketing API Rate Limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting) and [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization).

---

## What we run (and why)

We upload and manage ads from our own servers against Business Managers we admin. No public login, no third-party advertisers clicking “Allow”.

That means:

- **System users, never personal profile tokens.** Profile tokens die (`data_access_expires_at` ~90 days), they attach spend to a human, and they are the pattern associated with bans.
- **One app**, owned by a dedicated **toolmaker BM** that spends nothing. Share that app into each ad BM. Do **not** create a new app per BM — app *creation* is what trips verification storms, not sharing an existing app.
- **One Admin system user + one token per ad BM.** Assign only that BM’s ad accounts and Pages. A hub token that can reach every BM is one leak / one restriction away from total loss.
- **Pin a Graph version** (`v25.0` as of this writing). Never call unversioned / “latest”.

The Graph API **cannot** create Business Managers or Facebook Pages. That part stays manual.

---

## Part A — the app (once)

### 1. Toolmaker Business Manager

Create or pick a BM that will **own the app and never spend**. If an ad BM gets restricted, the app and the sharing survive.

Help: [Business Manager](https://developers.facebook.com/docs/business-management-apis/business-manager-api).

### 2. Create the app

[developers.facebook.com/apps](https://developers.facebook.com/apps) → create app → type **Business**.

Set the owning Business to the toolmaker BM (or add the app under that BM in Business Settings → Apps).

Basic settings that matter for a server-to-server uploader ([S2S apps](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps)):

| Setting | What we do |
|---|---|
| **App icon** | Company logo, 1024×1024, no Meta trademarks |
| **Business use** | **Yourself or your own business** if you only touch accounts you admin |
| **Platform** | **Website** + company URL (there is no UI) |
| **Privacy policy URL** | Required before review. Real URL. |
| **Live mode** | Stay in development until review is done |

Add the **Marketing API** product. That puts you on **Limited Access** automatically.

Record `APP_ID` and `APP_SECRET`. Those stay in a private env, never in git, never in Airtable, never in chat.

### 3. Do not publish. Do not invent a second app.

Leave it unpublished while you wire tokens. Repairing a token or adding a BM is **not** an excuse to recreate the app.

---

## Part B — system user (repeat per ad BM)

Official: [System Users](https://developers.facebook.com/docs/business-management-apis/system-users), [create](https://developers.facebook.com/docs/business-management-apis/system-users/create-retrieve-update), [install app + mint token](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens), [assign assets](https://developers.facebook.com/docs/business-management-apis/system-users/guides/permissions). Clicks: [Add a system user](https://www.facebook.com/business/help/503306463479099?id=2190812977867143).

### 1. Share the app into this ad BM

From the **toolmaker** BM’s app settings, add the ad BM. It then shows up under the ad BM → Business Settings → Apps.

Do this. Do **not** create another app.

### 2. Create an Admin system user in the ad BM

Business Settings → Users → System users → Add. Role **Admin**.

Each BM gets its own. Never reuse one system user across BMs.

### 3. Assign the app to that system user with **full control**

This is the step people skip. If the app is not assigned with full control, the scopes you tick at mint time are not actually available.

### 4. Assign **only this BM’s** ad accounts and Pages

Same UI as assigning assets to a human. Tasks we use on ad accounts: `MANAGE`, `ADVERTISE`, `ANALYZE`. Pages need to be assigned **separately** — a Page sitting in the same BM is not enough. Missing Page assignment is `code 10 / subcode 1341012` (“no permission to access this profile”).

No cross-BM assets.

### 5. Generate a long-lived system-user token

Mint **after** steps 3–4. A token does **not** pick up permissions you add later. Scope repair = mint a new token.

UI is fine. API is `POST /{system-user-id}/access_tokens` with `business_app`, `scope`, `appsecret_proof`. Meta now pushes **60-day expiring** tokens (`set_token_expires_in_60_days=true`) and some businesses cannot mint non-expiring ones. If you mint expiring, you need a rotation job.

Store one env var per BM, e.g. `META_SU_TOKEN_<BUSINESS_ID>`. Never a single global `META_ACCESS_TOKEN`.

### 6. Prove it before you onboard the next BM

```bash
GET /debug_token?input_token=TOKEN
# authenticate this call with app_id|app_secret, not the user token

GET /{business_id}?fields=id,name
# must return the BM you think you just wired
```

Check: token valid, bound to **this** `APP_ID`, required scopes present, BM id matches. `/debug_token` is the inspector: [Access Token Debugger](https://developers.facebook.com/tools/debug/accesstoken/).

---

## Scopes we actually mint

For an ads uploader that creates campaigns / ad sets / ads, reads performance, and talks to Pages:

| Scope | Why |
|---|---|
| `ads_management` | Create / edit campaigns, ad sets, ads |
| `ads_read` | Read config and reports |
| `business_management` | BM assets, system users, audits |
| `pages_read_engagement` | Page-side read for ads on Pages |
| `pages_manage_ads` | Page-linked ads (keep if you run Page ads) |
| `pages_show_list` | List Pages the token can see |

Optional, only if you use them: `pages_manage_posts`, `pages_manage_engagement`, `read_insights`.

Grant the ones you need **at mint time**. Extra scopes on the token are fine; missing ones are not.

If you **only** manage ad accounts you already admin, **standard access** to `ads_management` / `ads_read` is enough for those accounts. You still want the **Marketing API Access Tier → Full Access** feature for rate limits. Those are different buttons.

Permission reference: [Permissions](https://developers.facebook.com/docs/permissions). System-user allowed scopes: [Install apps and generate tokens](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens).

---

## Limited vs Full Access

### What Limited (dev) actually is

Default when you add Marketing API.

- Heavily rate-limited **per ad account**.
- **Ad-account score cap: 60.** Decay 300s. Hit it → blocked **300 seconds**.
- BUC `ads_management` hourly: `300 + 40 × active_ads`.
- **1 Admin system user + 1 system user** on the app. That is a hard cap.
- Fine for wiring tokens and a handful of calls. Not fine for an uploader.

The killer is the **60 max score**, not the 300+40×N BUC formula people quote. A read is 1 point, a write is 3. In theory ~60 reads or ~20 writes in the window — except **not every endpoint counts toward that score**, and the docs do not tell you which. You find out by watching headers and error `17/2446079` or `613/1487742`.

We tried to live here. Do not.

### What Full Access actually is

After App Review on the **Marketing API Access Tier** feature:

- Score cap **9000**, block **60s** (still 300s decay).
- BUC `ads_management` hourly: `100000 + 40 × active_ads`.
- Insights, custom audience, catalog quotas jump the same way.
- **10 system users + 1 admin system user**.
- Full Business Manager / Catalog API surface.

It does **not** change whether your create payload is valid. It only changes capacity.

Compare: [Authorization — Limited vs Full](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization).

### How to get Full Access

Requirements Meta currently publishes ([same page](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization#get-full-access)):

1. **≥ 500 successful Marketing API calls in the last 15 days**
2. **Error rate &lt; 15% on the last 500 calls**

Then: App Dashboard → App Review → Permissions and Features → **Marketing API Access Tier** → **Upgrade**.

Do **at least one successful call per permission** you will request, within 30 days of submit ([submission guide](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review/submission-guide)). Graph API Explorer counts.

Warmup tactic: cheap **reads** (1 point) against your own accounts until 500 is on the board. Do not burn writes on Limited. Round-robin a few endpoints (`/me/adaccounts`, a campaign GET, a Page GET) so every requested scope shows usage.

### App Review as a server-to-server app

This is the path people miss. The generic App Review tutorial screams **screen recordings**. That is for apps with a UI.

If your app has **no interface** and talks Graph from a server, use:

- [Server-to-Server Apps](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps)
- [App Review](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review)

What we did:

- App type / platform: **Website**.
- Business use: **Yourself or your own business**.
- Testing instructions: there is nothing to click. Describe that a system user token calls Marketing API to create and read ads on accounts we admin. Reuse that text in the usage boxes.
- **No screen recording.** S2S docs say: describe how the data is used; if you already described it, paste it again.
- Fill the form with an LLM. Paste the S2S page + the permission “Allowed Usage” blurb + one paragraph of what your uploader actually does. Tell it to write a specific usage description **per permission**, not the same paragraph six times. Meta’s own guide says do not copy-paste.
- Submit **Marketing API Access Tier** plus the scopes you mint.

Ours came back in about **two hours**. Consumer-app reviews can take days and want videos. S2S is a different queue.

Business Verification is a **separate** process ([Business Verification](https://developers.facebook.com/docs/apps/business-verification)). You may be prompted. It is not the same button as Full Access.

After approval, **prove it live**: one Ads Management response must show `ads_api_access_tier: "standard_access"`. Then treat Full Access as real. Dashboard copy without a header is not proof.

---

## Rate limits — several systems, none of them honest

Marketing API is **excluded** from Graph Platform rate limits. You still have **multiple independent** Marketing limits. Hitting one does not mean the others are fine. Merging them into one “utilization %” is how you false-stop the wrong account.

| System | Typical signal | Scope | Limited | Full |
|---|---|---|---|---|
| Ad-account **score** | `17/2446079`, `613/1487742`, header `X-Ad-Account-Usage` | app + ad account | max 60, block 300s | max 9000, block 60s |
| **BUC** ads_management | `80004`, header `X-Business-Use-Case-Usage` | app + ad account + BUC | `300 + 40×active_ads` / hour | `100000 + 40×active_ads` / hour |
| BUC ads_insights | `80000` | same | `600 + 400×active_ads` | `190000 + 400×active_ads` |
| Insights **platform** | `4/1504022`, `4/1504039` | **whole app** Insights | undocumented capacity | undocumented capacity |
| Mutation **QPS** | `613/5044001` | app + ad account | 100 QPS on create/edit | 100 QPS |
| App-wide | code `4` (no Insights subcode) | whole app | — | — |
| Abuse | `613` **with no subcode** | ad account | they cut your quota | same |
| Spend-cap edits | `17/1885172` | account | 10/day | 10/day |
| Ad-set budget edits | `613/1487632` | ad set | 4/hour then blocked 1h | same |

Official: [Marketing API rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting), [Graph / BUC headers](https://developers.facebook.com/docs/graph-api/overview/rate-limiting/).

### What the headers actually give you

They do **not** give remaining calls, remaining score points, or the `active_ads` denominator.

`X-Business-Use-Case-Usage` — percentages and a recovery guess:

```json
{
  "BUSINESS_ID": [{
    "type": "ads_management",
    "call_count": 45,
    "total_cputime": 80,
    "total_time": 30,
    "estimated_time_to_regain_access": 0,
    "ads_api_access_tier": "standard_access"
  }]
}
```

`call_count` / `total_cputime` / `total_time` are **0–100+ percents**, not counts. You can burn CPU to 100% on heavy Insights while `call_count` is still low.

`estimated_time_to_regain_access` is **minutes**. `X-Ad-Account-Usage.reset_time_duration` is **seconds**. Mix those up and you sleep 60× too long or 60× too short.

`X-Ad-Account-Usage` (`acc_id_util_pct`) is the score telemetry. In live Limited probes we often **never saw this header at all**, and the score error never fired, while BUC percentages climbed with raw HTTP calls. Absence means “no new information”, not “you are at 0”.

`X-FB-Ads-Insights-Throttle` is a **third** system. Do not fold it into Ads Management.

### Rules we actually use

- **Read headers. Do not invent a local score ledger.** The documented 1-point/3-point math is not an operationally valid control signal if the score header is missing.
- **Stop on throttle. Do not retry.** Retries extend the penalty.
- **Do not treat every `17` / `613` as a throttle.** Spend-cap (`17/1885172`) and ad-set budget-frequency (`613/1487632`) are business rules, not rate gates.
- **Code `4` is app-wide.** One Insights platform throttle can freeze Insights for every account on that app. Ads Management should keep moving if you classified it correctly.
- **QPS 100** is real on mutation endpoints. Burst creates will hit `613/5044001` even when hourly BUC is empty.
- Spread calls. A 0.2s floor between Graph calls is boring and works.
- Auth errors (`102`, `190`): **do not retry the same token.**

---

## Things we learned the hard way

**ZIP image upload is dead.** `POST /act_…/adimages` with a zip returns `100/1815814` — deprecated. Upload one `jpg`/`png`/… at a time. Docs and SDKs still imply otherwise.

**Image hashes are account-scoped.** Reusing another account’s `image_hash` can “succeed” and show the wrong image. Re-upload the bytes on the target account.

**`paging.next` contains the access token.** Graph returns credential-bearing paging URLs. Strip them before you log, persist, or print anything. Rotate if one leaked into a transcript.

**Tokens do not inherit later grants.** New Page, new scope, new app assignment → mint again.

**`status` is not delivery.** Filter on `effective_status` when you care whether something is actually on. An Ad can be `ACTIVE` under a `PAUSED` Ad Set (`ADSET_PAUSED`). We create Ads ACTIVE only behind PAUSED parents on purpose.

**Exact-name `filtering` works** on campaign / ad set / ad edges in v25, and is missing from a lot of parameter tables. Keep an unfiltered fallback.

**Creative names get rewritten.** Meta appends a date/hash suffix. Do not treat name equality as identity.

**Instagram identity shows up as `instagram_user_id`**, not `instagram_actor_id`, even if you did not send it.

**EU:** set `dsa_beneficiary` / `dsa_payor` (and regional categories where required) on Ad Sets. Silent omit → Ads Manager looks empty / ads pause in regulated regions.

**Do not put the token in the query string.** `Authorization: Bearer …` plus `appsecret_proof` over that same token ([secure requests](https://developers.facebook.com/docs/graph-api/guides/secure-requests)).

**Pin the version.** Deprecations are quarterly and they will change create shapes under you.

---

## Extra topics people ask us next

Worth knowing once the token works:

- **Never mutate on a 5xx / missing body.** Meta may have applied the write. Treat it as uncertain; read back; do not POST again.
- **Batch API** counts each sub-request against rate limits. It saves round trips, not quota.
- **SDK auto-pagination** will silently walk into a throttle. Paginate yourself and sleep.
- **Insights unique metrics** (`reach`) silently drop when their own header hits 100%, or with breakdowns older than ~13 months. Not an error.
- **Budget +20%** resets learning. Ad-set budget 4×/hour freezes that ad set for an hour (`613/1487632`).
- **Copy API is same-account only.** Cross-account means re-upload images, swap pixel / page / audience IDs.
- **Pages vs Ads quotas are different.** `ads_volume` on a Page is not the BUC `active_ads` denominator.

---

## Official links

### App, review, tiers

- [Create a server-to-server app](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps)
- [App Review](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review)
- [App Review submission / tutorial](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review/submission-guide)
- [Authorization (Limited vs Full, scopes vs tier)](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization)
- [Marketing API Access Tier (feature)](https://developers.facebook.com/docs/features-reference#marketing-api-access-tier)
- [Access levels](https://developers.facebook.com/docs/graph-api/overview/access-levels)
- [Business Verification](https://developers.facebook.com/docs/apps/business-verification)
- [Permissions reference](https://developers.facebook.com/docs/permissions)
- [App Dashboard](https://developers.facebook.com/apps)

### System users and tokens

- [System Users](https://developers.facebook.com/docs/business-management-apis/system-users)
- [Create / retrieve / update](https://developers.facebook.com/docs/business-management-apis/system-users/create-retrieve-update)
- [Install apps, generate / refresh / revoke tokens](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens)
- [Assign ad accounts and Pages](https://developers.facebook.com/docs/business-management-apis/system-users/guides/permissions)
- [Help Center: add a system user](https://www.facebook.com/business/help/503306463479099?id=2190812977867143)
- [Access Token Debugger](https://developers.facebook.com/tools/debug/accesstoken/)
- [Graph API Explorer](https://developers.facebook.com/tools/explorer/)
- [Secure requests / appsecret_proof](https://developers.facebook.com/docs/graph-api/guides/secure-requests)

### Rate limits and errors

- [Marketing API rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting)
- [Graph API / BUC rate limits](https://developers.facebook.com/docs/graph-api/overview/rate-limiting/)
- [Insights best practices](https://developers.facebook.com/documentation/ads-commerce/marketing-api/insights/best-practices)
- [Error handling](https://developers.facebook.com/docs/graph-api/guides/error-handling)
- [Marketing API error reference](https://developers.facebook.com/docs/marketing-api/error-reference/)

### Keep watching

- [Marketing API changelog](https://developers.facebook.com/docs/marketing-api/marketing-api-changelog/versions/)
- [Graph API changelog](https://developers.facebook.com/docs/graph-api/changelog)
- [Meta Developer blog](https://developers.facebook.com/blog/)
- [Developer Community](https://developers.facebook.com/community/)

---

## Disclaimer

Not affiliated with Meta. Not legal advice. Tiers, quotas, and form UX change without the docs catching up — we have watched the same page use three names for one flag. Verify against live `ads_api_access_tier` headers and exact `code`/`error_subcode` pairs.

If you are fighting Limited-tier score `60` right now: stop optimizing sleeps. Submit the S2S review.
