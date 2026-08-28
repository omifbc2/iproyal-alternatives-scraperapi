# ScraperAPI PHP Web Scraping: A Practical Guide to Building Reliable Scrapers with Goutte, Guzzle, and the Official SDK — Setup, Code Examples, Geotargeting, and Pricing Plans Compared

If you've ever tried to scrape a website from a PHP stack, you've probably hit the same wall as everyone else: the page renders fine in your browser, but your cURL request returns a Spanish version, a CAPTCHA page, or a 403. That gap between "what the browser shows" and "what your script gets" is the entire reason proxy-and-rendering services exist — and it's also the reason the search term "scraperapi php" keeps coming up in developer forums.

This article walks through what actually works in PHP today: which libraries to pick, how to wire them into ScraperAPI, what the credit model really costs you, and which plan makes sense once your scraper leaves the prototype stage. No fluff, just the moving parts.

## Why PHP Still Shows Up in Web Scraping Conversations

PHP isn't the first language people reach for when they think "scraper." Python has Scrapy, Node has Cheerio, and both ecosystems have louder evangelists. But PHP runs a huge slice of the web — WordPress alone is a significant chunk of it — and a lot of internal tooling, legacy dashboards, and Laravel-based backends already speak PHP fluently. If your team already lives in PHP, switching languages just to pull product prices is overkill.

The honest truth is that PHP's scraping story is more mature than its reputation suggests. The community has been around long enough to build real tooling, and the language's request/response primitives are stable. What PHP historically lacked was a clean answer for proxy rotation, CAPTCHA handling, and JavaScript rendering — and that's exactly the gap a service like ScraperAPI fills. You keep your PHP code; you offload the hard infra to an API call.

## Choosing a PHP Scraping Library: Goutte, Guzzle, or the SDK

Before wiring anything to ScraperAPI, you need an HTTP client on the PHP side. There are three realistic paths:

**cURL directly** is the lowest-level option. It works, it's installed almost everywhere, and it's verbose. You'll spend a lot of lines doing things a library would do in one. Fine for one-off scripts, painful for anything you'll maintain.

**Guzzle** is the modern PHP HTTP client most projects standardize on. It handles redirects, cookies, and streaming cleanly, and it's the foundation a lot of other libraries build on. If you're already using Guzzle in your app, you don't need to add anything new to talk to ScraperAPI — you just point your existing client at the ScraperAPI endpoint with your API key in the URL.

**Goutte** sits on top of Guzzle and Symfony components (BrowserKit, DomCrawler, CssSelector) and gives you a browser-like API for filtering the response DOM with CSS selectors. It's the choice when you want to write `$crawler->filter('.product-name')->text()` instead of hand-rolling XPath. The trade-off is a few extra dependencies.

**ScraperAPI's official PHP SDK** (`scraperapi/sdk` on Packagist) is the fourth option, and arguably the cleanest if ScraperAPI is your primary data source. It wraps the API in a small client object so you write `$client->get($url, ["render" => true])->raw_body` instead of constructing endpoint URLs by hand. For projects that are "ScraperAPI-first," this is the lowest-friction path.

Here's the SDK setup in one block:

php
<?php
require __DIR__ . '/vendor/autoload.php';
use ScraperAPI\Client;

$client = new Client("YOUR_API_KEY");
$result = $client->get("https://example.com/", ["render" => true])->raw_body;

echo $result;


Install it with Composer:

bash
composer require scraperapi/sdk


That's the entire integration. The SDK accepts the same parameters as the raw API — `render`, `country_code`, `premium`, `ultra_premium`, `session_number`, `device_type`, `autoparse` — so you don't lose any functionality by using it.

## A Real PHP Scraper with Goutte + ScraperAPI

For a more complete example, here's the pattern the ScraperAPI team itself documents in their PHP tutorial — using Goutte for the DOM walk and ScraperAPI for the request layer:

php
<?php
require 'vendor/autoload.php';
use Goutte\Client;

$client = new Client();

// Route the request through ScraperAPI with a US proxy
$scraperApiUrl = 'http://api.scraperapi.com'
    . '?api_key=YOUR_API_KEY'
    . '&url=' . urlencode('https://example.com/products')
    . '&country_code=us';

$crawler = $client->request('GET', $scraperApiUrl);

$crawler->filter('.product-item')->each(function ($node) {
    $name  = $node->filter('.product-name')->text();
    $price = $node->filter('.product-price')->text();
    $link  = $node->filter('a.product-link')->attr('href');

    // Store $name, $price, $link somewhere
});


The interesting thing here isn't the parsing — that's just Goutte. The interesting thing is the `country_code=us` parameter. Without it, the same script run from a server in, say, Spain will return the Spanish-language version of the page, because the target site geolocates the request. With it, ScraperAPI routes through a US proxy and you get the same HTML a US visitor would see. That single parameter solves a problem that would otherwise require you to source, rotate, and manage your own US proxies.

## The Parameters That Actually Matter for PHP Projects

ScraperAPI exposes a handful of parameters that change both behavior and cost. Knowing which ones you need before you write the integration saves real money:

- `render=true` — executes JavaScript on the target page. Costs 10 extra credits per request. Use it only when the data you need is loaded by JS after initial render.
- `country_code=xx` — routes the request through a proxy in that country. No extra cost. Essential for any site that varies content by geography.
- `premium=true` — uses residential proxies. Costs 10 extra credits. Worth it for sites that block datacenter IPs aggressively.
- `ultra_premium=true` — for the hardest anti-bot targets. Costs 30 extra credits (or 75 with `render=true`). Only available on paid plans.
- `session_number=x` — keeps the same IP across multiple requests so you can log in or maintain a cart. No extra cost.
- `autoparse=true` — returns structured JSON instead of HTML for supported domains (Amazon, Google, Walmart, etc.). Costs 5 credits for those structured endpoints.

The cost math matters because ScraperAPI bills in **credits**, not requests. A plain HTML request is 1 credit. The same request with `render=true` is 10. A request to a Google SERP endpoint is 25. A heavily protected domain with `ultra_premium=true` and `render=true` is 75. The headline "100,000 credits" on the Hobby plan can mean anywhere from 1,333 to 100,000 actual requests depending on what you're scraping.

## What PHP Developers Hit When They Scale Up

Once your scraper moves from "runs once a day on cron" to "runs every five minutes across 14 servers," the problems stop being about PHP and start being about infrastructure. The recurring ones:

- **Concurrency limits.** Each ScraperAPI plan caps concurrent threads. Hobby is 20, Startup is 50, Business is 100, Scaling is 200. If your PHP workers fire more parallel requests than your plan allows, requests queue or fail. Either raise the plan or throttle your workers.
- **Credit burn from rendering.** JavaScript-heavy SPAs cost 10 credits each. A scraper that "looked cheap" at 1 credit/request gets expensive fast when every page needs rendering.
- **Geotargeting ceilings.** Hobby and Startup only support US and EU geotargeting. If you need data from Asia-Pacific or Latin America, you need Business or higher — that's a $299/month floor, not $49.
- **Running out mid-month.** On Hobby, Startup, and Business, hitting 100% of credits means you stop scraping until renewal. On Scaling and above, you get pay-as-you-go overages — you keep scraping at a fixed per-credit rate.

These are the things that decide which plan you actually need, not the credit count on the pricing page.

## ScraperAPI Pricing: Every Plan on the Table

Here's the full current lineup, with nothing omitted. Prices reflect the monthly billing option; annual billing knocks 10% off across the board.

| Plan | Monthly Price | API Credits / Month | Concurrent Threads | Geotargeting | Pay-As-You-Go | Get Started |
| --- | --- | --- | --- | --- | --- | --- |
| **Free** | $0 | 1,000 | 5 | — | ❌ | [Start free](https://www.scraperapi.com/?fp_ref=coupons) |
| **Hobby** | $49/mo | 100,000 | 20 | US & EU | ❌ | [Try Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Startup** | $149/mo | 1,000,000 | 50 | US & EU | ❌ | [Try Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Business** | $299/mo | 3,000,000 | 100 | Global | ❌ | [Try Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Scaling** | $475/mo | 5,000,000 | 200 | Global | ✅ | [Try Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Professional** | $975/mo | 10,500,000 | 300 | Global | ✅ | [Try Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Advanced** | $1,975/mo | 21,500,000 | 500 | Global | ✅ | [Try Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Enterprise** | Custom | 22M+ | 500+ | Global | ✅ | [Contact Sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

All plans include JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA and anti-bot detection, custom sessions, desktop and mobile user agents, automatic retries, unlimited bandwidth, and a 99.9% uptime SLA. The Professional and Advanced plans also include a limited-time bonus of 250K and 500K extra credits respectively.

## Which Plan Makes Sense for a PHP Project

The right plan depends less on PHP and more on what you're scraping and how often. A few concrete scenarios:

**You're scraping one site, once a day, for an internal report.** Start with the Free plan (1,000 credits, 5 concurrent connections) to validate. If it works, Hobby at $49 is plenty — 100,000 credits is a lot of plain HTML requests.

**You're building a price-monitoring tool in Laravel that hits 20 e-commerce sites hourly.** Each site costs 5 credits (Amazon-style structured endpoints) or 10 credits (with rendering). At hourly polling across 20 sites, you're burning roughly 14,400–28,800 credits/day, which lands you in Startup ($149) or Business ($299) territory. Pick Business if any of the sites need global geotargeting.

**You're running a multi-tenant SaaS where each customer's scraping volume is unpredictable.** This is the Scaling plan's sweet spot. The pay-as-you-go overage means a sudden spike from one customer doesn't break the whole pipeline — you keep scraping at a known rate instead of hard-failing. The $475/month floor is the price of not having to babysit capacity.

**You're an agency or data team pulling millions of pages per month across many clients.** Professional ($975) or Advanced ($1,975) — the per-credit rate drops meaningfully at these tiers, and the priority support starts mattering when something breaks at 2am on a deadline.

**You're at enterprise scale with custom SLAs.** Enterprise is a conversation with their sales team, not a checkout button.

## Common PHP Integration Pitfalls

A few things that come up repeatedly when developers wire ScraperAPI into PHP codebases:

**Forgetting to URL-encode the target URL.** When you build the ScraperAPI endpoint by string concatenation, any `&` or `=` in your target URL breaks the request. Always wrap the target in `urlencode()`. The official SDK handles this for you; raw Guzzle integrations don't.

**Mixing up credit cost with request count.** A scraper that "worked on the free tier" suddenly burns through Hobby in a week when you turn on `render=true`. Model the actual credit cost per request before you commit to a plan — ScraperAPI's dashboard has a Domain Multiplier tool that tells you the exact cost for any URL before you run it at scale.

**Ignoring the `sa-credit-cost` response header.** Every ScraperAPI response includes this header with the actual credits charged for that request. Logging it during development is the fastest way to catch surprise costs before they hit production.

**Not using `session_number` for login flows.** If your scraper needs to log in, every request needs to come from the same IP or the session invalidates. `session_number=x` pins the proxy across requests — without it, your "logged in" state disappears between calls.

**Treating failed requests as wasted credits.** ScraperAPI only charges for successful responses (200 and 404 status codes). If your scraper is failing a lot, you're not burning credits — you're burning time. Fix the failure rate first.

## A Note on the Async Option

For PHP projects specifically, the async endpoint is worth knowing about. PHP isn't great at long-running concurrent connections — the language's synchronous model means a worker tying up a connection for 30 seconds is a worker not doing anything else. ScraperAPI's async service lets you submit a batch of URLs in one call and collect results later, which fits PHP's request/response model better than holding 50 long-lived connections open.

You submit a job, get a job ID, and poll for results. For batch scraping — nightly product catalog pulls, weekly competitor sweeps — this pattern is more PHP-friendly than trying to run 200 parallel Guzzle promises.

## The Bottom Line on ScraperAPI and PHP

PHP doesn't need to apologize for its scraping story. With Guzzle or Goutte on the parsing side and ScraperAPI on the request side, you get a stack that handles proxies, rendering, CAPTCHAs, and geotargeting without leaving the language your app already speaks. The official PHP SDK makes the integration a five-line affair, and the credit model — once you actually understand it — is predictable enough to budget against.

The plan you pick should follow from your actual scraping pattern, not from the headline credit count. Run your real workload against the free trial first, watch the `sa-credit-cost` headers, and let the actual numbers tell you which tier you belong in. Most PHP projects land somewhere between Hobby and Business; the ones that need Scaling or higher usually know it before they finish reading the pricing page.

If you want to test it against your own PHP codebase, the free trial doesn't require a credit card and gives you 5,000 credits to work with — enough to know whether the integration fits your stack before you spend anything.

👉 [Start your free ScraperAPI trial](https://www.scraperapi.com/?fp_ref=coupons)
