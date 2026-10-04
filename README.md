# python requests proxy: Auth, Rotation, SOCKS5 and Retry Handling Explained, With 9Proxy's Full Plan Prices

A script that works on your laptop and dies in production is usually not a scraping-logic problem. It is an IP problem wearing a different costume: a 403 that shows up after request number 60, a `ProxyError` that only appears on your server, a 407 you have never seen before and have to look up.

The `requests` library handles proxies with a dictionary and about three lines of code, which is exactly why so many people get it slightly wrong. Below is the plumbing that matters — scheme prefixes, auth encoding, SOCKS5 DNS handling, retries — plus a look at 9Proxy's two residential billing models, because they map onto two completely different ways of wiring a proxy into a Python script.

## The four lines that do most of the work

python
import requests

PROXY = "http://user:pass@gate.example.com:8000"
proxies = {"http": PROXY, "https": PROXY}

r = requests.get("https://api.ipify.org?format=json", proxies=proxies, timeout=(5, 20))
print(r.json())


If the JSON says anything other than your home IP, the tunnel is up. That is the whole integration. Everything after this point is about keeping it up.

The `proxies` dict keys (`http`, `https`) describe **the scheme of the destination URL**, not the type of proxy you bought. A single residential endpoint handles both, so you point both keys at the same address. This is the source of the most common confusion in the library: people assume `"https"` means "an HTTPS proxy" and go hunting for a TLS-terminating endpoint they don't need.

## The `MissingSchema` trap

In requests 1.x you could write `{"http": "10.10.1.10:3128"}` and it worked. Since 2.x that raises `requests.exceptions.MissingSchema`. The proxy value needs a scheme like any other URL:

python
proxies = {"http": "http://10.10.1.10:3128", "https": "http://10.10.1.10:3128"}


Note that a plain HTTP proxy endpoint still tunnels HTTPS traffic fine via `CONNECT`. You do not need `https://` in the proxy string, and using it can cause its own TLS negotiation problems with some providers.

## Credentials, and passwords that break your URL

Residential providers rarely hand you IP-whitelisted access for a crawler running from a laptop with a changing IP, so you will be using `user:pass@host:port`. The failure mode nobody expects: a password containing `@`, `:`, `/`, `#` or `%` silently corrupts the URL, and you get a 407 that looks like a wrong password.

python
import os
from urllib.parse import quote

user = os.environ["PROXY_USER"]
pwd = quote(os.environ["PROXY_PASS"], safe="")

proxy_url = f"http://{user}:{pwd}@{os.environ['PROXY_HOST']}:{os.environ['PROXY_PORT']}"

session = requests.Session()
session.proxies.update({"http": proxy_url, "https": proxy_url})
session.trust_env = False   # ignore HTTP_PROXY / ALL_PROXY leaking in from the shell


Two details in there are worth keeping. `session.proxies` applies the tunnel to every request made through that session, so you stop repeating the argument in thirty call sites. And `trust_env = False` matters on machines where someone exported `HTTP_PROXY` for a corporate network; without it, requests will merge environment proxies with yours and you will spend an afternoon debugging an endpoint you never configured.

If you'd rather not hardcode anything, requests also reads `HTTP_PROXY`, `HTTPS_PROXY` and `ALL_PROXY` from the environment when `trust_env` is left on. Useful for a one-off script, risky for a scraper you'll hand to someone else.

## SOCKS5, and the difference between `socks5://` and `socks5h://`

requests does not support SOCKS out of the box. You need PySocks:

bash
pip install "requests[socks]"


Then:

python
proxies = {
    "http": "socks5h://user:pass@gate.example.com:1080",
    "https": "socks5h://user:pass@gate.example.com:1080",
}


Without PySocks you get `InvalidSchema: Missing dependencies for SOCKS support`, which is at least a clear error message.

The `h` is the part that matters. `socks5://` resolves the hostname on your machine and sends the IP to the proxy; `socks5h://` sends the hostname and lets the proxy resolve it. The second one avoids DNS leaks (your resolver still sees every domain you scrape, which defeats part of the point) and it handles hostnames that don't resolve locally. For anything production-facing, use `socks5h://`.

## Rotating or sticky: decide this before you pick a provider

There are two ways to get a new IP on every request, and they have different costs.

**Rotate inside your script.** Hold a list of endpoints, pick one per request, handle the failures yourself. Maximum control, more code, and you pay per IP or per port.

**Rotate at the provider.** One hostname and port, with session behaviour controlled by the username. This is the approach most residential networks use, and it removes the retry-the-whole-list logic from your codebase.

Sticky sessions are the other half of the decision. Rotating IPs are correct for price monitoring, SERP checks and bulk list pages. A sticky IP held for 10 to 30 minutes is what you need when a login, cart, or multi-step API flow has to look like one continuous visitor. Mixing the two up shows up as random 403s in the middle of a checkout flow that worked yesterday.

## Where 9Proxy fits into a `requests` setup

9Proxy is a residential network with 20M+ IPs across 90+ countries, HTTP/HTTPS/SOCKS5 support, and targeting down to country, state, city, ZIP and ISP. It sells access two ways, and the split is genuinely relevant to Python users rather than being a marketing distinction.

### Residential by GB: the one that behaves like a normal proxy

You buy a pool of traffic (5 GB up to 10,000 GB), get a hostname and port from the dashboard, and control everything from the username string:


<sub_user>-country-us-st-ohio-city-newyork-isp-as22773_Cox_Communications_Inc.-sst-15-ssid-worker3


Every parameter is optional except the sub-user name. Leave out `sst` and `ssid` and each request gets a fresh IP, which is the rotating mode. Add `sst-15` to hold an IP for 15 minutes, and add a unique `ssid` per worker to run parallel sticky sessions from the same configuration.

python
def sticky(sub_user, country, minutes, worker_id):
    return f"{sub_user}-country-{country}-sst-{minutes}-ssid-{worker_id}"

worker_user = sticky("acct01", "us", 15, "w7")
proxy_url = f"http://{worker_user}:{PASSWORD}@gate.host:8000"


Because authentication is username/password or an IP whitelist, this model drops straight into `proxies=`. No desktop software, no local port, works from a container. GB packages carry a 180-day validity window, and the three largest tiers (3,000 GB and up) never expire at all.

One practical tip from the provider's own docs: target country only when you can. Stacking state, city and ISP filters shrinks the available pool for that request, and you'll see more timeouts and retries than the filtering is worth.

### Residential by IPs: unlimited bandwidth, but a desktop app in the middle

The per-IP model is a fixed package of IPs with no bandwidth metering. You pay per IP, not per gigabyte, so pulling heavy HTML or downloading assets does not move your bill. Unused IPs never expire, and a forwarded IP stays live for a few hours up to roughly 24 hours.

The catch for Python users is the access path. These IPs are forwarded through the 9Proxy desktop app (Windows, macOS, Linux) to a local port, so your code connects to `localhost:port`, optionally with credentials appended:

python
proxies = {"http": "http://user:pass@127.0.0.1:9100", "https": "http://user:pass@127.0.0.1:9100"}


That's a clean setup on a workstation that runs the app. It's a poor fit for a headless server or a container you deploy from CI, where there's no app to forward anything. Pick this model when the script runs on a box you control and the workload is bandwidth-heavy. Pick the GB model when the script runs where you can't install a GUI client.

The app also carries an auto-refresh option that replaces an IP when it goes offline, and auto-rotation that swaps proxies on a schedule you set — useful if you'd rather not build that logic in Python.

You can 👉 [👉 open a 9Proxy account here](https://bit.ly/9-Proxy) and see both dashboards before paying, since registration itself is free.

## Surviving real traffic: timeouts, retries, 407s

The default `requests` behaviour — wait forever, fail once, crash — is a bad fit for anything with a proxy in front of it. Two additions cover most of it.

python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=4,
    connect=4,          # covers proxy connection failures
    read=2,
    backoff_factor=0.6,
    status_forcelist=(429, 500, 502, 503, 504),
    allowed_methods=("GET", "POST"),
)

adapter = HTTPAdapter(max_retries=retry, pool_connections=50, pool_maxsize=50)
session.mount("http://", adapter)
session.mount("https://", adapter)


Always pass a timeout as a tuple: `timeout=(5, 20)` means 5 seconds to connect, 20 seconds between bytes. A single value applies to both, which is usually not what you want when a residential hop adds latency to the handshake but the response comes back fast.

Do *not* let the retry policy paper over a 407. That status means the proxy rejected your credentials, and no amount of backoff fixes a typo in a sub-user password.

| Symptom | What it actually means | Fix |
| --- | --- | --- |
| `MissingSchema` / `InvalidURL` | Proxy value has no scheme | Add `http://` or `socks5h://` |
| `InvalidSchema: Missing dependencies for SOCKS support` | PySocks not installed | `pip install "requests[socks]"` |
| 407 Proxy Authentication Required | Bad or mangled credentials | Re-check sub-user/password, percent-encode special characters |
| `ProxyError` / `ConnectTimeout` | Wrong host/port, or the local port isn't running | Verify the endpoint in the dashboard, confirm the app is up for IP-based plans |
| 403 / 429 / CAPTCHA from the target | IP is fine, request pattern isn't | Rotate, slow down, send realistic headers |

## Threading without making a mess

`requests` is blocking, so concurrency means threads. The rule that saves the most debugging time: one `Session` per worker thread, not one shared globally.

python
from concurrent.futures import ThreadPoolExecutor

def worker(job_id):
    sess = requests.Session()
    sess.mount("https://", adapter)
    sess.proxies.update({"https": sticky_proxy(job_id)})
    return sess.get(url, timeout=(5, 20), headers=HEADERS).status_code

with ThreadPoolExecutor(max_workers=8) as pool:
    results = list(pool.map(worker, range(8)))


Each worker gets its own sticky `ssid`, so eight threads hold eight stable IPs instead of fighting over one, and each session reuses its own connection pool. Eight to twenty threads is a realistic range for residential endpoints; pushing past that on one hostname mostly produces timeouts that look like network failures.

## The full 9Proxy package list

Prices below reflect the June 1 adjustment to the IP-based and bundle tiers. GB-based pricing was left unchanged in that update, which is why the per-gigabyte floor still sits at $0.68.

All packages are one-time purchases against a balance rather than subscriptions, and IP balance doesn't expire.

| Package | Model | What you get | Price | Billing / validity | Get it |
| --- | --- | --- | --- | --- | --- |
| 100 IPs | By IP | 100 residential IPs, unlimited bandwidth | $0.24/IP — **$24** | One-time, IPs never expire | [ Buy 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | By IP | 500 residential IPs, unlimited bandwidth | $0.144/IP — **$72** | One-time, IPs never expire | [ Buy 500 IPs](https://bit.ly/9-Proxy) |
| 1,000 + 500 bonus IPs | By IP | 1,500 IPs for the price of 1,000 | $0.084/IP — **$126** | One-time, IPs never expire | [ Buy the 1,500 IP pack](https://bit.ly/9-Proxy) |
| 2,500 IPs | By IP | 2,500 residential IPs | $0.084/IP — **$210** | One-time, IPs never expire | [ Buy 2,500 IPs](https://bit.ly/9-Proxy) |
| 5,000 IPs | By IP | 5,000 residential IPs | $0.072/IP — **$360** | One-time, IPs never expire | [ Buy 5,000 IPs](https://bit.ly/9-Proxy) |
| 15,000 IPs | By IP | 15,000 residential IPs | $0.048/IP — **$720** | One-time, IPs never expire | [ Buy 15,000 IPs](https://bit.ly/9-Proxy) |
| 25,000 IPs | By IP | 25,000 residential IPs | $0.035/IP — **$863** | One-time, IPs never expire | [ Buy 25,000 IPs](https://bit.ly/9-Proxy) |
| 50,000 IPs | By IP | 50,000 residential IPs | $0.029/IP — **$1,438** | One-time, IPs never expire | [ Buy 50,000 IPs](https://bit.ly/9-Proxy) |
| 100,000 IPs | By IP (Business) | High-volume residential IP block | $0.023/IP — **$2,300** | One-time, IPs never expire | [ Request the 100k IP package](https://bit.ly/9-Proxy) |
| 200,000 IPs | By IP (Business) | High-volume residential IP block | $0.021/IP — **$4,140** | One-time, IPs never expire | [ Request the 200k IP package](https://bit.ly/9-Proxy) |
| 500,000 IPs | By IP (Business) | Lowest per-IP rate on the list | $0.018/IP — **$8,625** | One-time, IPs never expire | [ Request the 500k IP package](https://bit.ly/9-Proxy) |
| 5 GB | By GB | Rotating or sticky residential endpoints | $3.00/GB — **$15** | 180-day validity | [ Buy 5 GB](https://bit.ly/9-Proxy) |
| 50 + 5 bonus GB | By GB | 55 GB for the price of 50 | $2.10/GB — **$105** | 180-day validity | [ Buy the 55 GB pack](https://bit.ly/9-Proxy) |
| 100 GB | By GB | Rotating or sticky residential endpoints | $1.50/GB — **$150** | 180-day validity | [ Buy 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | By GB | Rotating or sticky residential endpoints | $1.00/GB — **$200** | 180-day validity | [ Buy 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | By GB | Rotating or sticky residential endpoints | $0.80/GB — **$800** | 180-day validity | [ Buy 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | By GB | Rotating or sticky residential endpoints | $0.75/GB — **$1,500** | 180-day validity | [ Buy 2,000 GB](https://bit.ly/9-Proxy) |
| 3,000 GB | By GB (Enterprise) | No expiry on the balance | $0.72/GB — **$2,160** | Balance never expires | [ Buy 3,000 GB](https://bit.ly/9-Proxy) |
| 6,000 GB | By GB (Enterprise) | No expiry on the balance | $0.70/GB — **$4,200** | Balance never expires | [ Buy 6,000 GB](https://bit.ly/9-Proxy) |
| 10,000 GB | By GB (Enterprise) | Lowest per-GB rate, no expiry | $0.68/GB — **$6,800** | Balance never expires | [ Buy 10,000 GB](https://bit.ly/9-Proxy) |
| Starter Bundle | Bundle | 100 IPs + 5 GB | **$30** | 180-day validity on the GB portion | [ Buy the Starter bundle](https://bit.ly/9-Proxy) |
| Popular Bundle | Bundle | 1,500 IPs + 50 GB | **$180** | 180-day validity on the GB portion | [ Buy the Popular bundle](https://bit.ly/9-Proxy) |
| Pro Bundle | Bundle | 5,000 IPs + 500 GB | **$720** | 180-day validity on the GB portion | [ Buy the Pro bundle](https://bit.ly/9-Proxy) |

A few things the table makes obvious. The step from 100 IPs to 500 IPs cuts your per-IP cost by 40%, and the 1,000+500 tier is where the bonus IPs do real work — you're effectively paying $0.084 for IPs that cost $0.20 each at the entry tier. On the GB side, moving from 5 GB to 50 GB takes the price from $3.00 to $2.10 per gigabyte for the same product, so buying the smallest package and topping up repeatedly is the expensive way to run.

## Picking a package for a `requests`-based crawler

If your script rotates IPs per request and pulls HTML pages of a few hundred kilobytes, per-GB is the honest answer: 100 GB at $150 covers a six-figure page count, and you get the 180-day window to burn it down. The sticky-username syntax also means rotation lives in a string, not in your retry logic.

If your workload moves a lot of bytes per IP — asset downloads, large JSON responses, video metadata passes — the per-IP model is cheaper, because bandwidth stops being a line item. Just make sure the script runs somewhere you can keep the 9Proxy app open. If it doesn't, don't buy this model; there's no way around the local port.

Bundles make sense in one specific case: you have a small number of sticky workflows (logins, carts, account checks) plus a broad rotating scrape, and you'd rather manage one balance than two. The Pro bundle at $720 is meaningfully cheaper than buying 5,000 IPs ($360) and 500 GB ($500) separately.

On discounts: 9Proxy publishes promo codes periodically, and the ones from earlier campaigns have expired. The discount that is currently baked into the page is the package bonus — the extra 500 IPs and the extra 5 GB. If a code is live, it appears automatically in your dashboard under My Coupons and applies at checkout. Signing up through an invite link is also how the referral discount attaches to a new account, so it's worth doing at registration rather than after your first purchase.

There is no self-serve free trial button on the site. Registration is free, and trials have historically been arranged case by case through the provider's community channels, so plan on buying the smallest package that covers your test run — $15 for 5 GB is a cheap way to find out whether their IPs work on your specific targets before you commit to a volume tier.

## Before you ship

- Verify the tunnel with `api.ipify.org` or `httpbin.org/ip` before you point the script at your real target.
- Set `timeout=(connect, read)` on every request. No exceptions.
- Confirm `ProxyError` handling exists, and that a 407 is treated as a credential bug rather than a transient failure.
- Use `socks5h://`, not `socks5://`, unless you have a reason to resolve DNS locally.
- Keep credentials in environment variables and percent-encode the password before it goes into the URL.
- Target country only when you can; every additional filter shrinks the pool of usable IPs.
- Run one thread with a sticky session first, then scale workers and watch where latency lands.

The library side of this takes an afternoon. The part that decides whether your crawler still works next month is which proxy model you bought and whether it matches where the script actually runs.

👉 [👉 Create your 9Proxy account and start with the smallest package that fits your test](https://bit.ly/9-Proxy)
