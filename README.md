# scrapy proxy rotation: How to Set It Up in Scrapy Without the Classic Middleware Traps, and When a Managed Proxy API Saves You the Whole Mess

Rotating proxies in Scrapy comes down to one job: putting a different value into `request.meta["proxy"]` before each request leaves the spider. Everything else — middleware order, authentication headers, retry behavior — exists to make sure that one assignment actually takes effect. Get it right with a proxy list you control, and you can spread requests across dozens of IPs. Get it wrong, and your crawler silently fires every request from your real IP while looking perfectly healthy in the logs.

This guide covers both routes people actually take: rolling your own rotation middleware, and delegating the whole problem to a managed service like ScraperAPI, which handles proxy rotation, header rotation, retries, ban detection, and CAPTCHA bypass on the server side. We'll also go through the pricing math, because "unlimited proxy rotation" means very different things depending on how a service charges.

> **Quick pick:** If you're scraping targets with real anti-bot protection and don't want to babysit a proxy pool, most readers do fine on ScraperAPI's Hobby plan — $49/month (or $44.10/month billed annually) for 100,000 API credits and 20 concurrent threads, with a 7-day free trial to validate your target sites first. 👉 [check ScraperAPI's current plans and start the free trial](https://www.scraperapi.com/pricing?fp_ref=coupons). If your targets are unprotected and you already own cheap proxies, the DIY middleware below costs nothing but your afternoon.

## How proxy rotation works in Scrapy (and where DIY setups go wrong)

Scrapy doesn't rotate proxies for you. The built-in `HttpProxyMiddleware` reads the proxy from `request.meta["proxy"]` — that single key is the entire interface. Your rotation logic, wherever it lives, just needs to write to that key at the right time.

The most common failure isn't bad rotation logic. It's middleware ordering. Scrapy runs `process_request` in ascending priority order, and `HttpProxyMiddleware` sits at **750**. Your custom rotation middleware has to run *before* it — registered at something like 350. If you register it above 750, it executes too late, the proxy never gets set, and every request leaves from your real IP. The crawl "works," your logs look normal, and you only find out when the target bans your server. This is the number one reason a middleware that looks correct quietly does nothing.

Three more traps worth knowing before you write code:

1. **The stale auth header.** If you embed credentials in the proxy URL (`http://user:pass@host:port`), `HttpProxyMiddleware` sets the `Proxy-Authorization` header for you — once. On redirects and retries it will *not* overwrite an existing header, so your retry goes to the new proxy carrying the old proxy's credentials, and you get a `407 Proxy Authentication Required` that has nothing to do with your actual credentials. The fix: whenever you change `meta["proxy"]`, either set a fresh base64-encoded `Proxy-Authorization` yourself or pop the old one with `request.headers.pop(b"Proxy-Authorization", None)`.

2. **The proxy scheme.** Even for HTTPS targets, write `http://` as the proxy scheme. The proxy tunnels HTTPS traffic via HTTP CONNECT; using `https://` as the scheme is a classic first-attempt mistake that produces `TunnelError` and "could not open CONNECT tunnel" errors.

3. **`start_urls` bypasses your meta.** If you set the proxy per-request in `start_requests`, requests generated from `start_urls` won't carry it. Either override `start_requests` or set the proxy in a middleware so every request gets covered.

The supporting settings matter too. With rotation in play, sensible starting points are `CONCURRENT_REQUESTS` of 16–32 (proxies add latency; concurrency hides it), `DOWNLOAD_TIMEOUT` of 20–30 seconds so dead proxies fail fast instead of hanging on the 180-second default, `RETRY_TIMES` of 3–5 with `403` and `429` added to `RETRY_HTTP_CODES` so a soft-banned request gets a fresh IP on retry, and `AUTOTHROTTLE_ENABLED = True`. For stateless jobs, `COOKIES_ENABLED = False` prevents a session cookie set behind one exit IP from leaking to another.

## When self-rotation stops being enough

Self-built rotation solves *rate limiting*, not *blocking*. If your target simply throttles by IP, spreading requests across a pool fixes it. But plenty of targets outright block datacenter IP ranges, serve CAPTCHAs to anything that smells like a bot, or sit behind Cloudflare, Datadome, or PerimeterX. In those cases rotating through the same datacenter pool just distributes your 403s across more IPs — the error page gets a new IP each time and you're still not getting data.

The usual escalation path is residential or mobile exit IPs, which cost real money and burn out, plus header rotation, fingerprint handling, and CAPTCHA solving. That's a stack to build and maintain. Managed scraping APIs exist because most teams decide that maintaining it isn't their product. ScraperAPI is one of them: you send it the target URL, and it handles proxy/header rotation, automatic retries, ban detection, and CAPTCHA bypassing before returning the HTML. Its premium tier uses residential and mobile IPs, and all plans include custom anti-bot logic for protected sites.

## ScraperAPI plans, prices, and what's actually included

Every paid plan includes the same feature set — JS rendering, premium residential and mobile IPs, rotating proxy pools, custom headers, CAPTCHA and anti-bot handling, custom sessions, desktop/mobile user agents, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. What changes between tiers is volume, concurrency, and geotargeting granularity.

| Plan | Monthly price | Annual (billed yearly) | API credits | Threads | Get it |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 connections | [ Sign up free](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49/mo | $44.10/mo | 100,000 | 20 | [ Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149/mo | $134.10/mo | 1,000,000 | 50 | [ Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299/mo | $269.10/mo | 3,000,000 | 100 | [ Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling *(most popular)* | $475/mo | $427.50/mo | 5,000,000 | 200 | [ Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975/mo | $877.50/mo | 10,500,000 | 300 | [ Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | [ Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | — | 22,000,000+ | 500+ | [ Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Geotargeting splits the lineup in half: Hobby and Startup are limited to **US & EU regions only**. Country-level geotargeting starts at Business. Annual billing takes 10% off every tier, so Hobby runs $44.10/month effective if you pay yearly.

On top of the paid tiers there's a free plan — sign up and get 1,000 API credits with up to 5 concurrent connections, no credit card required — plus a 7-day trial with 5,000 credits for testing whether the service actually works on your targets before committing to anything.

## Hooking ScraperAPI into a Scrapy project

There are three official integration paths, in ascending order of convenience:

**1. API endpoint.** Write a helper that wraps your target URL and API key into a request to ScraperAPI's endpoint, then yield `scrapy.Request` against that wrapped URL. It works, but every request goes through a URL-rewriting function.

**2. Python SDK.** Install with `pip install scraperapi-sdk`, then:

python
import scrapy
from scraperapi_sdk import ScraperAPIClient

client = ScraperAPIClient('YOUR_API_KEY')

class QuotesSpider(scrapy.Spider):
    name = "quotes"

    def start_requests(self):
        urls = [
            'https://quotes.toscrape.com/page/1/',
            'https://quotes.toscrape.com/page/2/',
        ]
        for url in urls:
            yield scrapy.Request(client.scrapyGet(url=url), callback=self.parse)


No URL rewriting, no middleware, no auth headers. This is the fastest path from a working spider to a rotating one.

**3. Proxy Mode.** Use ScraperAPI like an ordinary proxy: set it in `request.meta["proxy"]`, with your API key as the username, against their proxy server on port 8001. One caveat from the official docs: Scrapy skips SSL verification by default, so you don't need to disable it for these requests. This mode plays nicely with existing code that already reads `meta["proxy"]`.

Two settings adjustments make the whole thing behave: set `CONCURRENT_REQUESTS` to match your plan's thread count (20 on Hobby, 50 on Startup, and so on) to actually use the concurrency you're paying for, and make sure `DOWNLOAD_DELAY` and `RANDOMIZE_DOWNLOAD_DELAY` are not enabled — they throttle your throughput and are unnecessary when the provider is managing the rate limits for you.

Request-level behavior is controlled with parameters sent alongside the URL: `render=true` renders JavaScript before returning HTML, `country_code=us` targets US IPs, `premium=true` switches to residential and mobile IPs, `session_number=123` pins the same exit IP across requests (handy for login flows), `keep_headers=true` passes your own headers through, `device_type` picks desktop or mobile user agents, and `autoparse=true` returns structured JSON for supported sites.

## The credit math most articles skip

Here's the part that determines whether a plan fits your project, and it's rarely spelled out clearly: a "credit" is the cost of a standard page request, but you won't be scraping only standard pages.

- A standard page: **1 credit**
- Amazon: **5 credits**
- Google and Bing, subdomains included: **25 credits**
- LinkedIn: **30 credits**
- Any site behind Cloudflare, Datadome, or PerimeterX, when ScraperAPI bypasses the protection: **+10 credits per request**
- Premium proxies alone: **10 credits**; premium proxies combined with JS rendering: **25 credits**

So a Hobby plan's 100,000 credits is 100,000 ordinary pages, but only 4,000 Google searches, or 10,000 Cloudflare-protected pages, or 2,000 rendered requests through premium residential IPs. Before you pick a tier, check your specific targets with the Domain Cost Estimator in the dashboard, and set a `max_cost` per scrape so a single request can't eat a chunk of your quota. Third-party pricing trackers make the same point: measure usable responses against your actual targets before converting credits into an expected request volume.

Failed requests don't cost you anything — the API returns a 500 status and skips the charge. ScraperAPI states that on most sites over 97% of requests succeed on the first attempt. And there's no penalty for briefly exceeding your plan's concurrency limit: you get a 429 asking you to dial back and retry, the request is turned away rather than queued, and nothing extra is billed.

If you run out of credits mid-month, scraping pauses by default until renewal. Two opt-in mechanisms keep you running: **auto-upgrade** (on Hobby through Professional) automatically bumps you to the next plan when credits run out, and **pay-as-you-go** (on Scaling through Enterprise) keeps scraping at a fixed per-credit rate up to a monthly cap you set. Annual plans can also renew early for a fresh batch of credits.

Worth a look at the full rate table before you commit to a tier: 👉 [compare every plan and its credit rules on ScraperAPI's pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons).

## DIY middleware or managed API: which fits your project

Both routes are legitimate, and the deciding factors are concrete:

**Go DIY when** your targets have no serious anti-bot protection, you only need IP diversity to dodge rate limits, and you already have or can cheaply rent a proxy list. The middleware above is an afternoon of work, and the open-source `scrapy-rotating-proxies` package (which tracks proxy health and backs off when proxies die) can serve as a reference. Be aware the original package hasn't been updated since May 2019, so you'd be building on unmaintained code — fine for simple jobs, risky for anything subtle.

**Go managed when** your targets fight back: CAPTCHAs, Cloudflare-style protection, login walls, or aggressive fingerprinting. At that point DIY means sourcing residential IPs, writing header/fingerprint logic, and wiring a CAPTCHA solver — recurring maintenance for every new target. ScraperAPI charges for that as credits, and the free plan plus 7-day trial let you measure the success rate on your exact targets before paying. If the trial shows 95%+ first-attempt success on your sites, the credit cost is straightforward to budget.

A middle case worth naming: session-based flows (logins, multi-step forms) are awkward with naive rotation because each request lands on a new IP. ScraperAPI's `session_number` parameter pins one exit IP for a session; replicating that in DIY middleware means building sticky-session logic yourself.

## FAQ

**Does a failed request cost credits on ScraperAPI?** No. Failed requests return a 500 status code and are not charged. Most sites see over 97% of requests succeed on the first attempt.

**What happens if I exceed my plan's concurrent thread limit?** You get a 429 response and the request is simply refused until you're back within the limit — no queue, no overage charge.

**Is there a free option?** Yes. The free plan gives 1,000 API credits with up to 5 concurrent connections, no credit card required, and there's a 7-day trial with 5,000 credits for evaluating targets.

**How do I keep the same proxy across several requests?** Set `session_number` to a fixed value on the requests that belong to one session; ScraperAPI reuses the same exit IP for all of them.

## The short version before you decide

If you're scraping a handful of cooperative sites and enjoy tinkering, build the middleware: priority 350, pop the auth header on every proxy change, fail fast with short timeouts. It's free and you'll learn the machinery.

If your targets are protected, your time has a price, or you've already lost an evening to `407` errors and mystery 403s, hand the rotation to ScraperAPI and pay in credits instead of maintenance. Practical breakpoints: personal projects and testing start at the free plan; regular production crawls fit Hobby or Startup ($49–$149/month, US & EU); country-level geotargeting and bigger workloads start at Business ($299/month); heavy pipelines land at Scaling and up. Confirm your target sites' per-request credit cost in the dashboard, turn on auto-upgrade if you'd rather not pause mid-month, and start with the trial so you're spending credits on a success rate you've actually measured: 👉 [start your ScraperAPI free trial and pick a plan](https://www.scraperapi.com/signup?fp_ref=coupons).
