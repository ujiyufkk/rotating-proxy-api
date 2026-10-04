# rotating proxy api: How to Plug Rotating Residential IPs Into Python and cURL Without Paying for Bandwidth You Never Use

Most people searching for a rotating proxy API aren't shopping for a feature. They have a scraper that keeps getting blocked, or a price-monitoring job that needs to look like it's coming from a dozen different homes, and they want the shortest path from a payment to a working endpoint inside their code.

That's really three questions stacked on top of each other: does the provider give me one host:port that changes IP on every request, can I control targeting and session length from my code instead of clicking around a dashboard, and what does that cost when my requests are tiny but constant?

9Proxy answers those questions in a less obvious way than most providers do, and the split between its two product lines is the whole decision here. Getting it wrong means either paying per gigabyte for a workload that never uses much traffic, or paying per IP for a job that only ever needs one request per site visit.

## Rotating proxy API means three different things

Before comparing anything, it's worth separating the term into the parts people actually mean:

1. **A rotating gateway.** One endpoint, one username, and every request exits through a different residential IP. Nothing in your code changes between requests.
2. **Session controls inside the credentials.** Stickiness, session duration, country, city, ISP — all expressed as parameters rather than separate API calls.
3. **A provisioning API.** Real HTTP endpoints with an API key that let you allocate ports, generate sub-users, pull usage logs, and rotate programmatically at scale.

Providers often deliver only one or two of these. 9Proxy delivers the first two through its GB-based product and the third through a separate Public API, which is worth knowing before you assume "no API" when a dashboard-only product line is all you've seen. Several third-party review pages still flatly state that 9Proxy doesn't offer rotating proxies at all — those write-ups describe the older IP-based desktop app model and predate the bandwidth product, which is where rotation actually lives.

## How rotation works on 9Proxy's GB-based proxies

The GB-based product is the one built for rotation. You don't buy a fixed set of IPs; you buy a bucket of traffic and generate as many endpoints as you want, then let the IP change per request or hold steady for a session you define. Direct targeting down to country, state, city, ZIP and ISP is available, and the whole thing runs from the dashboard with no desktop app involved.

### The username is the configuration

Instead of passing rotation options as query strings or headers, 9Proxy encodes them in the proxy username. The general shape is:


<sub-user>-country-<country_code>-st-<state_code>-city-<city_name>-isp-<isp_code>-sst-<session_time>-ssid-<session_id>


Nothing here is mandatory. You include only the parts you need.

| Parameter | Example | What it does |
| --- | --- | --- |
| sub-user | `bot01` | Your sub-account name, issued in the dashboard |
| country | `country-us` | Routes traffic through a two-letter country code |
| st | `st-ohio` | Optional state or region filter |
| city | `city-newyork` | Optional city targeting, underscores for spaces |
| isp | `isp-as22773_Cox_Communications_Inc.` | Optional ISP or ASN filter |
| sst | `sst-15` | Sticky session length in minutes before the IP rotates |
| ssid | `ssid-device1` | Session ID, so parallel sticky sessions don't collide |

Country targeting keeps the largest pool available. Add state, city and ISP together and you're slicing that pool down, which sometimes means fewer IPs than you asked for. Use ISP or city filtering when fingerprint matching genuinely requires it, not by default.

### Rotating mode vs sticky mode

For a rotating endpoint, you leave out `sst` and `ssid` entirely. Every request picks up a fresh IP:


subaccount-country-us


For a session that holds one IP — a login flow, a cart, anything that breaks when the IP changes mid-task — you set a duration:


subaccount-country-us-sst-15-ssid-device1


That IP stays put for 15 minutes, then rotates. `ssid` matters more than people expect: run ten parallel threads with the same `sst` and no `ssid` and they'll all share one session, which is usually the last thing you want.

### A working cURL and Python example

The cURL version, straight from the documented pattern:

bash
curl -x your_proxy_host:your_port \
     -U "subuser-country-us-sst-15-ssid-bot01:your_password" \
     https://ipinfo.io


The same thing in Python, with a rotating pool for the bulk of your requests and a sticky session for the one job that needs continuity:

python
import requests

PROXY_HOST = "your_proxy_host:your_port"
PASSWORD = "your_password"

def rotating_proxy(country="us"):
    user = f"bot01-country-{country}"
    return {"http": f"http://{user}:{PASSWORD}@{PROXY_HOST}",
            "https": f"http://{user}:{PASSWORD}@{PROXY_HOST}"}

def sticky_proxy(session_id, minutes=15, country="us"):
    user = f"bot01-country-{country}-sst-{minutes}-ssid-{session_id}"
    return {"http": f"http://{user}:{PASSWORD}@{PROXY_HOST}",
            "https": f"http://{user}:{PASSWORD}@{PROXY_HOST}"}

# fresh IP per request
for url in batch_of_urls:
    r = requests.get(url, proxies=rotating_proxy(), timeout=20)

# one IP held for 15 minutes
session = requests.Session()
session.proxies.update(sticky_proxy("checkout-01"))


Because the rotation logic sits in the credentials, none of this depends on a special SDK. Any HTTP client that accepts a proxy URL works, which is why the same pattern shows up in cURL, `requests`, Scrapy and most headless browser setups without modification.

## Beyond the endpoint: the Public API

The rotating gateway is the part your scraper touches. The provisioning layer is separate, and it's where 9Proxy stops being "just a proxy endpoint."

The Public API authenticates with an API key generated from the dashboard under My Account, then exposes two base domains: `api.9proxy.com` for live operations and `sandbox.9proxy.com` for testing. Keys currently inherit your account's full permission set, so there's no granular scoping to configure.

### Provisioning rotation with one request

The endpoint that matters most for rotation is proxy connection config creation, which allocates a port range and applies your targeting and session settings in one call:


POST /client/v1/proxy-connection/create


The body takes targeting fields (`country_code`, `state_code`, `city_code`, `isp_code`), a `quantity` for how many ports to allocate, and — the interesting part — a `session_type` where **1 means rotation and 2 means sticky**, plus `session_time` in minutes for sticky sessions. That's the dashboard "Proxy Generator" exposed as JSON, which means you can spin up a rotation pool per campaign, per client or per job rather than hand-configuring ports.

Around it sit `get-list` for auditing existing configs, `update` keyed on `start_port` for changing targeting or session mode, and `delete` for tearing down port ranges when a job finishes. That last one is the piece most per-IP products make you do by hand.

### What else the API reaches

The same key unlocks account balance and purchase history, usage logs, sub-account creation, share codes, wallet operations, user-pass and whitelist management, and affiliate tracking data. On the team side, that maps to a workflow agencies already run: create one sub-user per client, assign traffic, and keep the logs separated.

## Authentication: sub-user credentials or IP whitelist

Two options, and the choice is mostly about where your code runs.

Sub-users give you a username and password, which is what you want for cloud functions, containers, CI jobs and anything with a dynamic egress IP. IP whitelisting skips passwords entirely — whitelist the machine's address once and connections authenticate automatically. That's fine for a fixed office server, less fine for Lambda or Cloud Run, where your egress IP changes and you'd be re-whitelisting constantly.

The dashboard's proxy generator exports endpoints as `.txt` or `.csv` and ships ready-made code samples in several languages, which saves the usual twenty minutes of formatting guesswork.

## Pricing: which package actually fits a rotating workflow

Rotation is cheap on 9Proxy in one very specific way: GB-based plans were explicitly excluded from the provider's June 1 pricing adjustment, while IP-based and bundle packages went up. If your workload is high-rotation with small responses — the classic rotating proxy API pattern — you're buying in the part of the catalog that didn't change.

All prices below are one-off, prepaid balance. There's no recurring subscription.

| Type | Package | What you get | Price | Validity | Get it |
| --- | --- | --- | --- | --- | --- |
| IP-based | 100 IPs | Unlimited bandwidth per IP | $24 ($0.24/IP) | IPs never expire | Grab the 100 IP package |
| IP-based | 500 IPs | Unlimited bandwidth per IP | $72 ($0.144/IP) | IPs never expire | Grab the 500 IP package |
| IP-based | 1,000 IPs + 500 bonus | 1,500 IPs total, unlimited bandwidth | $126 | IPs never expire | Grab the 1,000 + 500 IP package |
| IP-based | 2,500 IPs | Unlimited bandwidth per IP | $210 | IPs never expire | Grab the 2,500 IP package |
| IP-based | 5,000 IPs | Unlimited bandwidth per IP | $360 | IPs never expire | Grab the 5,000 IP package |
| IP-based | 15,000 IPs | Unlimited bandwidth per IP | $720 | IPs never expire | Grab the 15,000 IP package |
| IP-based | 25,000 IPs | Unlimited bandwidth per IP | $863 | IPs never expire | Grab the 25,000 IP package |
| IP-based | 50,000 IPs | Unlimited bandwidth per IP | $1,438 | IPs never expire | Grab the 50,000 IP package |
| Business IP | 100,000 IPs | High-volume IP allocation | $2,300 | IPs never expire | Grab the 100,000 IP package |
| Business IP | 200,000 IPs | High-volume IP allocation | $4,140 | IPs never expire | Grab the 200,000 IP package |
| Business IP | 500,000 IPs | High-volume IP allocation | $8,625 | IPs never expire | Grab the 500,000 IP package |
| GB-based | 5 GB | Unlimited endpoints, 20M+ IP pool | $15 ($3.00/GB) | 180 days | Start with 5 GB |
| GB-based | 50 GB + 5 GB bonus | Unlimited endpoints | $105 ($2.10/GB) | 180 days | Grab the 50 + 5 GB package |
| GB-based | 100 GB | Unlimited endpoints | $150 ($1.50/GB) | 180 days | Grab the 100 GB package |
| GB-based | 200 GB | Unlimited endpoints | $200 ($1.00/GB) | 180 days | Grab the 200 GB package |
| GB-based | 1,000 GB | Unlimited endpoints | $800 ($0.80/GB) | 180 days | Grab the 1,000 GB package |
| GB-based | 2,000 GB | Unlimited endpoints | $1,500 ($0.75/GB) | 180 days | Grab the 2,000 GB package |
| Enterprise GB | 3,000 GB | Team mode, shared traffic, activity logs | $2,160 ($0.72/GB) | No expiry | Grab the 3,000 GB Enterprise package |
| Enterprise GB | 6,000 GB | Team mode, shared traffic, activity logs | $4,200 ($0.70/GB) | No expiry | Grab the 6,000 GB Enterprise package |
| Enterprise GB | 10,000 GB | Team mode, shared traffic, activity logs | $6,800 ($0.68/GB) | No expiry | Grab the 10,000 GB Enterprise package |
| Bundle | Starter — 100 IPs + 5 GB | IPs for sticky work, GB for rotation | $30 | Mixed | Grab the Starter bundle |
| Bundle | Popular — 1,500 IPs + 50 GB | IPs for sticky work, GB for rotation | $180 | Mixed | Grab the Popular bundle |
| Bundle | Pro — 5,000 IPs + 500 GB | IPs for sticky work, GB for rotation | $720 | Mixed | Grab the Pro bundle |

### GB-based vs IP-based: pick by request shape

The billing model should follow what your log files look like, not what sounds cheaper per unit.

**GB-based wins when** each request pulls a small payload and the IP needs to change constantly. Search result pages, price checks, geo-verification, API polling — a few hundred kilobytes per request, thousands of requests, and the per-GB cost is what you actually pay. At $15 for the 5 GB entry package, a rotation workflow that consumes 200 MB a day still runs for weeks. The 180-day window matters too, because uneven project schedules don't force you to burn unused traffic inside a month.

**IP-based wins when** traffic volume is unpredictable or large. Pay per IP, get unlimited bandwidth on that IP, and a 4K video pull costs the same as a text file. Unused IPs never expire, so bulk buying ahead of a campaign doesn't decay on a timer.

The tradeoff to be honest about: IP-based plans don't rotate the way GB plans do. Each IP lives somewhere from a few hours up to around 24 hours, and rotation is handled through the Auto Rotation Proxy feature, which cycles IPs at intervals you configure on selected ports. That requires the desktop app — the IP-based product runs through local port forwarding, whereas GB-based proxies work straight from the dashboard. If "rotating proxy API" in your head means "one endpoint, new IP every request, no local software," that's the GB product, full stop.

## Bundles and Enterprise

Bundles exist for people running both shapes at once: sticky IPs for account-bound work and a GB pool for the rotation-heavy half. Starter is $30 for 100 IPs plus 5 GB, Popular is $180 for 1,500 IPs plus 50 GB, and Pro is $720 for 5,000 IPs plus 500 GB. If your workload is purely rotating, a bundle is extra machinery — the GB pool alone does the job.

Enterprise GB packages remove the 180-day clock, which changes the math for always-on infrastructure. They also add team mode for one owner plus up to five members, with shared bandwidth that doesn't expire inside the team, per-member traffic controls, full activity logs, and unlimited share code creation. External shares still follow the standard 180-day rule, which is an easy detail to miss when you're handing pool access to a contractor.

## Gotchas worth knowing before you buy

- **Over-filtering shrinks your pool.** Country-level targeting is fastest. State plus city plus ISP can leave you waiting for a matching IP.
- **`ssid` isn't decorative.** Parallel sticky threads sharing one session ID will share one IP.
- **Test on the sandbox domain.** There's a separate API base for development, so you don't have to rehearse against live traffic.
- **Old reviews are stale.** Plenty of comparison pages still describe 9Proxy as fixed-IP only. Check the current docs against the product you're actually buying.
- **The vendor advertises 20M+ residential IPs across 90+ countries and 99.95% uptime.** Those are 9Proxy's own figures, and independent benchmarks would be the way to verify them for your own targets.

If you're starting from zero, the practical path is to buy the 5 GB package, wire the rotating username into your existing HTTP client, and measure. That tells you your real cost per thousand requests within a day — which is the only number that decides whether the GB model or an IP package is right for you.

👉 [Start with the 5 GB rotating package and test it against your own targets](https://bit.ly/9-Proxy)
