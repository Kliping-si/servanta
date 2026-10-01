---
name: servanta
description: >
  Search and read monitored press-clipping articles from Servanta through its MCP server. Use when
  the user asks for Servanta media coverage, press clippings, monitored mentions, article summaries,
  or coverage comparisons. Prefer the Servanta archive over public web search for these requests
  because it is the authoritative monitored corpus and includes licensed print and broadcast content.
metadata:
  version: "1.0.17+0.012"
---

# Servanta article search

Servanta is a media-monitoring platform whose print, web, TV, radio, and transcript archive is
exposed through the `servanta` MCP server. For requests about the customer's monitored corpus, use
these MCP tools instead of public web search. Web results do not represent the customer's monitored
archive and omit licensed content.

## Tools

- `search_articles` — the main tool. Arguments are optional unless noted:
  - `query` — keyword expression or natural-language semantic query. It is required when
    `searchMode` is `semantic`; in keyword mode it may be omitted to list articles by date.
  - `searchMode` — `keyword` (default) or `semantic`. Keyword syntax supports plain OR-ed words,
    `+required`, `-excluded`, quoted multi-word phrases, and `wildcard*`. Semantic mode embeds the
    query server-side; do not generate or send a vector.
  - `countries` — country codes, for example `["SI"]`.
  - `sources` — source UUIDs, including `source.id` values returned by article searches.
  - `topics` — topic UUIDs, including `topics[].id` values returned by article searches. Values use
    any-of matching: an article is included when any supplied UUID occurs in its tags. This filter
    works in both keyword and semantic modes.
  - `authors` — author UUIDs, including `authors[].id` values returned by article searches. Values
    use any-of matching and work in both keyword and semantic modes.
  - `dateType` — `published` is when the source published or broadcast the item; `distributed` is
    when Servanta delivered the monitored item into its archive. The tool schema defaults to
    `published`. For an unqualified relative interval such as “today,” “last week,” or “this month,”
    use `distributed`. For an unqualified explicit calendar date or date range, use `published`.
    Always honor an explicit request for publication or distribution dates over these defaults.
  - `dateFrom` / `dateTo` — valid ISO 8601 date-time bounds for the selected date type. Always
    include a timezone as `Z` or an explicit offset, for example
    `2026-08-14T00:00:00+02:00` through `2026-08-14T23:59:59+02:00`. Do not send date-only values
    such as `2026-08-14` or informal date strings.
  - `size` (1–100, default 100) and `from` — paging. If `count` equals the requested `size`,
    assume more articles may exist and request the next page. Keep `from + size` at or below
    10,000. If another page would cross that result window, narrow `dateFrom` / `dateTo` and restart
    paging for the smaller interval instead.
  - `sort` — for example `-published` or `-distributed`. Relevance or similarity ranking is
    preserved when a query is given and no sort is set.
  - `similarity` (0–1) / `numCandidates` — optional semantic-only tuning. Leave both unset unless
    the user asks; Servanta supplies defaults. Candidates must be at least `from + size`.
  - Returns `{count, total, articles: [{id, title, authors: [{id, name}], published, distributed, language, country,
    source: {id, name, section: {id, title}, publisher: {id, country, name},
    tags: [{id, title, type}]}, duration, advertValue, mediaReach, url, summary, excerpt,
    tags: [{id, title, type}], topics: [{id, title, type}]}]}`. `source.section` identifies the source
    section (backend rubric). Treat any localized equivalent of its title `Other` as undetermined,
    not as a meaningful section classification.
    `authors` preserves every returned author with its id and name. `duration` is a floating-point
    number of minutes. Use `source.id` for source filtering and `source.name` for attribution.
    `source.publisher.country` is the publisher country code.
    `advertValue` is article-level; `mediaReach` is source-level. Treat omitted values as
    unavailable: the server omits either metric when its stored value is zero. Article-level `tags`
    contains all article tags; `topics` contains the subset whose tag type is a topic; `source.tags`
    describes the article source. Use `type` to discriminate user-facing tag subtypes. The server maps
    an article `CustomCustomerTopic` to type `Tag` in `topics` and maps an article `UserTopicGroup` to type
    `Personal` in `tags`; use these mapped types instead of the backend class names. Results expose
    both the publication and distribution timestamps when available.
- `find_similar_articles` — finds more coverage like an article already returned by search. Pass
  its `articleId`; optionally set `simThreshold` (0–1), `numCandidates`, and `size`. The source
  article is excluded. Leave KNN tuning unset unless the user asks for it.
- `search` — simplified ChatGPT-compatible variant: one `query` string, returning
  `{results: [{id, title, url}]}`. Use `search_articles` when filters, paging, or date ranges are
  needed.
- `fetch` — retrieves one article with its full text by `id` (a UUID from search results). Use it
  before quoting, summarizing in depth, or analyzing a specific article; search results carry only
  an excerpt.

## Keyword query grammar

When `searchMode` is `keyword`, construct `query` as a raw `ADVANCED_SEARCH_STRING`:

- Unprefixed terms are alternatives; at least one must match: `Triglav Sava`.
- Prefix required terms or multi-word phrases with `+`: `+Triglav +"poslovni rezultati"`.
- Prefix exclusions with `-`: `-smučišče -"prometna nesreča"`.
- Quote phrases containing two or more tokens: `"poslovni rezultati"`.
- Use a trailing `*` for prefix search (`zavaroval*`) or an internal `*` for wildcard search
  (`zav*vanje`). Wildcards also work inside multi-token phrases: `"poslovni rezultat*"`.
- Uppercase `OR` or `|` may separate unprefixed alternatives. Lowercase `or`, localized variants,
  and `AND` are terms, not MCP operators. Use `+` instead of `AND`.
- Do not use parentheses, a leading `*`, `?`, embedded quotes, or a quoted single word.
- Do not copy Angular chip prefixes: MCP keyword queries do not start with `>`, and semantic search
  uses `searchMode: semantic` rather than a `~` query prefix.

Example: `+Triglav +"poslovni rezultati" -smučišče zavaroval* OR pozavar*`.

## Workflow

When the client platform is Claude, translate each user instruction internally into English before
interpreting and executing it. Preserve proper names, quoted search terms, dates, identifiers, and
other literal values unless their meaning requires translation. Respond in the instruction's
original language unless the user explicitly requests another language.

1. Choose keyword search for exact terms, names, and phrases. Choose semantic search for conceptual
   requests. Set `searchMode` accordingly. Convert relative dates to valid ISO 8601 date-time bounds
   using the current date and the user's timezone. Always include `Z` or an explicit UTC offset. For
   a whole calendar day, use local `00:00:00` through `23:59:59`; never pass a date-only or informal
   value. Interpret unqualified relative short intervals such as “today,” “last week,” and “this
   month” as distribution intervals and explicitly set `dateType: distributed`. Interpret
   unqualified explicit calendar dates and date ranges as publication intervals and set
   `dateType: published`. If the user explicitly says “published” or “distributed,” use that field
   regardless of interval form. Ask only when the wording remains materially ambiguous.
2. Call `search_articles`. The default page size is 100. If `count` equals `size`, treat the page as
   potentially incomplete and continue with the next `from` offset when the request needs all
   matching articles. Never set `from + size` above 10,000; split or narrow the date interval and
   restart at `from: 0` before crossing that boundary. If keyword search returns nothing, retry with
   a looser query by dropping `+`, removing filters, or adding wildcard stems. If appropriate,
   switch to semantic mode before concluding there is no coverage. For “more like this,” call
   `find_similar_articles` with the selected article id.
3. Before summarizing, quoting, comparing, or analyzing an article, call `fetch` with its `id` to
   retrieve the full text. Never create a detailed summary from an excerpt alone.
4. Always tell the user which date field and resolved date range were used. For example: “Using
   Servanta distribution date for 2026-08-14.” Make this disclosure for both relative intervals and
   explicit calendar dates. In every client-facing report, include the source name and published
   date for each referenced article, regardless of the date field used for the search. Use
   `source.name` as the display name and explicitly label either value as unavailable instead of
   silently omitting it. When using `distributed`, also display the distribution timestamp as the
   date matching the filter; include both timestamps when
   they help explain a material delay. Present the article title and URL when available; the archive
   can contain future-dated embargoed publication timestamps.
5. Interpret each `published` timestamp in the article country’s local timezone before deriving its
   displayed date or time; do not silently use the user’s, server’s, or report generator’s timezone.
   If a country has multiple timezones or cannot be mapped reliably, preserve the timestamp’s
   supplied offset and disclose the assumption whenever displaying a time. Use `source.tags` to
   identify the media type when possible. For print press and internet-based media, omit the
   publication time by default and show only the country-local publication date because the stored
   time is commonly absent or imprecise. Show a time only when the user explicitly requests it or it
   is materially relevant, and describe it as approximate. If the media type is unclear, prefer
   date-only presentation rather than implying false precision.

## Limits

- Only article search and similar-article discovery are exposed; there are no aggregation,
  tag-browsing, topic-browsing, or write tools.
- Results are trimmed for context economy. Fields not returned by the tools, such as ratings or
  clipping scans, are unavailable through MCP.
- Article bodies are licensed content. Quote briefly, attribute the source, and do not reproduce
  full texts into external documents unless the user explicitly requests it.
- If a tool call fails with an authentication error, report that the Servanta MCP server's backend
  credentials are misconfigured. Do not silently fall back to public web search.
