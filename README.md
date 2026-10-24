# llm web scraping: how the fetch layer actually works, what each page really costs, and when ScraperAPI beats a raw LLM pipeline

Every "llm web scraping" project is two projects stitched together: a scraper that gets you the page, and a model that reads it. The model gets all the attention, but in production the fetching half decides whether your pipeline produces clean data or quiet garbage, because an LLM can only extract from whatever HTML you manage to hand it. This guide walks through the full pipeline, the real per-page costs on both the fetch and model side, and where a scraping API like ScraperAPI fits when your target sites don't hand pages over willingly.

> **Quick take:** ScraperAPI works as the fetch layer — it handles proxies, JavaScript rendering, and CAPTCHA/anti-bot bypassing (Cloudflare, DataDome, PerimeterX), and ships an official MCP server so Claude, Cursor or Windsurf can call it directly. Free tier: 1,000 credits/month, no card required. Paid plans run from **$49/mo** (100,000 credits) to $1,975/mo (21.5M credits), and a 7-day trial gives you 5,000 credits to test your actual target domains. 👉 [Start the free trial — 5,000 credits, no credit card](https://www.scraperapi.com/signup?fp_ref=coupons)

## What llm web scraping actually involves

It involves four stages, and the LLM only handles one of them. The clearest breakdown of the current workflow describes llm web scraping as a four-stage pipeline — fetch, render, clean, extract — with the model doing the last stage alone. Stages one and two are where scraping projects fail.

The stages, in order:

1. **Fetch** — HTTP requests, IP rotation, rate-limit handling, CAPTCHA solving. The most common failure point.
2. **Render** — a headless browser executes JavaScript so dynamic pages look the way they do in a real browser.
3. **Clean** — strip navigation and boilerplate, convert HTML to plain text or Markdown.
4. **Extract** — the LLM reads the cleaned page and returns structured records against your schema.

There's a whole ecosystem of open-source tools built around this pattern — Crawl4AI, LLM Scraper, Silkweb — and they're genuinely useful if you want to run your own infrastructure. What they all have in common is the same weak point: you still need proxies, rendering capacity and bypass logic for stage one and two, and that's the part that gets expensive to maintain yourself.

## Where LLMs help, and where they quietly fail

LLMs beat CSS selectors when page structure varies but the meaning stays stable: job listings, product reviews, messy description text. Selectors run free and deterministically, but they break the moment a site redesigns. The failure modes differ in an important way: a broken selector fails loudly — it returns empty rows and you notice immediately. An LLM handed an unexpected page (a login wall, a CAPTCHA interstitial) fails quietly — it returns plausible-looking results, and nobody notices for weeks.

That quiet failure mode is the strongest argument for putting solid anti-bot infrastructure in front of the model. If your fetch layer returns the actual page every time, the model has nothing strange to hallucinate about. Hand it a block page, and it will happily "extract" product prices from a Cloudflare challenge screen.

Three technical limits worth knowing before you build:

- **LLMs can't see data that lives only in markup.** A rating encoded as `class="star-rating Three"` is invisible to a model reading page text. Same for prices rendered inside images or canvas elements.
- **Models can't solve CAPTCHAs or defeat bot detection.** No prompt fixes a 403.
- **Prompting beats fine-tuning** for almost every extraction task; fine-tuning only pays off at high volume on a stable, narrow schema.

One framing that saves people real money: context windows stopped being the binding constraint. Most current models carry 1M-token windows, and a cleaned page is 2,000–8,000 tokens. Most "context limit" errors are really "I sent raw HTML" errors — raw HTML burns 10–50x the tokens of the same page as text. The fix is a converter, not a bigger model.

## What each page actually costs

Model cost per page is small if you clean the HTML first, and enormous if you don't. The single biggest cost lever in llm web scraping is converting HTML to plain text before prompting, which typically cuts input tokens by roughly 10x. Model choice is the second lever, worth another 10–25x between value and flagship tiers.

Here's what that means in dollars for the same extraction job across 1,000 pages, comparing cleaned text against raw HTML (~60k tokens each), using mid-2026 published prices:

| Model | Cleaned text (per 1,000 pages) | Raw HTML (per 1,000 pages) |
| --- | --- | --- |
| Gemini 2.5 Flash Lite | $0.58 | $6.08 |
| DeepSeek V4 Flash | $0.76 | $8.46 |
| Claude Haiku 4.5 | $6.00 | $61.00 |
| gpt-5.6-sol | $31.00 | $306.00 |

The same job runs from under a tenth of a cent to about three cents per page, purely based on decisions made before the model ever runs. Constrain output with a strict JSON schema rather than a polite request for JSON — JSON mode guarantees syntax, not completeness — and check your max output tokens if you're pulling hundreds of records from one page in a single call, since that ceiling (64k–384k depending on model) is a real constraint at scale.

Fetch costs follow a different logic, and this is where credit-based APIs vary wildly. Which brings us to the pricing tables.

## ScraperAPI plans and pricing: the full table

ScraperAPI runs seven paid tiers plus a free plan, all priced in USD with about 10% off for annual billing. Here's the complete lineup as shown on the official pricing page:

| Plan | Monthly | Annual (–10%) | Credits/mo | Threads | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | $0 | 1,000 | 5 | [Sign up free](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49/mo | $44.10/mo | 100,000 | 20 | [Start Hobby](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Startup | $149/mo | $134.10/mo | 1,000,000 | 50 | [Start Startup](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Business | $299/mo | $269.10/mo | 3,000,000 | 100 | [Start Business](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Scaling (most popular) | $475/mo | $427.50/mo | 5,000,000 | 200 | [Start Scaling](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Professional | $975/mo | $877.50/mo | 10,500,000 | 300 | [Start Professional](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Advanced | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | [Start Advanced](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22M+ | 500+ | [Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

Every paid tier includes JavaScript rendering, premium residential and mobile proxies, JSON auto-parsing, CAPTCHA and anti-bot bypassing, custom sessions and headers, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee.

A few plan differences that don't fit in the table but affect the decision:

- **Geotargeting**: Hobby and Startup are limited to US & EU locations. Business and above unlock country-level targeting worldwide — the first thing to check if you need results from specific countries.
- **Analytics history**: 30 days on Hobby/Startup, unlimited from Business up.
- **Overage**: when credits run out, scraping pauses by default until renewal. Hobby through Professional can opt into auto-upgrade to the next tier; Scaling and above can enable pay-as-you-go at a fixed per-credit rate, capped at a monthly limit you set. There's no silent overage billing on the lower tiers.
- Exceeding your concurrency limit returns a 429 and the request is turned away — no queuing, no extra charge.

## The credit math most reviews skip

Most reviews quote "100,000 credits for $49" and stop there. What a credit actually buys depends on the domain, and the spread is bigger than the gap between plan tiers:

- Standard page: **1 credit**
- Amazon: **5 credits**
- Google and Bing (all subdomains): **25 credits**
- LinkedIn: **30 credits**
- Sites behind Cloudflare, DataDome or PerimeterX: **+10 credits per bypass**

So the Hobby plan's 100,000 credits is 100,000 standard pages, but roughly 20,000 Amazon pages or 4,000 Google SERP scrapes. A LinkedIn page behind extra protection can run 35+ credits in a single request.

Two tools turn this from a guessing game into a budget: the **Domain Cost Estimator** in the dashboard, which shows the exact credit cost of any URL before you scrape it, and the **`max_cost` parameter**, which caps what any single request can consume. Set `max_cost` on every production job — it's the difference between "one stubborn domain behind heavy protection slowly ate my quota" and knowing exactly why your balance moved. 👉 [Check your target domains against the free 1,000 credits](https://www.scraperapi.com/signup?fp_ref=coupons)

## How ScraperAPI plugs into an LLM stack

Through an official MCP (Model Context Protocol) server, and it works with Claude Desktop, Cursor, Windsurf, Cline and other MCP-compatible clients. There are two modes: a hosted remote server (add the URL and your API key to the client config, no local install) or a self-hosted version via `pip install scraperapi-mcp-server` or Docker if you want control over timeouts and rate limits.

Once connected, the model gets real tools instead of guessing at URLs:

- **`scrape`** — any page, with `render=true` for JavaScript, `country_code` for geo-specific results, premium proxies, and output as Markdown, plain text, CSV or JSON. That Markdown output option matters for the token math above: the fetch layer does the HTML-to-text conversion before you pay for a single model token.
- **Structured Data Endpoints** — `google_search`, `google_news`, `google_jobs`, `amazon_product`, `amazon_search`, `walmart_search`, `ebay_product`, `redfin_for_sale` and more, all returning pre-parsed JSON. For supported sites, the LLM extraction step often becomes unnecessary: you pay more credits per request (5 for Amazon, 25 for Google) but skip model tokens entirely.
- **Crawler tools** — async full-site crawling (`crawler_job_start`, `crawler_job_status`), not just single-page fetches.
- **AI Parser** — build a reusable parser from one example URL, then apply it across similar pages at scale.

There's also a skills package (`npx skills add scraperapi/scraperapi-skills`) that helps the model pick the right tool per task, so it knows to use `google_search` when you want SERP data and `scrape` when you want raw page content. The free tier's 1,000 monthly credits work with the MCP server, which makes the whole thing cheap to evaluate. 👉 [Set up the MCP server with a free API key](https://www.scraperapi.com/signup?fp_ref=coupons)

If you'd rather skip the assistant and write code, the integration is one HTTP call: send the URL with your API key, set `render=true` where needed, request Markdown output, and feed the text to your LLM with a JSON schema. That's the whole pipeline.

## ScraperAPI vs Firecrawl for LLM projects

Firecrawl is the tool most llm web scraping comparisons name-check, because it was built around LLM-ready output — every basic scrape returns clean Markdown by default. It's a legitimate choice and sometimes the better one. The verified comparison, based on both companies' current public pricing and feature pages:

|  | ScraperAPI | Firecrawl |
| --- | --- | --- |
| Entry paid plan | $49/mo, 100,000 credits | $19/mo ($16/mo annual), 5,000 credits |
| Credits at the ~$300 level | 3,000,000 (Business) | 100,000 (Standard, $99/mo) |
| Credit pricing | Varies by domain: 1–30 credits | 1 credit = 1 page fixed; JSON/Question/Highlight formats add 4 credits/page |
| Output formats | HTML, JSON, Markdown, plain text, CSV | Markdown, HTML, JSON, screenshots |
| Structured data endpoints | Amazon, Google, Walmart, eBay, Redfin | Not listed among its public features |
| Anti-bot positioning | Core product: deep proxy rotation with global country targeting | Handled, less proxy-centric |
| Free tier | 1,000 credits/mo | 1,000 credits/mo |
| Trial | 7 days, 5,000 credits, no card | Free plan, no card |

The trade-off is concrete. Firecrawl's entry tier is cheaper, credits are predictable (one page, one credit), and extras like site mapping and an autonomous agent mode suit exploratory work. ScraperAPI's credits cost more per listed unit but buy infrastructure Firecrawl doesn't claim to match: deep proxy rotation, ultra-premium IPs for the hardest sites, and structured data endpoints that turn unpredictable domains like Google into a flat known cost. If your targets are mostly friendly docs pages and blogs, Firecrawl's Hobby plan is hard to beat on price. If they're SERPs, marketplaces, or anything behind an enterprise WAF, run both against your real URLs — the credit math usually flips once protection enters the picture.

## FAQ

**Do I still need proxies if I'm using an LLM to scrape?**
Yes. The model never touches the network. Proxies, rendering and CAPTCHA solving all happen in the fetch layer, and that's exactly the layer ScraperAPI replaces with one API call.

**How much does llm web scraping cost per page?**
Two costs stacked: fetching (credits, anywhere from 1 to 35 or more depending on domain and protection level) plus extraction (a fraction of a cent on value-tier models with cleaned text, up to tens of cents on flagships with raw HTML). Cleaning HTML to plain text before prompting is the biggest single lever, cutting input tokens roughly 10x.

**Can Claude or Cursor scrape websites directly?**
Not on their own — they have no network stack. Via MCP they can call a scraping backend as a tool, which is precisely what ScraperAPI's MCP server provides.

**What happens when ScraperAPI credits run out mid-month?**
Scraping pauses until renewal by default. Auto-upgrade (Hobby–Professional) moves you to the next plan automatically if you opt in, and pay-as-you-go (Scaling and above) keeps you scraping at a fixed per-credit rate under a monthly cap you set. Annual plans can also renew early for a fresh batch of credits.

## Which setup to pick

Match the tool to your targets:

- Mostly static, low-protection sites → Firecrawl's entry plan is the cheaper start; ScraperAPI's Hobby tier at $49/mo is the comparable ScraperAPI entry point.
- Google, Amazon, LinkedIn, or WAF-protected targets → ScraperAPI, starting at Hobby and checking the Domain Cost Estimator before committing.
- Driving everything from Claude or Cursor → ScraperAPI's MCP server plus structured data endpoints means the model often skips the extraction step entirely.
- Still validating the idea → spend nothing yet. The 7-day trial's 5,000 credits are enough to measure the real per-domain cost on your actual targets.

Before you pick a tier, do two things: run your real target URLs through the Domain Cost Estimator, and check whether a structured data endpoint already covers your site — pre-parsed JSON can cost more credits per request but far less than paying model tokens to parse raw HTML. When you're ready, 👉 [compare all ScraperAPI plans and start with the trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)
