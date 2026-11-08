# UO Is Not Dead!

An everchanging list of Ultima Online freeshards. The deliverable is the shard table in `readme.md`.

## Repo map

- `readme.md` — the shard table, plus the directory-site list under "Where to find more shards"
- `docs/update-shard-list.md` — the full workflow for growing the shard table
- `contributing.md` — human-facing rules; the table rules here mirror it
- `.github/workflows/validate-external-links.yml` — lychee link check, weekly and on every PR touching `readme.md`
- `.github/workflows/validate-table.yml` — runs the checks below in CI, on every push to `master` and every PR touching `readme.md`
- `lychee.toml` / `.lycheeignore` — link-check config; `.lycheeignore` lists sites that block datacenter IPs

## Updating the shard list

Follow `docs/update-shard-list.md`. The directory sites to scrape are exactly the ones listed under "Where to find more shards" in `readme.md`; never hardcode that list, it changes over time.

## Table rules

- One row per shard: `| [Name](final-url) | :flag: |`. Link the final redirect destination.
- Sort: name lowercased with spaces and punctuation removed, ordered by UTF-8 bytes (`LC_ALL=C`); non-Latin names land last. The sort check below is the definition of this order.
- Pad cells to display columns (CJK characters count 2) so every pipe aligns. The padded cell is the whole `[Name](url)` link; the flag column is sized by the header cell. Widen the column only when a new row needs it, never shrink it, and keep the header row plain text.
- Flags: en `:gb:`, de `:de:`, es `:es:`, fr `:fr:`, it `:it:`, pl `:poland:`, pt `:brazil:`, ru `:ru:`, tr `:tr:`, cs `:czech_republic:`, lv `:latvia:`, zh `:taiwan:`. A shard with several languages gets several flags; ambiguous means `:gb:`.

## Verification

Run after editing `readme.md`:

```sh
# Row count: growth must equal the shards added
grep -c '^| \[' readme.md

# Sort order: prints nothing when correct, fails loudly otherwise
LC_ALL=C sh -c "sed -n '/^| \[/p' readme.md | sed 's/^| \[\([^]]*\)\](.*\$/\1/' \
  | tr -d '[:space:][:punct:]' | tr 'A-Z' 'a-z' | LC_ALL=C sort -c" && echo SORT-OK

# Pipe alignment: must print exactly one display width
python3 -c "import unicodedata;print({sum(2 if unicodedata.east_asian_width(c) in 'WF' else 1 for c in l.rstrip()) for l in open('readme.md',encoding='utf-8') if l.startswith('| ')})"

# Duplicate domains: review each hit. Two shards on one site are legal
# (uogamers.com, uorealms.com), a re-added shard is not
sed -n 's/^| \[[^]]*\](https\?:\/\/\([^/)]*\).*/\1/p' readme.md \
  | tr 'A-Z' 'a-z' | sed 's/^www\.//' | sort | uniq -d

# Spot-check new URLs with GET, not HEAD: some shards answer 403 to HEAD
curl -sL --max-time 30 -A "<browser user-agent>" -o /dev/null -w '%{http_code}\n' <url>
```

## CI link checks

- A PR touching `readme.md` runs lychee; `lychee.toml` already accepts 403/429 as alive.
- If a live shard fails only from datacenter IPs, add its domain to `.lycheeignore` with a dated comment matching the existing entries. Do not remove the shard for that.

## Hard rules

- Accuracy over volume: a missed shard beats a dead one.
- The list holds playable freeshards only: no official UO, no client tools, no non-UO content.
- This workflow only adds rows. The weekly link check reports dead shards on its own; report any you notice too, but do not remove rows unless asked.
- Do not commit or push unless asked.
