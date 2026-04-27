# Real User Monitoring (RUM) — Study Notes

---

## What is Real User Monitoring (RUM)?

RUM is a performance monitoring technique that **captures and analyzes data from actual users' devices and browsers** as they interact with your application — in real time.

> Unlike server-side monitoring (where you check logs on your backend), RUM focuses on what's happening **on the client/user side**.

---

## The Problem RUM Solves

When your application runs on users' devices, many issues can occur that you simply **cannot see from the server**:

| Problem | Why You Can't See It Server-Side |
|---|---|
| App crashes on old browsers | Server only knows the request was made |
| High latency for users in remote locations | Server response time ≠ user experience time |
| Slow device / low RAM | Happens entirely on client |
| Poor network (e.g., 3G) | Network condition is client-side |
| JavaScript errors / console errors | Run in the browser, not on server |
| Memory leaks | Browser-level issue |

**Core insight:** Users complain your app is broken, but your server logs show everything is fine. The problem exists *between* the server and the user's screen.

---

## How RUM Works — The Technical Flow

```
User's Browser
     │
     ├── App runs (React / Next.js / vanilla JS / etc.)
     │
     ├── Events occur:
     │     - Console errors
     │     - Network call failures
     │     - Slow API responses
     │     - Memory leaks
     │     - Long blocking tasks
     │     - Route changes
     │
     ▼
Beacon API  ──────────────────────────────────►  Monitoring Backend
(non-blocking, async)                            (aggregates & displays data)
```

---

## The Beacon API

A **native browser JavaScript API** used by RUM tools under the hood.

```js
navigator.sendBeacon(url, data);
```

**Key characteristics:**
- Sends analytics/telemetry data to a backend endpoint
- **Non-blocking** — does not hold up the main JavaScript thread
- Fires asynchronously, even when the page is closing/unloading
- Ideal for sending error data, performance metrics, and events without affecting UX

**Why non-blocking matters:** You never want your monitoring code to slow down the actual application. The Beacon API ensures monitoring is a background task.

---

## What RUM Tracks

### 1. Network & API Calls
- Which APIs were called
- Response time per API
- DNS lookup time
- Connection time
- Redirect time
- Failure rate (e.g., 4xx, 5xx errors)
- CORS errors

### 2. Web Vitals
- **FCP** — First Contentful Paint (when first content appears)
- **FID** — First Input Delay (time before browser responds to first interaction)
- **LCP** — Largest Contentful Paint
- **CLS** — Cumulative Layout Shift

### 3. JavaScript / Browser Errors
- `console.error` and `console.warn` logs
- Uncaught exceptions
- Promise rejections

### 4. Performance Issues
- **Long tasks** — JS blocking the main thread (e.g., a 3-second synchronous operation freezes the UI)
- **Memory leaks** — heap memory growing without being released
- Slow renders / excessive re-renders

### 5. User Session Data
- Geographic location of users
- ISP / network type (e.g., Jio, Airtel)
- Browser and browser version
- Device type
- Active sessions count

### 6. Route Changes (for SPAs)
- Tracks navigation between pages in Single Page Applications
- Per-page performance stats (e.g., `/checkout` vs `/profile`)

### 7. Session Replays *(advanced)*
- Records the actual screen interaction of users
- Lets you "replay" what a user did before an error occurred

---

## RUM vs Server-Side Monitoring

| Aspect | Server-Side Monitoring | Real User Monitoring |
|---|---|---|
| Where data comes from | Your server logs | User's browser |
| What it catches | Backend errors, DB issues | JS errors, slow UIs, network issues |
| Visibility into user device | ❌ None | ✅ Full |
| Real user experience | ❌ Indirect | ✅ Direct |
| Tools used | Log aggregators, APM | Beacon API + RUM platforms |

---

## RUM in Single Page Applications (SPAs)

SPAs (React, Next.js, Vue, etc.) don't do full page reloads — they change routes client-side. RUM tools track:
- **Virtual route changes** (e.g., `/home` → `/checkout`)
- Performance of each "page" independently
- API calls scoped to each route

---

## Implementation — How Simple It Is

RUM is typically enabled by **adding a single `<script>` tag** to your HTML `<head>`:

```html
<head>
  <!-- Your RUM provider's script -->
  <script src="https://your-rum-provider.com/agent.js" data-app-id="YOUR_ID"></script>
</head>
```

That single line enables:
- Automatic capture of web vitals
- API call tracking
- Console error logging
- Session tracking
- Geographic and browser data

---

## Key Concepts Summary

```
RUM
 ├── Data Source: User's browser (client-side)
 ├── Transport: Beacon API (async, non-blocking)
 ├── Tracks:
 │    ├── Web Vitals (FCP, FID, LCP, CLS)
 │    ├── API call performance & failures
 │    ├── JS errors & console logs
 │    ├── Long tasks & memory leaks
 │    ├── User geography & browser info
 │    └── Route changes (SPA navigation)
 └── Output: Aggregated dashboard with session data
```

---

## Why RUM Matters at Scale

- Large-scale apps have users across **different geographies, networks, and devices**
- A bug that only affects Chrome on 3G in Tier-2 cities will **never show up in server logs**
- RUM gives you **real-world performance data**, not just synthetic benchmarks
- Helps you **prioritize fixes** based on actual user impact

---

## Quick Revision Checklist

- [ ] RUM monitors the **client side**, not the server side
- [ ] Uses the **Beacon API** to send data asynchronously
- [ ] Tracks **Web Vitals**, API performance, JS errors, memory, and sessions
- [ ] Essential for **SPA route-level** performance tracking
- [ ] **Session replays** let you watch what users actually experienced
- [ ] Setup is typically **one script tag** in your HTML head
- [ ] Data aggregates over time — more meaningful after 2–3 days of real usage
