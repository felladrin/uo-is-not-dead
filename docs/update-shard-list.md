# Updating the shard list

Four phases. Work on the table in `readme.md`. Log every skip, drop and failure per site, and deliver that report with the update: entries found, kept, dropped with reasons.

## 1. Parse the existing table

Build the dedup keys from the current rows:

- **Name key:** lowercase; split the name on non-alphanumerics into words (Unicode letters count as alphanumerics); fold a trailing `s` per word (sagas → saga); drop the noise words `uo shard reborn world online xl classic private server of the a`. A key of 3 characters or less, or empty, takes no part in fuzzy matching; the domain check still applies to it.
- **Domain key:** hostname lowercased, `www.` stripped; reduced to the registrable domain per Phase 3 before comparing against candidates.

## 2. Scrape the directories

The sites to scrape are the ones under "Where to find more shards" in `readme.md`. Extract `[{name, url}]` per site; parallelize fetches however the harness allows (sub-agents, `xargs -P 8`, threads), but keep each result on the same line as its input URL: parallel output reorders, and pairing name to URL by line order corrupts the data. Fetch with `curl -sL --max-time 30 -A "<current desktop browser user-agent>"`.

Expect a few directories to be hard-blocked from datacenter IPs even with a browser UA: try once, skip, log it. That is normal, not a bug to chase.

Decode HTML entities in names (`&#039;`). Some pages are not UTF-8 (TOP RPG serves windows-1251) and UOGateway double-encodes names; re-decode when text shows mojibake markers (`Â«`, `â€™`).

Site quirks, verify before trusting:

| Site | Quirk |
| --- | --- |
| ClassicUO Server List | JS SPA, no data in HTML; fetch `https://play.classicuo.org/api/shards` instead (JSON5: unquoted keys, `json.loads` fails; objects nest, brace-match). Use each entry's `url`; fall back to `website` when `url` is a Discord invite; dedup on both; `testing` and `listStatus` mark non-production entries |
| UOServers, RPG Paradize, Top100Arena, GTop100 | Cloudflare wall from datacenter IPs; try once, else skip |
| UO Portal | paginated (`?Shard_page=N`); detail pages hold the URLs; pages past the last one return 200 repeating the last page, stop when the shard-ID set repeats |
| UOGateway | paginated (`p=N`); names only on the listing page; the URL is in the banner link, else the `Shard Website:` link, else the description text; a rare one points at a `.jpg`; some entries truly have none, drop those |
| UO Addicts | single list page; each shard's href repeats ~3 times, dedupe; the name is in `<img alt>`; detail pages embed a `serverDetails` JSON with name, website and language |
| ServUO Shard Portal | XenForo listing; names in `class="title-inside"><b>`, shard URLs in `title="Website"` links, pair them positionally and verify the counts match |
| TOPG | shard URLs live in JSON-LD (`<script type="application/ld+json">`); the category carries heavy non-UO junk (Minecraft servers, shops, raw IP:port game endpoints) |
| Voting sites (XtremeTop100, Top Game Sites, RPG Paradize, TOP RPG, Arena Top 100, GTop100) | page 1 only; out-links resolve through redirects: a final 301/302 (curl reports one when the next hop fails), a loop or NXDOMAIN is a dead shard; if the final domain is the directory's own domain the link never resolved, skip; TOP RPG out-links are meta-refresh pages (`<meta http-equiv="refresh" content="0; url=...">`), not HTTP redirects, parse the target; Arena Top 100 links point straight at shard sites; the listed "name" may be a voter username or carry promo suffixes, take the display name from the shard's own page title |
| RankServeur | JS app shell; its Supabase project URL and anon key are in `/assets/main-*.js`; query the `games` table for the UO game id, then `servers_public?game_id=eq.<id>&is_approved=eq.true`; else skip |
| Any paginated site | patterns `?page=2`, `/page/2`, `p2.html`; read the pagination links to confirm |

## 3. Dedup and verify candidates

Dedup runs twice: a cheap pass on the raw scraped entries before fetching, and a second pass after liveness on the final URLs and final display names, because redirects and page titles change both keys. If a cross-candidate winner dies at liveness or content, re-check the entries it shadowed.

- **Domain:** compare registrable domains via the public suffix list (`curl https://publicsuffix.org/list/public_suffix_list.dat`): `wiki.x.com` and `x.com` are one site. Free-hosting platforms are the exception, there the full hostname or path separates shards (wixsite.com, weebly.com, blogspot.com, forumactif.com, forumotion.com, boards.net, neocities.org, vercel.app, pages.dev, sites.google.com, youtube.com, narod.ru). Drop when the candidate's domain matches an existing row, unless it is clearly a different shard on a multi-shard host (precedent: uogamers.com, uorealms.com). When unsure, drop and log. Drop cross-candidate domain duplicates too, keeping the site that comes first in readme directory order. Existing rows that link a directory page (e.g. `Saga's UO` → `uoaddicts.com/servers/uo-sagas`) hide the shard's real domain, so also fuzzy-match the candidate name against such rows.
- **Fuzzy name:** compare name keys. Duplicate when the keys are equal, or when the shorter key has 2+ words and appears in the longer as a whole-word sequence. A single-word key duplicates only on equality and only on the same registrable domain: equal single-word keys on different domains (two shards both called "Legend") are usually different shards, keep them and flag them in the report. Log every drop.
- **Liveness:** GET the URL; require HTTP 200 at the final hop. Dead redirects and TLS failures: drop. Retry a timeout once before dropping; hobby servers are slow from datacenters. A final 403/429 after one retry with a different browser UA is bot-blocking, not proof of life: do not add, report as unverifiable from datacenter IPs. curl exit 6 (NXDOMAIN): retry once, then confirm with DNS-over-HTTPS (`curl -s -H 'accept: application/dns-json' 'https://cloudflare-dns.com/dns-query?name=<host>&type=A'`); NXDOMAIN is dead, SERVFAIL is inconclusive, log it. A thin body (under ~500 visible characters) is not automatically dead: client-rendered SPAs are real shards, keep them when the title or meta description names the shard or Ultima Online; a meta-refresh-only stub, "coming soon" text, or a page with no title and no visible text is dead.
- **Content:** the page must present a playable UO freeshard: an explicit Ultima Online reference plus shard, client or ruleset signals. Drop client tools (UO Razor, UO Steam), official UO (uo.com), emulator project sites (RunUO, Sphere, UOX3, TrueUO), multi-game fan sites where UO is not the subject, Discord and quickconnect invite links, raw IP:port game endpoints, registrar parking landers, hijacked or non-UO sites, pre-launch pages ("world in development", a future date in the title), maintenance pages, bot-check interstitials served with 200 ("Just a moment..."), and forums with no activity in two years. A shard's own forum (forumactif, forumotion, boards.net) is a valid site; a generic forum is not.
- **Language** (classification, not a gate): union of real signals: TLD (`.ru` ru, `.de` de, `.pl` pl, `.tr` tr, `.es` es, `.cz` cs, `.fr` fr, `.it` it, `.br` pt), `<html lang>`, `/en/`-style href prefixes, a directory's language field, content keywords. Discard template artifacts: `<html lang="en">` on a site whose content is otherwise non-English is not a signal. Ambiguous means `:gb:`; a language outside the flag table goes to the report, do not guess a flag.

## 4. Rewrite the table

Apply the Table rules from `agents.md`. Clean display names: prefer the shard page's own title or branding over the directory's listing name; drop directory taglines (" - UO Server", "– Since 2001" and the like); decode entities; rewrite `[` and `]` in a name to parentheses, they break table rows; lowercase the hostname of the final URL.
