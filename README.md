# Meta Ads API notes

Field notes from wiring a **server-to-server ads uploader**: one Meta app, system users, Business Verification, App Review, Marketing API Access Tier, and rate limits.

Meta’s own pages use several names for the same flags and do not always match the headers on a live response. This guide separates **what the current docs say** from **what we observed**. Official links are at the bottom. Prefer live `ads_api_access_tier` headers and exact `code` / `error_subcode` pairs over dashboard copy.

---

## TL;DR

1. You need a **Business** app. A system user does not replace it.
2. Connect that app to a Business Manager and complete **Business Verification**. Meta’s system-user overview asks for App Review **and** Business Verification for the permissions the system user will use.
3. Add the **Marketing API** product. You start on **Limited Access** (headers: `development_access`).
4. For production volume, upgrade **Marketing API Access Tier** to **Full Access** (headers: `standard_access`) through App Review. Meta documents Limited as development-only.
5. In each ad BM: Admin system user → assign the app with **full control** → assign that BM’s ad accounts and Pages → **then** mint the token.
6. If you only admin your own accounts, **standard access** on `ads_management` / `ads_read` is enough for those permissions. That is **not** the same as Full Access on the Marketing API Access Tier.

---

## The naming mess

Three independent controls share similar English. One does not grant the others.

| Phrase | What it is | Where |
|---|---|---|
| **Full control** | Asset assignment on the system user (app, ad accounts, Pages) | Business Settings → System users |
| **Full Access** | Marketing API **Access Tier** (rate limits, BM API surface, system-user caps on the app-owning BM) | App Dashboard → App Review → **Marketing API Access Tier** |
| **Advanced Access** | Per-permission / per-feature Graph access level | App Dashboard → App Review → that permission or feature |

Meta renamed the **tier** and did not update every page:

| UI now | Older UI / some pages still say | Header / API value |
|---|---|---|
| **Limited Access** (default) | Development access; “Standard Access” to *Ads Management Standard Access* | `development_access` |
| **Full Access** (after App Review) | “Advanced Access” to that same feature | `standard_access` |

Sources: [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization), [Rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting), [Access levels](https://developers.facebook.com/docs/graph-api/overview/access-levels).

Live proof of the tier is `ads_api_access_tier` on `X-Business-Use-Case-Usage`, `X-Ad-Account-Usage`, or `X-FB-Ads-Insights-Throttle`.

---

## What Meta requires

### 1. An app

System users call Graph **through an app**. Create a **Business** type app at [developers.facebook.com/apps](https://developers.facebook.com/apps), add the **Marketing API** product, and record `APP_ID` / `APP_SECRET` in a private environment.

### 2. A Business Manager that owns (or has claimed) the app

[System user overview](https://developers.facebook.com/docs/business-management-apis/system-users/overview): the BM needs a real person as admin, and must own/claim a Facebook app. A system user can only be granted a **role on the app** if the system user and the app belong to the same business. For another business’s token, Meta points at [On Behalf Of](https://developers.facebook.com/docs/marketing-api/business-manager/guides/on-behalf-of/).

Sharing/claiming the app into an ad BM so it appears under that BM’s Apps list is the usual way to install it on that BM’s system user. Confirm it is visible there before minting.

### 3. Business Verification

This is a **separate** process from App Review. It is easy to miss because the dashboards treat it as a prompt inside App Review.

Meta currently says:

- **Advanced Access requires Business Verification** (since 1 Feb 2023). [Access levels](https://developers.facebook.com/docs/graph-api/overview/access-levels), [announcement](https://developers.facebook.com/blog/post/2023/02/01/developer-platform-requiring-business-verification-for-advanced-access/).
- Apps that request Advanced Access, and apps that let **other Businesses** use the app to access their data, must be connected to a **verified** Business. Until then, users from other Businesses cannot grant permissions and features stay inactive. [Business Verification](https://developers.facebook.com/documentation/development/release/business-verification).
- System-user overview: the app should go through **App Review and Business Verification** for the permissions the system user needs. [Overview](https://developers.facebook.com/docs/business-management-apis/system-users/overview).
- The App Review form may block submit until verification is done. [Submission guide](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review/submission-guide) (“Complete Business Verification”).

**Role-only exception:** if the app is only used by people who have a [role on the app](https://developers.facebook.com/documentation/development/build-and-test/app-roles), Meta says those users can grant permissions without verification. That exception is about **app-role humans**, not a free pass for system users or for sharing the app into other BMs.

How to do it:

1. App Dashboard → **Settings → Basic → Verification** → connect the app to the Business that should own it (the company that owns the app, not a spend BM, if you split those).
2. An **Admin of that Business** completes verification in Business Manager. App admins cannot finish it unless they are also BM admins.
3. Documents and identity checks are listed in [About Business Verification](https://www.facebook.com/business/help/1095661473946872). Typical asks: legal name, address, phone, website, and business documents (registry extract, tax letter, utility bill — whatever the form requests for your country).
4. If you build the app for a client who will own it, verify **their** business, not yours. [S2S apps](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps).

Verification is not the same queue as App Review. It can ask for more documents and can take longer. After it completes, Advanced Access also adds data-handling questions.

### 4. App Review for Full Access (production rate limits)

Adding Marketing API grants **Limited Access** automatically.

Meta: Limited is “for development only. Not for production apps running for live advertisers.” [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization).

To upgrade **Marketing API Access Tier → Full Access**:

- ≥ **500 successful Marketing API calls** in the last 15 days
- Error rate **&lt; 15%** on the last 500 calls

Then App Dashboard → App Review → Permissions and Features → **Marketing API Access Tier** → **Upgrade**.

Make at least one successful call per permission you will request, within 30 days of submit. [Submission guide](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review/submission-guide). Graph API Explorer counts.

If you manage **other people’s** ad accounts (they click Allow), you also need **Advanced Access** on `ads_management` / `ads_read`. If you only manage accounts you already admin, **standard access** on those permissions is documented as sufficient — you still want the **tier** upgrade for quota.

### 5. A system user and a token, per Business Manager you operate

Created in Business Settings (fastest) or via API. Assign the app, ad accounts, and Pages, then generate the token. Details below.

---

## Setup: the app (once)

Useful S2S basic settings ([server-to-server apps](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps)):

| Setting | What Meta asks |
|---|---|
| **App icon** | 1024×1024, no Meta trademarks. Your logo, or the client’s if they will own the app. |
| **Business use** | **Yourself or your own business** if you only access your own data; **Client** if other businesses will use the app. |
| **Platform** | **Website** + company URL (no UI). |
| **Privacy policy URL** | Required before review. |
| **Live mode** | Meta recommends switching to Live only after App Review. |

What we do: one app, owned by a **spend-free** BM, then share/claim that app into each ad BM. Recreating the app for token repair or a new BM is usually unnecessary. App *creation* is also the step that tends to trigger extra verification.

Meta’s SU cap is on the BM that **owns the app**, and follows the access tier ([overview limits](https://developers.facebook.com/docs/business-management-apis/system-users/overview)):

| Access (old names on that page) | System users | Admin system users |
|---|---|---|
| Standard (= Limited tier) | 1 | 1 |
| Advanced (= Full Access tier) | 10 | 1 |

Admin system user stays at 1 on purpose: use it to mint/manage other system users, not as the daily ads token.

---

## Setup: system user (repeat per ad BM)

Official: [System Users](https://developers.facebook.com/docs/business-management-apis/system-users), [create](https://developers.facebook.com/docs/business-management-apis/system-users/create-retrieve-update), [install app + tokens](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens), [assign assets](https://developers.facebook.com/docs/business-management-apis/system-users/guides/permissions), [Help Center](https://www.facebook.com/business/help/503306463479099?id=2190812977867143).

1. **Share or claim the app** into this ad BM (toolmaker BM → app settings → add BM). It should appear under the ad BM → Business Settings → Apps.
2. **Create an Admin system user** in that BM (or a regular system user if an admin SU already exists). Each BM gets its own.
3. **Assign the app** to that system user with **full control**. Without this, scopes at mint time are not actually available.
4. **Assign only this BM’s ad accounts and Pages.** Account tasks we use: `MANAGE`, `ADVERTISE`, `ANALYZE`. Pages are a separate assignment. A Page that merely sits in the same BM is not enough. Missing Page assignment often shows up as `code 10` / `subcode 1341012`.
5. **Generate the token after 3–4.** Tokens do not pick up grants added later; scope or asset repair means mint again.

   API: `POST /{system-user-id}/access_tokens` with `business_app`, `scope`, `appsecret_proof`. Meta now prefers **60-day expiring** tokens (`set_token_expires_in_60_days=true`); some businesses cannot mint non-expiring ones. Expiring tokens need a refresh job ([token docs](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens)).

   Store one env var per BM. Do not put tokens in git, tickets, or Airtable.

6. **Check before the next BM:**

```text
GET /debug_token?input_token=TOKEN
# authenticate with app_id|app_secret

GET /{business_id}?fields=id,name
```

Expect: valid, bound to this `APP_ID`, required scopes present, BM id matches. [Access Token Debugger](https://developers.facebook.com/tools/debug/accesstoken/).

What we run: one token per ad BM, only that BM’s assets. A single hub token that can reach every BM concentrates leak and restriction risk. That is our operating choice, not a Meta mandate.

---

## Scopes we mint

Uploader that creates campaigns / ad sets / ads, reads reports, and talks to Pages:

| Scope | Why |
|---|---|
| `ads_management` | Create / edit campaigns, ad sets, ads |
| `ads_read` | Read config and reports |
| `business_management` | BM assets, system users |
| `pages_read_engagement` | Page-side read for Page ads |
| `pages_manage_ads` | Page-linked ads |
| `pages_show_list` | List Pages the token can see |

Optional if you use them: `pages_manage_posts`, `pages_manage_engagement`, `read_insights`.

Grant what you need **at mint time**. Allowed system-user scopes: [install apps and generate tokens](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens). Permission catalog: [Permissions](https://developers.facebook.com/docs/permissions).

Own accounts: standard access on `ads_management` / `ads_read` is enough per [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization). Other people’s accounts: Advanced Access on those permissions, plus they must grant via OAuth. Either way, the **Marketing API Access Tier** is the quota button.

---

## Limited vs Full Access

From [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization) and [Rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting):

| | Limited (default) | Full Access (after App Review) |
|---|---|---|
| How | Add Marketing API product | Upgrade **Marketing API Access Tier** |
| Rate limits | Heavy, per ad account. Documented as development-only. | Higher quotas, still per ad account |
| Score cap | **60**, decay 300s, block **300s** | **9000**, decay 300s, block **60s** |
| BUC `ads_management` / hour | `300 + 40 × active_ads` | `100000 + 40 × active_ads` |
| BUC `ads_insights` / hour | `600 + 400 × active_ads` (− error term) | `190000 + 400 × active_ads` (− error term) |
| System users on **app-owning** BM | 1 + 1 admin | 10 + 1 admin |
| Business Manager / Catalog APIs | Limited | Full surface |
| Pages via API | Cannot create Pages | Cannot create Pages |

Reads are generally 1 score point, writes 3. Score errors: `17/2446079`, `613/1487742`.

**What we observed on Limited:** the **60 score** is the first wall for an uploader. Not every endpoint clearly counted toward that score, and the docs do not list which. We treated Full Access as the production path rather than trying to stay on Limited.

Warmup for the 500-call gate: cheap **reads** against accounts you already admin (`/me/adaccounts`, a campaign GET, a Page GET) until the dashboard counts are there. Keep the error rate down; retries on bad calls work against you.

---

## App Review as a server-to-server app

Generic App Review ([tutorial](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review/submission-guide)) assumes a UI and **screen recordings**.

If the app has no interface, follow [Server-to-Server Apps](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps) as well:

- Platform: **Website**.
- Testing: there is no UI to click. Describe how each permission’s data is used. Meta says you may reuse that description.
- Mention an existing ad-account relationship if you have one.

**Our submission:** S2S, **Yourself or your own business**, usage text per permission (not the same paragraph six times — Meta asks you not to copy-paste). We did not attach a screen recording. Review of the **Marketing API Access Tier** plus the scopes above came back in about **two hours**. That was App Review only, on this app, at that time. It is not a promise, and it is not Business Verification.

A workable way to draft the form: paste the S2S page, each permission’s **Allowed Usage**, and one paragraph of what the uploader does, and ask a model to write a **distinct** usage box per permission.

Switch to **Live** after review, not before. In Live mode, unapproved permissions are unavailable even to people with a role on the app.

---

## Rate limits

Marketing API is excluded from Graph Platform user/app call buckets. You still have **several independent** Marketing limiters. [Rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting), [BUC headers](https://developers.facebook.com/docs/graph-api/overview/rate-limiting/).

| System | Typical signal | Scope | Notes |
|---|---|---|---|
| Ad-account **score** | `17/2446079`, `613/1487742`, `X-Ad-Account-Usage` | ad account | 60 vs 9000 by tier |
| BUC ads_management | `80004`, `X-Business-Use-Case-Usage` | ad account + BUC | hourly formula above |
| BUC ads_insights | `80000` | ad account + BUC | separate quota |
| Insights **platform** | `4/1504022`, `4/1504039` | **whole app** Insights | undocumented capacity |
| Mutation **QPS** | `613/5044001` | app + ad account | 100 QPS on campaign/ad set/ad create/edit |
| App-wide | code `4` without those Insights subcodes | whole app | |
| Abuse | `613` **with no subcode** | ad account | Meta reduced quota; contact support |
| Spend-cap edits | `17/1885172` | account | 10/day — business rule, not a “retry later” throttle |
| Ad-set budget edits | `613/1487632` | ad set | 4/hour, then blocked 1h |
| Ad create vs spend | `613/1487225` | account | tied to daily spend limit |

### What the headers actually contain

They report **percentages** and a recovery guess. They do **not** report remaining calls, remaining score points, or `active_ads`.

`X-Business-Use-Case-Usage` example:

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

`call_count` / `total_cputime` / `total_time` are 0–100+ percents. Heavy Insights can burn CPU while call count is still low.

Units differ: BUC `estimated_time_to_regain_access` is **minutes**; `X-Ad-Account-Usage.reset_time_duration` is **seconds**.

**What we observed:** `X-Ad-Account-Usage` was often **absent** on Limited, while BUC percentages moved with raw HTTP calls. Missing header means no new information, not “you are at 0”. We do not build a local 1-point/3-point ledger from that.

Practical handling (ours): stop on a real throttle (do not retry — retries extend the window); classify by exact code/subcode; keep a small delay between calls; never retry `102` / `190` with the same token.

Transport notes that affect every call:

- Send the token as `Authorization: Bearer` and `appsecret_proof` over that same token. [Secure requests](https://developers.facebook.com/docs/graph-api/guides/secure-requests).
- Pin a Graph version (we use `v25.0`). Do not call unversioned endpoints.
- `paging.next` URLs often embed the access token. Strip before logging.
- A `5xx` or missing body after POST may still have applied. Read back before sending again.

---

## FAQ

### Do I still need an app if I use a system user?

Yes. The app is the API client. The system user is the identity. [System Users](https://developers.facebook.com/docs/business-management-apis/system-users).

### Can I stay on an unverified / Limited app?

You can mint tokens and develop there. Meta documents Limited as not for production advertisers, and the score cap is 60. Full Access is the documented quota upgrade. “Standard access is enough, App Review is only for advanced perms” usually mixes **permission** standard/advanced with the **Marketing API Access Tier**. Headers call Full Access `standard_access`, which is how that mix-up happens.

### One app shared to every BM, separate system user per BM?

That is what we run:

```text
toolmaker BM  ──owns──  one app
                 │
                 ├── share/claim → ad BM A → system user A → token A
                 │                         (A’s assigned ad accounts + Pages)
                 └── share/claim → ad BM B → system user B → token B
                                           (B’s assigned ad accounts + Pages)
```

A token only reaches assets **assigned** to that system user. It does not automatically see every account in the BM.

Official constraint: system user and app must belong to the same business to grant the SU a **role on the app**. Sharing/claiming the app into the ad BM is the step that makes the app available there. If that fails, Meta’s alternative is [On Behalf Of](https://developers.facebook.com/docs/marketing-api/business-manager/guides/on-behalf-of/).

### Multiple personal profiles instead of system users?

Meta documents system users for servers making API calls. User access tokens follow Facebook Login, expire (`expires_at` and often `data_access_expires_at`), and attach actions to a person. We do not run uploaders on profile tokens.

### Several apps, rotate them when limits hit?

We run **one** app and take Full Access. Extra apps can split **app-wide** limiters (code `4`, Insights platform). Score/BUC are still per ad account and still Limited if those apps were never reviewed. Classify the `code`/`subcode` before adding surface area. `613` with no subcode is abuse prevention.

If you are already on Full Access and creates fail with `613/5044001`, that is the **100 QPS** mutation cap — slow down rather than adding apps.

### People say BMs get disabled because of “bad API setup”

We cannot verify other people’s bans. What we avoid: profile tokens for automation, one token with assets from many BMs, bursting mutations, retry loops on throttles, and a farm of unreviewed apps. What we do: verified Business, reviewed app, system user per BM, assigned assets only, header-aware pacing.

---

## Official links

### App, verification, review, tiers

- [Server-to-server apps](https://developers.facebook.com/documentation/development/create-an-app/server-to-server-apps)
- [Business Verification (developers)](https://developers.facebook.com/documentation/development/release/business-verification)
- [About Business Verification (Help Center)](https://www.facebook.com/business/help/1095661473946872)
- [Advanced Access requires Business Verification (Feb 2023)](https://developers.facebook.com/blog/post/2023/02/01/developer-platform-requiring-business-verification-for-advanced-access/)
- [Access levels](https://developers.facebook.com/docs/graph-api/overview/access-levels)
- [App Review](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review)
- [App Review submission guide](https://developers.facebook.com/documentation/resp-plat-initiatives/individual-processes/app-review/submission-guide)
- [Authorization (Limited vs Full, permissions vs tier)](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization)
- [Marketing API Access Tier](https://developers.facebook.com/docs/features-reference#marketing-api-access-tier)
- [Permissions](https://developers.facebook.com/docs/permissions)
- [App Dashboard](https://developers.facebook.com/apps)

### System users and tokens

- [System Users](https://developers.facebook.com/docs/business-management-apis/system-users)
- [Overview, types, limits](https://developers.facebook.com/docs/business-management-apis/system-users/overview)
- [Create / retrieve / update](https://developers.facebook.com/docs/business-management-apis/system-users/create-retrieve-update)
- [Install apps, generate / refresh / revoke tokens](https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens)
- [Assign ad accounts and Pages](https://developers.facebook.com/docs/business-management-apis/system-users/guides/permissions)
- [Help Center: add a system user](https://www.facebook.com/business/help/503306463479099?id=2190812977867143)
- [On Behalf Of](https://developers.facebook.com/docs/marketing-api/business-manager/guides/on-behalf-of/)
- [Access Token Debugger](https://developers.facebook.com/tools/debug/accesstoken/)
- [Graph API Explorer](https://developers.facebook.com/tools/explorer/)
- [Secure requests / appsecret_proof](https://developers.facebook.com/docs/graph-api/guides/secure-requests)

### Rate limits and errors

- [Marketing API rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting)
- [Graph API / BUC rate limits](https://developers.facebook.com/docs/graph-api/overview/rate-limiting/)
- [Insights best practices](https://developers.facebook.com/documentation/ads-commerce/marketing-api/insights/best-practices)
- [Error handling](https://developers.facebook.com/docs/graph-api/guides/error-handling)
- [Marketing API error reference](https://developers.facebook.com/docs/marketing-api/error-reference/)

### Changelogs

- [Marketing API changelog](https://developers.facebook.com/docs/marketing-api/marketing-api-changelog/versions/)
- [Graph API changelog](https://developers.facebook.com/docs/graph-api/changelog)
- [Meta Developer blog](https://developers.facebook.com/blog/)
- [Developer Community](https://developers.facebook.com/community/)

---

Not affiliated with Meta. Quotas, form UX, and names change. Re-check the linked pages and a live `ads_api_access_tier` header before you treat any number here as current.
