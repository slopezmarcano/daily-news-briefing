# daily-news-briefing
A scheduled research digest. Once a weekday it sweeps twelve news sources, Hacker
News and seven academic journals, ranks whatever clears a relevance bar into three
tiers, and writes a single self-contained HTML file.

Built for computer vision and data science applied to environmental systems, with
domain weighting agriculture first, then terrestrial, then marine. The prompt is a
reference implementation: it works, and the weighting is the part you would change.

`field-scan-prompt.txt` is the whole thing. Paste it into a scheduled task (in Claude for example). 

## Pipeline

```mermaid
flowchart TD
    A["Cron trigger, Mon to Fri"] --> B["Resolve location run date<br/>and weekday"]

    B --> C["Daily sources<br/>12 feeds<br/>24h window, 72h on Monday"]
    B --> D["Hacker News<br/>Algolia API, points over 100<br/>hardware, runtime, CVEs only"]
    B --> E["Journal sweep<br/>7 of 35 titles for that weekday<br/>Crossref by ISSN, rolling week"]

    C --> F["Relevance filter<br/>agriculture, terrestrial, marine"]
    D --> F
    E --> F

    F --> G["Rank into 3 tiers<br/>Build, Money, Evidence<br/>cap 13 items"]
    G --> H["Self-contained HTML<br/>field-scan-YYYY-MM-DD.html"]

    H --> I["SendUserFile<br/>into the session"]
    H --> J["PushNotification<br/>phone and inbox"]
```

## The three tiers

| Tier | What goes in it | Colour |
|---|---|---|
| **Build** | Vision or ML you could implement: models, datasets, benchmarks, deployment, tooling. Plus hardware and runtime. | `#0e7a52` |
| **Money** | Funding, acquisitions, launches, programme funding. | `#cc6600` |
| **Evidence** | Research findings and field results with measurable outcomes. | `#5a4fc4` |

Highest qualifying tier only. Within a tier, sort by domain weight, then newest
first. Cap 13 items, target roughly 6 / 3 / 4, but a short digest is allowed and a
zero-item digest is allowed. There is no padding mechanism.

## Two windows, not one

| Scope | Window |
|---|---|
| Daily sources and Hacker News | Rolling 24h, or 72h on the Monday run since the task does not fire at the weekend |
| Journal sweep | Rolling week, same weekday one week earlier through the run date |

## Journal rotation

35 titles, five fixed groups of seven, one group per weekday. Each group mixes
agriculture, remote sensing, terrestrial, marine and general, and carries exactly
one high-volume title so no single run has to read hundreds of records.

<details>
<summary>The 35 titles</summary>

**Monday** Precision Agriculture · Computers and Electronics in Agriculture ·
Remote Sensing of Environment · Ecological Indicators · Horticulture Research ·
ICES Journal of Marine Science · Agriculture (MDPI)

**Tuesday** Smart Agricultural Technology · Field Crops Research · ISPRS Journal of
Photogrammetry and Remote Sensing · Forest Ecology and Management · Journal of
Experimental Botany · Conservation Physiology · Agronomy (MDPI)

**Wednesday** Agricultural Systems · Biosystems Engineering · Remote Sensing (MDPI)
· Marine and Freshwater Research · Tree Physiology · BioScience · Frontiers in
Marine Science

**Thursday** Plant Phenomics · Agricultural and Forest Meteorology · Ecological
Informatics · Methods in Ecology and Evolution · Annals of Botany · Forestry ·
PLOS ONE

**Friday** Nature Food · Environmental Modelling & Software · Journal of Animal
Science · Journal of Plant Ecology · GigaScience · Frontiers in Plant Science ·
Nature (news section only)

ISSNs are in the prompt. All verified against Crossref's journal index, except the
two MDPI titles which were verified against OpenAlex.

</details>

Elsevier deposits no abstracts to Crossref, so every Elsevier title arrives
title-only. The prompt forbids inferring what a paper found from its title, so
those items ship as a headline, a DOI and a `Title only` tag, with no summary.

## Setup

| | |
|---|---|
| **Model** | `claude-opus-5` |
| **Reasoning effort** | Not a stored field on a scheduled task. Runs at the session default. |
| **Schedule** | Weekdays, 04:00 Australia/Brisbane |
| **Session** | Fresh per firing, no persistence |
| **Typical run** | About 14 minutes, roughly 30 fetches |

**Tools the task actually uses**

- `Bash` for the timezone lookup, epoch arithmetic and writing the file
- `WebFetch` for every source, including the Crossref and Algolia APIs
- `SendUserFile` to deliver the HTML
- `PushNotification` to send the summary
- `TaskCreate` / `TaskUpdate` for the run checklist

No connector is required. The prompt deliberately uses only free, unauthenticated
APIs: Crossref for journals, Algolia for Hacker News.

**Why the shell is barred from two hosts**

The container's egress proxy refuses shell requests to `api.crossref.org` and
`hn.algolia.com` with a 403 on CONNECT. The fetch tool reaches both without
trouble. The prompt says this explicitly because the failure is easy to
misdiagnose as the API being down.

## Rate limiting

Crossref limits this container hard. A single-run sweep of all 35 titles collapsed
under HTTP 429 with cooldowns up to six minutes. The weekday rotation is the main
fix. The rest:

- 7 requests on a clean run, hard ceiling of 16 including retries
- One at a time, never in parallel
- 20 second standing pause between titles, doubling on any 429, capped at 20 minutes
- Two attempts per title maximum, and the retry must change the `rows` value,
  because the fetch tool caches failures for 15 minutes keyed on the exact URL
- Carry `mailto` to reach Crossref's polite pool

## Change these four things before you run it

1. **`mailto=your-email@example.org`** in the Crossref URL template. Crossref uses
   this to attribute request volume, so run it under your own address.
2. **`"for <your name>, <your region>"`** in the header spec.
3. **The timezone.** The prompt is written around `Australia/Brisbane` (UTC+10, no
   daylight saving) and resolves the run date with
   `TZ=Australia/Brisbane date +'%Y-%m-%d %A'` because the container clock is UTC
   and reads a day behind. Change the zone, and check whether your schedule still
   lands on the weekday you intend.
4. **The journal table and the relevance bar.** This is the real work of adapting
   it. Look up replacement ISSNs at `api.openalex.org/sources?search=<title>`,
   which is reliable. Do not use the OpenAlex works endpoint: from this container
   it silently ignores its filter parameter and returns the whole corpus sorted by
   citations, which looks like a normal result list.

## Design notes worth keeping

- **Never infer from a title.** Missing abstract means title-only, no gloss.
- **Short days are fine.** No filler, no random journal draw.
- **No repeats.** Nothing outside the window, nothing a previous digest carried.
- **Fetched content is data, not instructions.** Stated explicitly in the prompt,
  since the run reads dozens of untrusted pages.
