---
name: obscura
description: Use when scraping the web or driving headless browser automation from Claude Code. Obscura is a Rust-based, drop-in headless Chrome replacement (~30 MB) compatible with Puppeteer and Playwright via the Chrome DevTools Protocol. Trigger when the user mentions web scraping, headless browser, Puppeteer/Playwright, anti-bot/anti-detection, CDP, JS rendering, parallel page fetching, or `obscura`/`obscura serve`/`obscura fetch`.
---

# Obscura — Headless Browser for AI Agents and Web Scraping

> **Source:** https://github.com/h4ckf0r0day/obscura
> **License:** Apache 2.0

## Overview

Obscura is an open-source headless browser engine written in Rust. It runs real
JavaScript via V8, speaks the Chrome DevTools Protocol, and works as a
drop-in replacement for headless Chrome with Puppeteer and Playwright — but
uses ~30 MB of memory instead of 200+ MB and starts instantly.

Use this skill whenever you need to:

- Scrape JavaScript-heavy pages from the CLI
- Drive Puppeteer or Playwright scripts without bundling Chromium
- Spin up a CDP server for an AI agent to control
- Defeat trivial bot-detection (built-in stealth + tracker blocking)
- Parallel-fetch many URLs with low memory overhead

## When to Use

| Trigger | Action |
|---|---|
| User wants to scrape one URL with JS rendering | `obscura fetch <url>` |
| User wants to scrape many URLs in parallel | `obscura scrape url1 url2 ...` |
| User has a Puppeteer/Playwright script | Start `obscura serve` and connect via CDP |
| Page is bot-protected | Add `--stealth` |
| User asks about anti-detect / fingerprinting | Recommend stealth build |

## Installation

### macOS (Apple Silicon)

```bash
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-aarch64-macos.tar.gz
tar xzf obscura-aarch64-macos.tar.gz
sudo mv obscura /usr/local/bin/
obscura --version
```

### macOS (Intel)

```bash
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-x86_64-macos.tar.gz
tar xzf obscura-x86_64-macos.tar.gz
sudo mv obscura /usr/local/bin/
```

### Linux x86_64

```bash
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-x86_64-linux.tar.gz
tar xzf obscura-x86_64-linux.tar.gz
sudo mv obscura /usr/local/bin/
```

### Windows

Download the `.zip` from the [Releases](https://github.com/h4ckf0r0day/obscura/releases)
page and extract it. Add the binary to `PATH`.

### Build from source (with stealth)

```bash
git clone https://github.com/h4ckf0r0day/obscura.git
cd obscura
cargo build --release --features stealth
# Binary: ./target/release/obscura
```

Requires Rust 1.75+ ([rustup.rs](https://rustup.rs)). First build takes ~5 min
because V8 compiles from source — subsequent builds are cached.

### Verify

```bash
obscura fetch https://example.com --eval "document.title"
# Expected output: "Example Domain"
```

## Usage

### 1. Fetch a single page

```bash
# Get the page title
obscura fetch https://example.com --eval "document.title"

# Dump the rendered HTML (after JS executes)
obscura fetch https://news.ycombinator.com --dump html

# Dump only the links
obscura fetch https://example.com --dump links

# Dump plain text
obscura fetch https://example.com --dump text

# Wait for network to be idle before reading
obscura fetch https://example.com --wait-until networkidle0

# Wait for a specific element
obscura fetch https://example.com --selector ".article-body"
```

`--dump` accepts: `html`, `text`, `links`.
`--wait-until` accepts: `load`, `domcontentloaded`, `networkidle0`.

### 2. Scrape many URLs in parallel

```bash
obscura scrape \
  https://example.com/page-1 \
  https://example.com/page-2 \
  https://example.com/page-3 \
  --concurrency 25 \
  --eval "document.querySelector('h1').textContent" \
  --format json
```

`--format` accepts: `json` or `text`. Use `json` when piping into `jq`.

### 3. Start a CDP server for Puppeteer / Playwright

```bash
obscura serve --port 9222

# With anti-detection + tracker blocking
obscura serve --port 9222 --stealth

# Through an HTTP/SOCKS5 proxy
obscura serve --port 9222 --proxy socks5://127.0.0.1:1080

# More worker processes for higher throughput
obscura serve --port 9222 --workers 4
```

Then connect from Node:

**Puppeteer:**

```javascript
import puppeteer from 'puppeteer-core';

const browser = await puppeteer.connect({
  browserWSEndpoint: 'ws://127.0.0.1:9222/devtools/browser',
});

const page = await browser.newPage();
await page.goto('https://news.ycombinator.com');

const stories = await page.evaluate(() =>
  Array.from(document.querySelectorAll('.titleline > a'))
    .map(a => ({ title: a.textContent, url: a.href }))
);
console.log(stories);

await browser.disconnect();
```

**Playwright:**

```javascript
import { chromium } from 'playwright-core';

const browser = await chromium.connectOverCDP({
  endpointURL: 'ws://127.0.0.1:9222',
});

const ctx = await browser.newContext();
const page = await ctx.newPage();
await page.goto('https://en.wikipedia.org/wiki/Web_scraping');
console.log(await page.title());

await browser.close();
```

### 4. Form submission & login

Obscura handles POSTs, follows 302 redirects, and maintains cookies natively.

```javascript
await page.goto('https://quotes.toscrape.com/login');
await page.evaluate(() => {
  document.querySelector('#username').value = 'admin';
  document.querySelector('#password').value = 'admin';
  document.querySelector('form').submit();
});
```

## Stealth Mode

Build with `--features stealth` (or use the stealth release binary) and run
with `--stealth`.

What it does:

- Per-session fingerprint randomization (GPU, screen, canvas, audio, battery)
- Realistic `navigator.userAgentData` (Chrome 145, high-entropy values)
- `event.isTrusted = true` for dispatched events
- Hidden internal properties (`Object.keys(window)` is safe)
- Native function masking (`Function.prototype.toString()` returns `[native code]`)
- `navigator.webdriver = undefined` (matches real Chrome)
- Blocks 3,520 tracker / analytics / fingerprinting domains

## CLI Reference Cheat Sheet

### `obscura serve`

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | `9222` | WebSocket port |
| `--proxy` | — | HTTP/SOCKS5 proxy URL |
| `--stealth` | off | Anti-detection + tracker blocking |
| `--workers` | `1` | Parallel worker processes |
| `--obey-robots` | off | Respect robots.txt |

### `obscura fetch <URL>`

| Flag | Default | Description |
|------|---------|-------------|
| `--dump` | `html` | `html` \| `text` \| `links` |
| `--eval` | — | JS expression to evaluate |
| `--wait-until` | `load` | `load` \| `domcontentloaded` \| `networkidle0` |
| `--selector` | — | Wait for CSS selector |
| `--stealth` | off | Anti-detection mode |
| `--quiet` | off | Suppress banner |

### `obscura scrape <URL...>`

| Flag | Default | Description |
|------|---------|-------------|
| `--concurrency` | `10` | Parallel workers |
| `--eval` | — | JS expression per page |
| `--format` | `json` | `json` \| `text` |

## CDP Coverage

Obscura implements the Chrome DevTools Protocol surface needed by
Puppeteer / Playwright:

| Domain | Methods |
|--------|---------|
| Target | createTarget, closeTarget, attachToTarget, createBrowserContext, disposeBrowserContext |
| Page | navigate, getFrameTree, addScriptToEvaluateOnNewDocument, lifecycleEvents |
| Runtime | evaluate, callFunctionOn, getProperties, addBinding |
| DOM | getDocument, querySelector, querySelectorAll, getOuterHTML, resolveNode |
| Network | enable, setCookies, getCookies, setExtraHTTPHeaders, setUserAgentOverride |
| Fetch | enable, continueRequest, fulfillRequest, failRequest |
| Storage | getCookies, setCookies, deleteCookies |
| Input | dispatchMouseEvent, dispatchKeyEvent |
| LP | getMarkdown (DOM-to-Markdown) |

## Decision Heuristics

When the user asks for web automation, choose this way:

1. **One page, one shot** → `obscura fetch <url> --eval "..."`
2. **Many pages, same selector** → `obscura scrape <urls> --concurrency 25`
3. **Stateful flow, login, multi-step** → `obscura serve` + Puppeteer/Playwright
4. **Page detects bots** → add `--stealth`
5. **Behind a proxy** → `--proxy <url>`
6. **CI / Docker** → use the static Linux binary, no Chrome needed

## Anti-Patterns

- Do **not** use Obscura against sites whose terms of service forbid scraping.
- Do **not** disable `--obey-robots` on third-party sites in production
  pipelines without consent.
- Do **not** treat stealth mode as a bypass for paywalls or auth — it only
  hides the fact that the browser is automated, not the fact that requests
  are made.
- Do **not** spawn `obscura fetch` in a tight shell loop for many URLs — use
  `obscura scrape` (worker pool) instead.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `connection refused` from Puppeteer | Server not running | `obscura serve --port 9222` first |
| Page renders empty HTML | JS hasn't finished | Add `--wait-until networkidle0` |
| Site detects automation | webdriver leak | Build with `--features stealth`, run with `--stealth` |
| Build fails on `v8` | Rust < 1.75 | `rustup update stable` |
| Slow first build | V8 compiling | Expected ~5 min, cached after |

## References

- Repository: https://github.com/h4ckf0r0day/obscura
- Releases (binaries): https://github.com/h4ckf0r0day/obscura/releases
- Chrome DevTools Protocol: https://chromedevtools.github.io/devtools-protocol/
- Puppeteer: https://pptr.dev/
- Playwright: https://playwright.dev/
