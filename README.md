# Web Scraping for LLM Pipelines: Markdown or Raw HTML, How Many Tokens per Page, and When a Managed API Beats Playwright (Plan Table + Chunking Rules)

A price tracker needs one number off the page. A model needs the page's meaning: which line was the heading, which table compared the plans, which block was code that must not be reworded. That difference decides your output format, your chunk size, and whether running Playwright yourself is worth the maintenance. The rules for web scraping for LLM consumption start with structure, not volume, and everything below follows from that.

This guide covers the five pipeline stages, the six input formats you can hand an extraction model, where the token bill actually comes from, the failure modes that show up in self-hosted stacks, and how ScraperAPI's credit pricing works if you would rather not maintain proxies and retry logic.

> **Quick take:** start on the free plan (1,000 credits/month, 5 concurrent connections, plus 5,000 credits for your first 7 days) to price out your target sites, then move to Hobby at **$49/month for 100,000 credits** once the pipeline runs on a schedule. Stay self-hosted if you scrape a few static documentation sites and can babysit a headless browser.
>
> 👉 [Compare every ScraperAPI plan and current credit allowance](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans at a glance

ScraperAPI's ladder runs Free, Hobby, Startup, Business, Scaling, Professional, Advanced, Enterprise. Three allowances are documented in ScraperAPI's own pricing notes: Free at 1,000 credits, Hobby at 100,000 credits for $49/month, and Scaling at 5,000,000 credits. The middle and upper tiers scale credits and price together; I confirmed the policy differences below but not the price of every tier, so those rows link to the live page rather than carry a number I could not verify.

| Plan | Monthly credits | Price | What changes at this tier | Get started |
| --- | --- | --- | --- | --- |
| Free | 1,000 (5,000 in first 7 days) | $0 | 5 concurrent connections, enough to cost-test a target list | [Start free with 1,000 credits](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | $49/mo | Auto-upgrade available; 20,000 Amazon pages/mo at 5 credits each | [Open Hobby at $49/month](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Scales with tier | Live pricing page | Auto-upgrade; account gets a 429 rather than a charge when credits run out | [Compare Startup against Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Scales with tier | Live pricing page | Same auto-upgrade and 429 behaviour as Startup | [See the Business tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | Live pricing page | First tier with pay-as-you-go past the monthly cap | [See Scaling and pay-as-you-go](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Scales with tier | Live pricing page | Pay-as-you-go plus auto-upgrade | [Compare Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Scales with tier | Live pricing page | Pay-as-you-go, limits set with sales | [See Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom quote | Negotiated rates; Ultra Premium requests carry their own minimum rate | [Request Enterprise pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two billing details matter more than the tier names. Unused credits are lost at each renewal instead of rolling over, and annual billing is exactly 10% cheaper than monthly while giving you the full year's credits upfront. Cancelling is free at any time, and there is a 7-day no-questions-asked refund window.

## What a page actually costs in credits

A credit is not a flat unit. ScraperAPI prices requests by how hard the target is:

| Target type | Credits per request |
| --- | --- |
| Standard public page | 1 |
| Amazon (all domains) | 5 |
| Google and Bing, including subdomains | 25 |
| LinkedIn | 30 |
| Anti-bot bypass activated (Cloudflare, DataDome, PerimeterX) | +10 on top of base cost |

For RAG work this table is the whole budget. Vendor docs and changelogs on plain hosting sit at 1 credit, so 100,000 credits is roughly 100,000 pages a month. The same pages behind Cloudflare cost 11 each, which turns that same allowance into about 9,000 pages. A support bot that refreshes 5,000 documentation pages weekly needs 5,000 credits in the clean case and 55,000 in the protected case. Amazon at 5 credits gives you 20,000 product pages on Hobby; on Scaling the 5,000,000-credit allowance covers 1,000,000 of them. You can check any single URL's real cost with the Domain Cost Estimator in the dashboard, and `max_cost` caps what one request is allowed to spend.

## Why a scraper built for humans breaks on a model

Traditional scraping targets were humans or databases, and both tolerate noise. A model does not. Three principles carry the whole design.

**Structure beats volume.** One clean article is worth more than a hundred raw HTML dumps, because the dump costs tokens on navigation bars, cookie banners and inline scripts that carry zero answer value.

**Semantics have to survive extraction.** Headings tell the model that a section follows; tables tell it that rows are being compared; code blocks must not be reworded into prose. A converter that flattens everything into a text blob destroys the cues the model needs.

**The pipeline is iterative.** Sources change. A system that ingests once and never re-checks will answer from stale content within a quarter.

Production setups follow the same five stages: discovery, fetch, render, extract, store. Static fetching comes first, and a headless browser only runs when the content genuinely needs JavaScript. The store is usually markdown, JSON, or chunked embeddings in a vector database.

## The six input formats, and which one to pick

Libraries like llm-scraper expose six formats: `html` for pre-processed HTML you already cleaned, `raw_html` for the untouched source, `markdown` for converted content, `text` for Readability-style article extraction, `image` for a screenshot, and `custom` for anything you precompute yourself.

The decision rules that hold up:

- **Markdown for articles, docs, changelogs.** Headings, lists, links and tables survive, and the token count drops once scripts, styles and navigation are stripped.
- **`text` when you want only the article body** and the page has heavy chrome around it. Readability throws away more, which is what you want for a news pipeline and not what you want for a page whose sidebar holds the data.
- **`html` when you have already sanitised the DOM yourself.** Handing raw markup to a model and asking it to ignore the noise is the most expensive way to solve a cleaning problem.
- **`image` when layout is the information.** Pricing tables with merged cells, dashboards, canvas-rendered charts. This requires a multimodal model and burns vision tokens, so reserve it for pages where the DOM lies.
- **`raw_html` for debugging a bad extraction.** When markdown output is missing a table, run the same page as raw HTML to see whether the converter or the prompt is at fault.
- **`custom` for API responses, PDF-to-text output, or anything you assemble before the model sees it.**

## Where the token bill comes from

Chunking is the line item most teams underestimate. One open-source pipeline splits converted markdown with tiktoken at **1,500 tokens for local models and 6,000 for remote ones**, cutting on line boundaries so a chunk does not end mid-sentence. The local figure is smaller because 7B to 14B models have tighter context windows, not because the content deserves different treatment.

Two rules make chunking behave. Split by semantics, keeping headings, paragraphs and tables intact and carrying section metadata into the vector store. Arbitrary character-count splits wreck retrieval accuracy, because the chunk that contains the answer no longer contains the heading that identifies it. Then measure retrieval quality by answer accuracy rather than page count; a pipeline that ingested 50,000 pages and answers nothing correctly is a failed pipeline with good metrics.

## Self-hosting: the stack and the five failures you will hit

A working open-source stack looks like this: Playwright with stealth fingerprint randomisation for fetching, a converter that strips script, style, nav and ad tags before producing markdown, tiktoken for chunking, and Instructor plus Pydantic to force the model's output into a schema at the API boundary. Temperature sits at 0.0 to cut hallucinated fields, and `max_retries=3` handles the occasional malformed response.

The failure list is short and repetitive:

1. **Empty CSV or missing rows.** The system prompt is too vague, the chunk size is too small, or the page never contained the data.
2. **Invalid JSON.** The prompt needs to demand JSON only, with no fences and no commentary, plus schema validation on the way out.
3. **403 responses.** Anti-detection is off, no proxy is configured, or requests are too fast.
4. **`Connection refused` from a local model.** The runtime is not running on its default port.
5. **`TimeoutError` on navigation.** `networkidle` never settles on pages with polling scripts; switching to `wait_until="load"` usually clears it.

There is also a cost trick worth knowing. Instead of sending every page through a model, you can have the model generate a reusable Playwright script once and evaluate it against the page, which amortises the inference cost across every subsequent run. That mirrors the direction research has taken: LLM-generated XPath scrapers are cheap per page but were never designed for extracting an entire catalogue, so script generation works best on stable, repetitive layouts.

The boundary between self-hosting and a managed API is maintenance, not capability. Playwright, stealth patches, proxy rotation, retry logic, change detection and schema validation all break on someone else's schedule. When the hours you spend fixing 403s and malformed JSON exceed the subscription cost, the managed route wins.

👉 [See what 100,000 credits a month costs on Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Do you get blocked, and where is the legal line?

Yes, you get blocked, and the fix is rarely more aggression. Rotate user agents, use residential proxies where appropriate, slow down between requests, and keep a cookie jar for consent banners and session redirects. Set hard limits first: robots.txt, terms of service, and copyright define what you may collect regardless of what the tooling allows.

A 2026 benchmark across 35 sites and five security tiers, including authentication, anti-bot and CAPTCHA controls, found that end-to-end LLM agents made complex scraping reachable with as little as a single prompt and fewer than five refinements. The same study found the trade-off: agents take longer to execute and produce less stable output because their reasoning is non-deterministic. On static sites, LLM-generated scripts were simpler and faster. Pick agents for unfamiliar interactive flows, scripts for known layouts.

Log every request with URL, timestamp, status code, latency and error. When a rights holder asks what you collected and when, that log is your answer.

👉 [Check how anti-bot bypass is billed per request](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Freshness, deduplication and the work nobody budgets for

Change detection is where RAG pipelines quietly rot. Store the source URL and a hash of the extracted content for every document, re-scrape on a schedule matched to how often each source updates, and compare hashes before re-embedding. Keep older snapshots so a retrieval regression can be traced to a specific date rather than guessed at.

Handle failures without stalling. If a page fails after retries, record it, move on, and requeue it separately; one dead URL should never block a 5,000-page refresh. Deduplicate before storage, since the same documentation page reachable through three URLs will otherwise occupy three slots in your index and crowd out the answer.

This is the part of the build that a managed API absorbs, and it is also the part most comparison articles skip. Rendering, conversion and change detection are the recurring cost; the initial scraper is the cheap part.

## FAQ

**How do I cut token costs without losing accuracy?**
Convert to markdown before chunking so scripts, styles and navigation never reach the model, then split on headings and keep tables whole. Feeding raw HTML and asking the model to ignore boilerplate costs more and extracts worse.

**What chunk size should I use?**
Start at 1,500 tokens for a local 7B to 14B model and 6,000 for a hosted frontier model, splitting at line boundaries. Adjust upward only if answers are missing context that sits just past the boundary.

**Is a local model good enough for extraction?**
For schema-driven field extraction with a strict prompt and temperature 0.0, often yes. The constraint is context window, which is why local defaults are smaller, not model quality alone.

**Will the LLM invent fields that are not on the page?**
Schema validation at the API boundary plus zero temperature makes it rare. Empty results are the more common failure, and they usually mean the prompt was too vague or the target page genuinely lacks the data.

**Can an agent replace my scraper script?**
On unfamiliar interactive sites, sometimes. On static documentation and product pages, a generated script runs faster, costs less and returns the same output every time.

## Which setup fits you

Scrape a handful of static documentation sites, run it weekly, and you can stay on Playwright plus a schema validator. You will spend a few hours a month on maintenance and nothing on credits.

Run the pipeline on a schedule, hit Cloudflare-protected pages, need change detection and a spending cap, and the managed route pays for itself. Start on the free plan to price your actual target list with the Domain Cost Estimator, then size the tier against real per-domain costs rather than page counts.

Before you order, confirm two things on the live page: how many credits your specific domains consume per request, including the +10 bypass surcharge, and whether your monthly volume fits inside a single allowance, since unused credits do not roll over. Pay-as-you-go is only available from Scaling upward; on Hobby, Startup and Business you get a 429 when credits run out unless auto-upgrade is on.

👉 [Check current ScraperAPI plans, credits and the 7-day refund terms](https://www.scraperapi.com/pricing?fp_ref=coupons)
