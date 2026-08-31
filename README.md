[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22206853.svg)](https://doi.org/10.5281/zenodo.22206853)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-blue.svg)](https://creativecommons.org/licenses/by/4.0/)

# Political party WhatsApp channels

An open dataset of European political parties' presence on WhatsApp — their
Channels, groups and click-to-chat numbers — with the public follower counts of
their Channels tracked over time and ranked by city, region, country and
continent.

It powers the ranking pages at
[wha.tools/political-parties-whatsapp-channels](https://wha.tools/political-parties-whatsapp-channels),
and is published here as a citeable, self-describing snapshot under CC BY 4.0.

## What this is

A WhatsApp Channel has a public preview page at
`https://www.whatsapp.com/channel/<id>` that needs no login and prints the
channel's name, its blurb and its follower count. There is no API — that page is
the entire data source. This dataset reads those pages on a schedule and records
what they say, so the follower count of a party's Channel can be followed over
time rather than seen once.

Around the follower readings sits a curated registry of parties (who exists,
where, and which WhatsApp surfaces are theirs) and the geography needed to
aggregate a city into a region into a country into a continent.

Everything here comes from a public page. Nothing is estimated, modelled or
back-filled. A failed read is recorded as a failure, not dropped and not
zeroed.

## What's in here

```
datasets/
  party-channels/
    parties/<iso2>.json     Registry: one file per country. Curated by people.
    parties/_europe.json    Registry: supranational parties with no country.
    profiles.json           Observed channel state: title, description, avatar,
                            latest count, verification badge. Overwritten each check.
    avatars/<id>.<ext>      Mirrored channel avatars, one file per surface.
    observations/<YYYY-MM>.csv  Append-only follower readings. One row per check.
  places/
    countries.json          European countries (ISO 3166-1).
    regions.json            Country subdivisions (ISO 3166-2).
datapackage.json            Frictionless descriptor for every file above.
```

Three kinds of data meet here with different lifecycles, so they live apart:

| Store | File | Written by | Lifecycle |
| --- | --- | --- | --- |
| Registry | `parties/<country>.json`, `parties/_<continent>.json` | people, in pull requests | rarely, deliberately |
| Profiles | `profiles.json` | the collector | overwritten each check |
| Avatars | `avatars/<id>.<ext>` | the collector | one file per surface, overwritten in place |
| Observations | `observations/<YYYY-MM>.csv` | the collector | append-only, forever |

Keeping them apart matters. A single spreadsheet holding all three breaks
immediately: a robot and a human end up writing the same rows, and overwriting a
follower count in place destroys the one thing worth having, which is the series.
The channel title is *observed*, not curated — so it lives in profiles; put it in
the registry and it goes stale the moment a party renames their channel, and you
can no longer tell a rename from a typo.

Because everything is committed to git rather than kept in a database, the git
history *is* the archive: every number carries the commit that added it, and a
bad collection run can be reverted like any other change.

## The follower readings (`observations/*.csv`)

One row per check of one surface. Files are partitioned by month; all months
share the same header. Rows are never edited or deleted.

| Column | Notes |
| --- | --- |
| `checked_at` | ISO-8601 UTC, second precision. |
| `party_id` | Registry party id (join key to `parties/*.json`). |
| `channel_id` | Surface id: channel id, group invite code, or phone number (join key to `profiles.json`). |
| `status` | `ok`, `not_found`, `no_count`, `http_error`, `network_error`, `not_supported`. |
| `followers` | Empty unless `status` is `ok`. Never `0` standing in for "unknown". |
| `precision` | `exact` or `rounded` (see the locale note below). |
| `locale` | The `Accept-Language` that produced the reading. |
| `channel_name` | The surface's own name at check time. |
| `http_status` | Response status, when there was one. |
| `kind` | `channel`, `group`, `community` or `account`. |

New columns go on the end; existing ones are never reordered or removed, because
rows written months ago still have to line up with the header.

## The registry entry (`parties/*.json`)

Each country file is an object with a `country` code and a `parties` array (the
`_<continent>.json` files carry `continent` instead of a country). One entry is
**one party in one place**, with at most one Channel and at most one other
surface — both optional. A party with no known WhatsApp presence is still a valid
entry.

| Field | Meaning |
| --- | --- |
| `id` | Stable unique id and the join key to years of observations. **Never renamed.** |
| `name` | The party or branch name in its own language. |
| `abbreviation` | Short form (e.g. `CDU`). |
| `englishName` | English name, where helpful. |
| `region` | ISO 3166-2 subdivision code. Required on a region- or city-tier entry. |
| `city` | Free text. The only free-text place field (there is no global municipality code list). |
| `continent` | Set on a supranational entry (e.g. `EU`); lives in `_europe.json`. |
| `worldwide` | `true` on a global entry (lives in `_world.json`). |
| `parentId` | The national party this is a branch of, so a branch's audience is never folded into the parent's ranking. |
| `website` | Official site. |
| `email` | Contact address, where known. |
| `epGroup` | The European Parliament group the party's MEPs sit in: `EPP`, `SD`, `RENEW`, `GREENS_EFA`, `ECR`, `PFE`, `ESN`, `LEFT`, `NI`. |
| `europarty` | The pan-European federation, for movements with no MEPs. |
| `orientation` | Broad position (`far-left` … `far-right`), **derived** from `epGroup`. An override requires `orientationSource`. |
| `orientationSource` | Citation required whenever `orientation` is set by hand. |
| `cadence` | How often to read this entry: `daily`, `weekly` (default), `monthly`, `quarterly`. |
| `verified` | ISO date a person last confirmed the entry. |
| `noChannel` | Dated flag meaning a person looked and the party runs no WhatsApp at all (distinct from an absent channel, which only means nobody has found one yet). |
| `note` | Free-text note. |
| `channel` | The party's WhatsApp Channel surface (the only kind with a follower count). |
| `other` | One non-Channel surface: a group, community or account. |

A surface (`channel` or `other`) is:

| Field | Meaning |
| --- | --- |
| `kind` | `channel`, `group`, `community` or `account`. |
| `url` | The public URL. |
| `id` | Channel id, group/community invite code, or phone number. |
| `added` | ISO date the surface was recorded. |
| `source` | Where the link was found (optional). |
| `personal` | On a `channel` only: `true` when the linked Channel is a candidate's personal one, not the party's official presence. |

### The place ladder

Every entry sits on exactly one rung of **world > continent > country > region >
city**. That rung is *derived* from the place fields it carries, not stored — a
stored tier could disagree with the fields describing it. The chain has to be
unbroken: a city entry carries its region, a region entry its country. Places are
referenced by ISO code, never by typed-in name, so two contributors cannot spell
a region two ways and split its ranking in half.

## Geography (`places/*.json`)

`countries.json` lists European countries with ISO 3166-1 alpha-2 codes, names,
slugs and capitals. `regions.json` lists their subdivisions with ISO 3166-2
codes. These are generated reference tables used to validate and resolve the
codes an entry carries; they are not a list of pages.

## How the follower counts are collected

The collector reads each Channel's public preview page and records what the page
prints. Two things about that page shape the whole method.

**The count is rounded, and how hard depends on the language you ask for.** In
English the page says `7.6K followers`; in German it says `7.648`. Only German
and Italian print the whole number — everything else rounds to two significant
figures, which cannot tell 7,551 from 7,649. So the collector asks for `de-DE`
and falls back to `it-IT`, and every row records the `precision` it got. If a
reading is only available rounded, it is marked `rounded` rather than silently
losing precision.

**Only Channels publish a number.** A group or community invite page does not say
how many members it has, and a `wa.me` link is just a redirect into the app. So
rankings are built on Channels; other surfaces are recorded for completeness (so
a party whose only presence is a group still reads as "on WhatsApp") but never
carry a follower count.

Each entry has its own cadence, and reads are staggered so the whole dataset is
not fetched in one burst. The collector is deliberately gentle: a few requests at
a time, spaced out, with backoff, stopping early when rate-limited. A short run
is a gap, not a corruption — every row is independent.

### What the dataset refuses to publish

The honesty rules are the point of the design, not an afterthought:

- **A failed check is recorded as a failure**, with a status and no number. A gap
  that says `http_error` is honest; a silently missing row makes a rate-limited
  afternoon indistinguishable from a channel nobody tracked.
- **A count is never stored as `0`** to mean "could not read". In a time series
  those are worlds apart.
- **An unknown magnitude suffix is refused, not guessed.** `60 B` is 60 billion
  in English and 60 thousand in Turkish.
- **A party with no reading is reported as unmeasured**, never shown as zero.
- **An implausible jump is withheld** — more than fivefold up, or more than half
  the audience gone inside a day, is far more likely a parser regression than a
  fact.
- **A stretched comparison reports its real span.** With a quarterly cadence the
  baseline is rarely exactly 90 days old, so a change is reported against the
  actual gap rather than an implied precision.

## Scope: political parties only

The dataset holds **political parties and their territorial branches**. A youth
wing, a parliamentary group, a campaign, a candidate's personal organisation and
an activist movement are all out of scope: they are a different kind of thing,
and ranking them beside parties compares audiences that are not comparable.

There is deliberately no field recording what kind of organisation an entry is —
if none of those things belong in the data, none of them need a label. The rule
is applied where entries come in (research, review, pull request), not by the
schema. The one exception is already handled: a channel may be flagged `personal`
when a party's only WhatsApp presence is its lead candidate's own Channel, so the
two are never conflated.

## Methodology and limitations

- **Coverage is not exhaustive.** The dataset records who runs a Channel *and who
  does not*, but the bottleneck is discovery: measuring a channel is one request;
  finding the official one is a person's judgement. A party shown with no channel
  may simply have one nobody has found yet — unless it carries a dated
  `noChannel` flag, which means a person looked and there is none.
- **A channel announced only on Instagram or in a press article is invisible**
  to the automated crawler, which only follows parties' own websites.
- **Follower count is a public vanity metric.** It is what the Channel's own page
  advertises — not verified membership, reach, or engagement.
- **Orientation is derived, not asserted.** It comes from European Parliament
  group membership, a checkable fact, rather than an editorial judgement made
  from nowhere; a hand override must carry a source.
- **Geography currently covers Europe.** The place tables and the registry are
  European; the schema generalises, but the coverage does not yet.
- **Rounded readings happen.** If Meta changes the page's number formatting, rows
  come back marked `rounded` rather than losing precision quietly.

## Terms

These are public pages, but they are Meta's pages. `whatsapp.com/robots.txt` does
not disallow `/channel/`, and the collector identifies as a normal browser and
reads each page a few times a year. That file also carries a header notice that
automated collection of Facebook data requires written permission, which is a
terms-of-service question rather than a technical control — worth a decision
before scaling this to thousands of channels or building a commercial product on
it. The mitigations that matter are in the collection method already: low
frequency, small concurrency, backoff, and stopping early when rate-limited. If
WhatsApp starts refusing the requests, the failure is visible rather than silent
— rows are recorded with `http_error` and the run reports it.

## How to cite

Please cite the dataset if you use it. Cite the **concept DOI**
(Zenodo's "Cite all versions"), which always resolves to the newest version:

> WhaTools (2026). *Political party WhatsApp channels: an open dataset of parties'
> WhatsApp presence across Europe* [Data set]. Zenodo.
> https://doi.org/10.5281/zenodo.22206853

To cite the exact snapshot you used, use the version DOI Zenodo shows for that
release instead (v0.1.0 is `10.5281/zenodo.22206854`).

`CITATION.cff` in this repository carries the same metadata, so GitHub's "Cite
this repository" button produces a ready-made citation, and reference managers
can import it directly.

## License

This dataset is licensed under the
[Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/)
(CC BY 4.0). You may share and adapt it for any purpose, including commercially,
as long as you give appropriate credit. A suggested attribution:

> Political party WhatsApp channels — WhaTools (https://wha.tools), CC BY 4.0.

The full license text is in [`LICENSE`](LICENSE).

## How this dataset is maintained

This repository is a **published mirror**. The dataset is produced and curated in
the source project that also runs [wha.tools](https://wha.tools), and synced here
on a weekly schedule, so the files here are read-only copies — edits made here
would be overwritten on the next sync.

Corrections and additions are welcome through the "propose an update" links on
each party page, or the [contact form](https://wha.tools/contact) on the site.
