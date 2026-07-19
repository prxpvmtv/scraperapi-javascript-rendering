# ScraperAPI JavaScript Rendering Complete Guide: How render=true Works, Instruction Sets, Credit Costs, Concurrency Limits and Which Plan to Pick (With Real Python & Node.js Code Examples)

If you've ever fired a `requests.get()` at a React, Vue, or Next.js site and stared at an empty `<div id="root"></div>` in the response, you already know the problem this article is about. The HTML your HTTP client receives has none of the data you actually want, because the data only exists after the browser runs JavaScript. **ScraperAPI JavaScript rendering** is one of the most popular ways to close that gap without spinning up your own headless Chrome fleet, but the moment you start using it, a second, less obvious question appears: how much does it actually cost in credits, how do you keep success rates high, and which plan is the right fit for your workload?

This guide walks through the whole picture — from the basic `render=true` parameter, to the Render Instruction Set for clicking and scrolling, to the credit multipliers that decide your real monthly bill, to a full comparison of every ScraperAPI plan. If you want to test it on your own targets first, you can 👉 [start a 7-day free trial with 5,000 credits — no credit card required](https://www.scraperapi.com/signup?fp_ref=coupons).

---

## Why JavaScript Rendering Is a Separate Problem

Static HTML scraping is mostly solved. You point a request at a URL, you get HTML back, you parse it. The trouble starts when the page you're targeting is a single-page application or relies on client-side scripts to populate content. The server sends a shell document, JavaScript runs in the browser, and only then do the prices, reviews, search results, or infinite-scroll items appear in the DOM.

A plain HTTP request never runs that JavaScript. You get the shell, not the content. Traditional workarounds — running Puppeteer or Playwright yourself, managing browser instances, handling timeouts, rotating proxies behind the browser — work, but they're heavy. You're now maintaining infrastructure, not just writing scrapers.

ScraperAPI's pitch is that it takes that whole problem off your hands. You add `render=true` to your API call, and the request is routed through a managed Chromium instance. The browser executes the page's scripts, waits for the DOM to settle, and returns the rendered HTML to you as a normal response. No browser process on your machine, no Playwright dependency in your project, no proxy rotation to babysit.

The feature is available on every plan — including the free trial — which is one of the reasons it's become a default starting point for developers who need to scrape JavaScript-heavy sites without building a rendering stack.

---

## How ScraperAPI JavaScript Rendering Actually Works

At the API level, enabling JS rendering is a single parameter. The same endpoint you'd use for any ScraperAPI request gets `render=true` appended, and that's the entire switch.

python
import requests

target_url = 'https://example.com/'
api_key = 'API_KEY'

request_url = (
    f'https://api.scraperapi.com?'
    f'api_key={api_key}'
    f'&render=true'
    f'&url={target_url}'
)

response = requests.get(request_url)
print(response.text)


The same call in Node.js, using the headers-based syntax ScraperAPI also supports:

javascript
import fetch from "node-fetch";

const url = new URL("https://api.scraperapi.com");
url.searchParams.append("url", "https://example.com");

const response = await fetch(url, {
  method: "GET",
  headers: {
    "x-sapi-api_key": "API_KEY",
    "x-sapi-render": "true"
  }
});

console.log(await response.text());


Behind the scenes, ScraperAPI spawns a headless Chromium instance, loads the URL, lets the page's JavaScript run, and only returns the response once the page has reached a steady state. You get back the same HTML a real browser visitor would see after the page finished loading — including any content that was injected by client-side scripts.

A practical example makes the value obvious. Hit a Vite/React demo page with a plain `requests.get()` and the body comes back as `<div id="root"></div>` — empty. Add `render=true` and route the same URL through ScraperAPI, and the same response now contains the full rendered DOM: headers, navigation, buttons, state, everything that was supposed to appear after the JavaScript bundle executed. The scraping code didn't change; one parameter did.

---

## Handling Slow-Loading Content With `wait_for_selector`

Rendering alone doesn't always solve the timing problem. Some pages load their initial shell quickly, then fetch the data you actually care about via XHR a few seconds later. If ScraperAPI returns the rendered HTML the moment the page reaches a steady state, you might still miss content that's deliberately delayed.

This is what `wait_for_selector` is for. It tells the rendering engine to hold the response until a specific element appears in the DOM, giving late-loading content time to render before the HTML is captured.

python
import requests

url = "https://api.scraperapi.com"
params = {
    "url": "https://example.com/products",
    "render": "true",
    "wait_for_selector": ".product-card"
}

headers = {"x-sapi-api_key": "API_KEY"}
response = requests.get(url, params=params, headers=headers)
print(response.text)


One important detail: `wait_for_selector` only works when `render=true` is also set. Specify it on its own and the API silently ignores it. The two parameters are a pair.

By default, ScraperAPI doesn't render every single resource on a page — images, tracking scripts, and other non-essential assets are skipped to keep response times down. That's good for speed, but occasionally it means something you need is missing from the response. `wait_for_selector` is the standard fix: tell the API what specific element you're waiting for, and it'll hold the page open until that element exists.

---

## The Render Instruction Set: Clicking, Scrolling, and Typing Inside the Rendered Page

Basic rendering gets you the page as it would appear on first load. A lot of modern sites need more than that — search forms have to be filled in, "load more" buttons have to be clicked, infinite-scroll pages have to be scrolled before the data you want exists in the DOM. This is where ScraperAPI's Render Instruction Set comes in.

The Instruction Set is a JSON array of actions the headless browser will execute in order, before the final HTML is returned. You pass it as the `x-sapi-instruction_set` header alongside `render=true`. Supported instructions cover most of what you'd do manually in a browser:

- **`click`** — click an element selected by CSS, XPath, or visible text
- **`input`** — type a value into an input field
- **`scroll`** — scroll the page vertically or horizontally, by pixel count or to top/bottom, or to a specific element
- **`wait`** — pause for a fixed number of seconds
- **`wait_for_event`** — wait for a browser event like `networkidle`, `domcontentloaded`, `load`, `navigation`, or `stabilize`
- **`wait_for_selector`** — wait for a specific element to appear
- **`loop`** — repeat a block of instructions N times (useful for infinite scroll, though nested loops aren't supported)

A real example: searching Wikipedia for "cowboy boots" by filling the form, clicking submit, and waiting for results to render.

python
import requests
import json

url = "https://api.scraperapi.com"
params = {"url": "https://www.wikipedia.org"}

instruction_set = [
    {
        "type": "input",
        "selector": {"type": "css", "value": "#searchInput"},
        "value": "cowboy boots"
    },
    {
        "type": "click",
        "selector": {
            "type": "css",
            "value": "#search-form button[type=\"submit\"]"
        }
    },
    {
        "type": "wait_for_selector",
        "selector": {"type": "css", "value": "#content"}
    }
]

headers = {
    "x-sapi-api_key": "API_KEY",
    "x-sapi-render": "true",
    "x-sapi-instruction_set": json.dumps(instruction_set)
}

response = requests.get(url, params=params, headers=headers)
print(response.text)


A few practical notes from ScraperAPI's own documentation:

- The `x-sapi-` prefix on every header is intentional. It prevents collisions with headers the target site itself might be reading.
- Fewer instructions means a higher success rate. Every action adds latency, and latency is what causes rendering requests to time out. ScraperAPI explicitly recommends limiting the instruction set to **3–4 actions** for best results.
- The `loop` instruction is the cleanest way to handle infinite scroll: pair a `scroll` to the bottom with a short `wait`, loop N times, and the page progressively loads more content into the DOM before the final HTML is captured.

For high-volume jobs, the same instruction set works against ScraperAPI's async endpoint (`https://async.scraperapi.com/jobs`) — you submit the job, get a job ID back, and poll for the result. Async is the right choice when you're firing thousands of rendered requests and don't want to hold a connection open for the 10–25 seconds each render can take.

---

## The Cost Question: How JS Rendering Affects Your Credits

This is the part most getting-started guides under-explain, and it's where most ScraperAPI budget surprises come from.

ScraperAPI bills in **API credits**, not requests. A standard scrape of an unprotected page costs 1 credit. JavaScript rendering is not free on top of that — it adds a multiplier:

| Request configuration | Credits per request |
|---|---|
| Standard scrape (no rendering) | 1 |
| JS rendering (`render=true`) | 10 |
| JS rendering + premium proxy | 25 |
| JS rendering + ultra premium proxy | 75 |

That multiplier stacks with **domain-based** multipliers. Scraping Amazon costs 5 base credits; Google and Bing cost 25; LinkedIn costs 30; sites behind Cloudflare, DataDome, or PerimeterX add another 10. So a rendered scrape of a Cloudflare-protected Amazon page isn't 1 credit — it's potentially 15 or more, depending on which proxy tier is required to get through.

This is why a plan with "100,000 credits" doesn't necessarily mean 100,000 successful scrapes. If your target is a JavaScript-rendered e-commerce page costing 10 credits per request, that 100,000-credit allowance covers roughly 10,000 successful scrapes. Before committing to a plan, it's worth running a few test requests and using the **Domain Cost Estimator** in the ScraperAPI dashboard to see the real per-request cost for your specific targets.

One genuinely fair detail: **you're only billed for successful requests.** Anything outside a 200 or 404 response doesn't burn credits, so you're not paying for the service's own failures — only for data it actually delivered.

---

## Concurrency: The Hidden Rendering Limit

Even if you have the credits, there's a second ceiling: rendering concurrency.

By default, ScraperAPI caps rendering throughput at **10 rendered requests per second** (the "burst limit"). This is separate from the concurrent-threads limit on your plan. If each rendered request takes 25 seconds to complete and you're consistently firing 10 per second, you can have up to 250 rendered requests running in parallel at any moment — but you can't start more than 10 new ones each second.

For most workloads this is plenty. If you're running rendering jobs at a scale where 10 req/sec starts to feel restrictive, that's enterprise territory, and ScraperAPI's support team can raise the limit on request.

The practical implication for plan selection: a higher-tier plan with more concurrent threads doesn't automatically mean more rendering throughput. If your workload is rendering-heavy, the burst limit, not the thread count, is often the real bottleneck.

---

## Full ScraperAPI Plan Comparison (All Tiers)

Every ScraperAPI plan includes the same feature set — JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA and anti-bot bypass, custom sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. The differences between tiers are volume (credits/month), concurrency (concurrent threads), geotargeting scope, analytics history length, and whether pay-as-you-go overflow is available when you run out of credits mid-cycle.

| Plan | Monthly Price | Annual Price (10% off) | API Credits / Month | Concurrent Threads | Geotargeting | Get Started |
|---|---|---|---|---|---|---|
| **Free Trial** | $0 (7 days) | — | 5,000 (one-time) | 5 | — |  [Start free trial — no card required](https://www.scraperapi.com/signup?fp_ref=coupons) |
| **Hobby** | $49/mo | $44.10/mo | 100,000 | 20 | US & EU only |  [Get the Hobby plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Startup** | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU only |  [Get the Startup plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Business** | $299/mo | $269.10/mo | 3,000,000 | 100 | Global |  [Get the Business plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Scaling** *(most popular)* | $475/mo | $427.50/mo | 5,000,000 | 200 | Global |  [Get the Scaling plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Professional** | $975/mo | $877.50/mo | 10,500,000 | 300 | Global |  [Get the Professional plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Advanced** | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global |  [Get the Advanced plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Enterprise** | Custom quote | Custom quote | 22,000,000+ | 500+ | Global |  [Contact sales for Enterprise pricing](https://www.scraperapi.com/?fp_ref=coupons) |

A few things worth noting that aren't obvious from the price column:

- **Geotargeting is gated by tier.** Hobby and Startup are restricted to US and EU proxies. If your target site shows different content by country and you need anything beyond US/EU, Business ($299/mo) is the minimum.
- **Pay-as-you-go overflow only kicks in at Scaling and above.** On Hobby, Startup, and Business, running out of credits mid-cycle means upgrading or waiting for renewal — there's no overage billing.
- **Credits don't roll over.** Whatever you don't use resets at each renewal, so it's worth sizing your plan to actual monthly volume rather than overbuying "just in case."
- **Unlimited analytics history** starts at Business; Hobby and Startup are capped at 30 days of dashboard data.
- **Priority support** is included from Professional upward.

Annual billing applies an automatic 10% discount across all paid plans — no promo code needed, it's applied at checkout. You can 👉 [check current pricing and sign up here](https://www.scraperapi.com/?fp_ref=coupons).

---

## Which Plan Should You Pick for JavaScript Rendering?

The right plan for JS rendering workloads depends on two numbers: how many pages per month you need to render, and how many credits each rendered page actually costs you. Here's how that maps to the tiers in practice.

**Pick the Free Trial first.** Always. The 5,000-credit trial costs nothing, requires no credit card, and is the only reliable way to find out your real per-page cost before committing. Run a handful of test requests against your actual targets with `render=true` enabled, then check the credit consumption in the dashboard. A 5,000-credit trial against a normal blog gets you 5,000 test scrapes; the same trial against a JavaScript-rendered Amazon page might get you 300. That single number tells you more than any pricing table can.

**Pick Hobby ($49/mo) if** you're running a personal project, a small side hustle, or a prototype. 100,000 credits covers a lot of ground for plain pages, but with rendering's 10x multiplier, your real ceiling is closer to 10,000 rendered pages per month — and less if your targets also carry domain-based multipliers. Geotargeting is US/EU only.

**Pick Startup ($149/mo) if** you've outgrown casual scraping and need consistent volume — say, a small SaaS product or an agency running scraping jobs for a handful of clients. 1,000,000 credits with 50 concurrent threads is a meaningful step up, and at 10 credits per rendered request, it covers roughly 100,000 rendered pages per month. You're still capped at US/EU geotargeting.

**Pick Business ($299/mo) if** you need global geotargeting (not just US/EU), unlimited analytics history, or you're running production-grade infrastructure that other parts of your business depend on. The jump to 100 concurrent threads also starts to matter for parallel rendering jobs, though remember the 10 req/sec rendering burst limit still applies.

**Pick Scaling and above if** you're past the "which plan" question and into "how do I keep this predictable at high volume" territory. From Scaling upward, pay-as-you-go overflow billing means you're never hard-capped mid-month — overage is billed at a fixed rate. From Professional upward, you also get priority support. These tiers exist for teams running millions of rendered requests per month against hard targets.

To put a number on it: a project scraping 200,000 JavaScript-rendered pages per month, where each rendered request costs 10 credits, needs 2,000,000 credits. That fits within the Business plan (3,000,000 credits) but not Startup. The same project with global geotargeting requirements also forces the Business tier, since Startup doesn't support it. Run your own numbers — every workload is different.

👉 [Start with the free trial, run your numbers against your real targets, then pick a plan](https://www.scraperapi.com/signup?fp_ref=coupons).

---

## Tips for Higher Success Rates With JS Rendering

JS rendering has more failure modes than plain scraping. Pages time out, content appears later than expected, clicks don't register on the first attempt. A few practices consistently improve success rates:

1. **Limit your instruction set to 3–4 actions.** ScraperAPI's own documentation is explicit about this — every additional action adds latency, and latency is what causes rendering requests to time out. If you find yourself building a 10-step instruction set, you're probably better off splitting the work across multiple requests.
2. **Use `wait_for_selector` instead of fixed `wait` times.** A fixed 10-second wait is wasteful on fast pages and insufficient on slow ones. Waiting for the specific element you care about is both faster and more reliable.
3. **Use `wait_for_event: networkidle` for XHR-heavy pages.** If the page loads its content via background API calls, waiting for the network to go idle is a better signal than waiting for a particular selector.
4. **Use async for high-volume rendering.** The synchronous endpoint holds your connection open for the full render duration. At scale, you'll exhaust your connections long before you exhaust your credits. The async endpoint accepts the job, returns immediately, and lets you poll for results.
5. **Skip rendering when you don't need it.** ScraperAPI's documentation is unusually direct about this: rendering increases latency and can reduce success rates, which reduces the volume you can process. Only enable `render=true` for pages that genuinely require it. For static pages, leave it off and pay 1 credit per request instead of 10.
6. **Test with the Domain Cost Estimator first.** Before scaling any rendering workload, run a few requests through the cost estimator in the dashboard. The difference between expected and actual credit consumption is where most budget surprises come from.

---

## What Users Actually Say About ScraperAPI

Independent review aggregation paints a fairly consistent picture. ScraperAPI sits around **4.5/5 on Trustpilot** and **4.4/5 on G2**, with the majority of reviews in five-star territory.

The recurring praise points are the same across most platforms: clean documentation, a genuinely simple integration (drop it into existing code as a proxy replacement), and responsive support. One long-time reviewer specifically called out that upgrading or downgrading plans was painless, which tracks with how the credit system is structured — there's no lock-in beyond the current billing cycle.

The most common complaint isn't about reliability — it's about the credit math being less intuitive than the headline number suggests, especially once you start mixing rendering and premium-proxy parameters on harder targets. Independent benchmarking from third-party testers has also noted that performance varies a lot by target: ScraperAPI performs very well on mainstream sites like Amazon, GitHub, and standard e-commerce pages, but less consistently on sites with aggressive, frequently-changing anti-bot systems.

For a focused JS-rendering workload, the relevant takeaway is this: the rendering feature itself is reliable, the documentation is clear, and the integration is genuinely simple. The friction, when it appears, is almost always about cost — which is exactly why running your own numbers during the free trial matters more than any review.

---

## How ScraperAPI Compares to Other JS Rendering APIs

If you're evaluating ScraperAPI against alternatives for a JavaScript-rendering workload, here's roughly how the positioning shakes out based on current market comparisons:

- **vs. Bright Data** — Bright Data is the more powerful, more expensive enterprise option, generally starting around $499/mo, aimed at teams that need the highest possible success rates regardless of cost. Choose ScraperAPI for simplicity and lower cost at small-to-mid scale.
- **vs. Scrape.do** — Scrape.do undercuts ScraperAPI on raw entry price (around $29/mo), which appeals to budget-conscious solo developers running simple, unprotected scrapes. Choose ScraperAPI if you need structured data endpoints, the Render Instruction Set, or higher concurrent thread counts.
- **vs. ScrapingBee** — Similar developer experience and a comparable $49/mo entry point, generally without the same credit multiplier system, which makes its costs more predictable for some workloads. Choose ScraperAPI if your targets include Amazon, Google, or LinkedIn — ScraperAPI's prebuilt structured-data endpoints for those domains are a real time-saver.
- **vs. Firecrawl** — Firecrawl positions itself as the AI-ready option, with cleaner Markdown and LLM-friendly output. Choose ScraperAPI if you want raw HTML control and the ability to script browser interactions via the Instruction Set.

None of these are universally better — it depends on whether your priority is price predictability, raw success rate on hard targets, or ease of integration. For most developers running moderate-volume JS-rendering scrapes against mainstream sites, ScraperAPI's balance of price, simplicity, and the Instruction Set is exactly why it remains one of the most recommended starting points in this category.

---

## Getting Started: A Practical First Run

The fastest way to evaluate whether ScraperAPI JavaScript rendering fits your workload is to actually run it against your real targets. The whole process takes less than five minutes.

1. **Sign up for the free trial.** You get 5,000 API credits for 7 days, no credit card required. 👉 [Start the free trial here](https://www.scraperapi.com/signup?fp_ref=coupons).
2. **Grab your API key** from the dashboard.
3. **Run your first rendered request** — the Python or Node.js snippet earlier in this article works as-is, just swap in your key and target URL.
4. **Check the credit consumption** in the dashboard. The number you see per request is your real per-page cost — not the headline credit allowance.
5. **Test the Render Instruction Set** if your target needs clicks, scrolls, or form input. Start with 2–3 actions and add more only if needed.
6. **Size your plan to the numbers you observed.** If 5,000 credits got you 500 rendered pages, you know exactly what 100,000 credits (Hobby) or 1,000,000 credits (Startup) will buy you.

The free trial is generous enough to test against real targets, not toy examples. That's the whole point — by the time the trial ends, you should know your actual cost per successful rendered scrape and be able to pick a plan with confidence. 👉 [Claim your 5,000 free trial credits here](https://www.scraperapi.com/signup?fp_ref=coupons).

---

## Frequently Asked Questions

**Does ScraperAPI JavaScript rendering work on every plan?**  
Yes. The `render=true` parameter is available on every plan, including the 7-day free trial with 5,000 credits. There's no "rendering-only on Business and above" restriction — the difference between tiers is volume, concurrency, and geotargeting scope, not feature gating on rendering itself.

**How many credits does a single JS-rendered request cost?**  
A rendered request costs 10 credits. If you combine rendering with a premium proxy, it costs 25 credits. With an ultra premium proxy, it costs 75 credits. These multipliers stack on top of any domain-based multiplier (Amazon = 5, Google = 25, LinkedIn = 30, etc.) and any anti-bot bypass surcharge.

**What's the rendering concurrency limit?**  
The default burst limit is 10 rendered requests per second. This controls how many new rendered requests you can start each second, separate from your plan's concurrent-threads cap. If each render takes 25 seconds and you're firing 10 per second consistently, you can have up to 250 rendered requests running in parallel at any moment. Enterprise customers can request a higher limit through support.

**Can I click buttons and fill forms during rendering?**  
Yes, via the Render Instruction Set. You pass a JSON array of actions (click, input, scroll, wait, wait_for_selector, wait_for_event, loop) as the `x-sapi-instruction_set` header alongside `render=true`. The browser executes the actions in order before returning the final HTML. ScraperAPI recommends keeping instruction sets to 3–4 actions for best success rates.

**What happens if I run out of credits mid-month?**  
On Hobby, Startup, and Business, you can upgrade to the next tier or contact support about a custom arrangement. On Scaling, Professional, Advanced, and Enterprise, pay-as-you-go overflow billing kicks in at a fixed rate, so you can keep scraping without a hard cap. Credits don't roll over either way — whatever you don't use resets at renewal.

**Is there a refund policy?**  
Yes — ScraperAPI offers a 7-day, no-questions-asked refund if you're not satisfied with the service. Combined with the no-card-required free trial, you can evaluate the service thoroughly before any paid commitment.

**Does `wait_for_selector` work without `render=true`?**  
No. `wait_for_selector` requires `render=true` to be set in the same request. Specify it on its own and the API silently ignores it.

---

## Bottom Line

ScraperAPI JavaScript rendering solves a real, common problem — getting data out of pages that only exist after JavaScript runs — without forcing you to maintain a headless-browser fleet. The `render=true` parameter is genuinely one-line, the Render Instruction Set handles the click-and-scroll cases that basic rendering can't, and the feature is available from the free trial upward, so you can validate it against your real targets before paying anything.

The catch, as with any credit-based scraping API, is the cost math. Rendering multiplies your per-request credit cost by 10, and that multiplier stacks with domain and proxy multipliers. The plan you pick should be driven by your measured cost per successful rendered scrape, not the headline credit allowance. Run your numbers during the free trial, use the dashboard's Domain Cost Estimator, and size accordingly.

If your workload is mostly plain pages, the Hobby plan at $49/month covers a lot of ground. The moment JavaScript rendering, global geotargeting, or hard targets enter the picture, run the math first — the sticker price and the real cost per successful scrape are two different things, and the free trial is the cleanest way to find out which one applies to you. 👉 [Start your free ScraperAPI trial — 5,000 credits, no credit card required](https://www.scraperapi.com/signup?fp_ref=coupons).
