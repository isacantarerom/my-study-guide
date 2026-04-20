# 🟡 Monitor Client-Side Errors

> *"Your servers can be perfectly healthy while your users are experiencing a broken product. Client-side monitoring is the only way to know."*

**⏱ Reading time:** ~10 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Observability

---

## Table of Contents

1. [What Client-Side Error Monitoring Does](#1-what-client-side-error-monitoring-does)
2. [The Visibility Gap — Why Server-Side Isn't Enough](#2-the-visibility-gap--why-server-side-isnt-enough)
3. [Types of Client-Side Errors](#3-types-of-client-side-errors)
4. [How Client-Side Errors Are Captured](#4-how-client-side-errors-are-captured)
5. [Real User Monitoring (RUM)](#5-real-user-monitoring-rum)
6. [Crash Reporting for Mobile Apps](#6-crash-reporting-for-mobile-apps)
7. [Frontend Performance Metrics](#7-frontend-performance-metrics)
8. [Privacy and Data Sensitivity](#8-privacy-and-data-sensitivity)
9. [How Client-Side Monitoring Connects to Other Building Blocks](#9-how-client-side-monitoring-connects-to-other-building-blocks)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. What Client-Side Error Monitoring Does

Your API is returning 200 OK. Your servers show no errors. Your monitoring dashboards look healthy. But users are reporting the checkout button doesn't work.

This is the client-side error monitoring problem. The error exists in JavaScript running in the user's browser — or in your mobile app on the user's device. Your server never knows about it. No log entry is created. No metric is incremented. The problem is completely invisible to your server-side observability stack.

Client-side monitoring captures errors, performance issues, and behavioral anomalies that occur on users' devices — before those issues ever make a network request to your server, or when the network request itself is the problem.

---

## 2. The Visibility Gap — Why Server-Side Isn't Enough

Consider the scenarios your server-side monitoring completely misses:

```
JavaScript crash before a request is made:
  User clicks "Add to Cart"
  JS error: "Cannot read property 'id' of undefined"
  No request ever sent to server
  Server sees: nothing
  User sees: button doesn't work

Network failure on the client:
  User's WiFi drops mid-transaction
  Request never reaches server
  Server sees: nothing
  User sees: spinner forever, then error

Browser compatibility issue:
  Works in Chrome, crashes in Safari
  Server sees: fewer Safari requests (users give up)
  Monitoring: "Safari traffic is low today" (misread as normal variation)

Slow JavaScript rendering:
  Server responds in 50ms
  Client takes 8 seconds to render the response (JS bundle too large)
  Server metric: 50ms latency (looks great)
  User experience: 8 second wait (feels broken)

Third-party script failure:
  Payment widget from Stripe fails to load
  User can't enter credit card
  Your server: never involved, sees no errors
  User: can't pay
```

In each case, the server is fine. The user experience is broken. Only client-side monitoring catches these.

---

## 3. Types of Client-Side Errors

### JavaScript Errors (Browser)
Uncaught exceptions and runtime errors in browser JavaScript.

```javascript
// Common JavaScript errors
TypeError: Cannot read property 'user' of null
ReferenceError: analyticsTracker is not defined
SyntaxError: Unexpected token '<'  (server returned HTML instead of JSON)
RangeError: Maximum call stack size exceeded  (infinite recursion)
```

These can happen because of:
- Bugs in your own code
- Browser compatibility issues (API not available in older browsers)
- Third-party script failures (analytics, ads, payment widgets)
- Race conditions in async code
- API responses with unexpected shapes

### Network Errors
Failed HTTP requests from the client perspective.

```javascript
fetch('/api/orders/123')
  .catch(err => {
    // Network errors that the server never sees:
    // - CORS errors (browser blocks the request)
    // - DNS resolution failures
    // - Connection timeouts
    // - SSL certificate errors
    // - Request aborted by user navigating away
  })
```

### Resource Load Failures
Scripts, stylesheets, or images that fail to load.

```
Failed to load: https://cdn.yourapp.com/bundle.v3.js
Failed to load: https://fonts.googleapis.com/css?family=Inter
Failed to load: https://yourapp.com/images/hero.jpg
```

If your main JavaScript bundle fails to load, your entire application is broken — but your server served a 200 for the HTML that references the bundle.

### Console Errors
Warnings and errors logged to the browser console — not crashes, but symptoms of problems.

```
[Warning] Mixed content: page served over HTTPS but includes HTTP resource
[Error] Content Security Policy violation: blocked 'inline' script
[Warning] Deprecated API usage: webkitRequestAnimationFrame
```

---

## 4. How Client-Side Errors Are Captured

### Browser SDK Integration
A JavaScript library included in your web application automatically captures unhandled errors and sends them to an error monitoring service.

```html
<script src="https://browser.sentry-cdn.com/7.x.x/bundle.min.js"></script>
<script>
  Sentry.init({
    dsn: "https://your-key@sentry.io/project-id",
    environment: "production",
    release: "v2.4.1",  // links errors to specific code versions
  });
</script>
```

Once initialized, the SDK:
- Hooks into `window.onerror` to capture unhandled exceptions
- Hooks into `window.onunhandledrejection` to capture unhandled Promise rejections
- Intercepts `fetch` and `XMLHttpRequest` to capture network errors
- Records user actions before the error (breadcrumbs) for context

### Explicit Error Capture
For handled errors you still want to track:

```javascript
try {
  const result = await processPayment(order);
} catch (error) {
  Sentry.captureException(error, {
    extra: {
      order_id: order.id,
      payment_method: order.payment_method,
    }
  });
  showErrorMessage("Payment failed. Please try again.");
}
```

### Breadcrumbs
A trail of user actions leading up to an error — what the user did before it crashed:

```
Breadcrumbs for error at 10:32:47:
  10:32:40  navigation  → user navigated to /checkout
  10:32:41  click       → clicked "Proceed to Payment"
  10:32:42  xhr         → GET /api/cart/123 → 200 OK
  10:32:44  xhr         → GET /api/user/profile → 200 OK
  10:32:46  click       → clicked "Apply Promo Code"
  10:32:47  error       → TypeError: Cannot read property 'discount' of null
```

Breadcrumbs answer "what was the user doing?" — often the most useful debugging context.

---

## 5. Real User Monitoring (RUM)

Beyond error capture, **Real User Monitoring (RUM)** collects performance data from actual users' browsers — not synthetic tests run in a lab.

```
For each page load, RUM collects:
  Time to First Byte (TTFB):    how long until the first byte arrives
  First Contentful Paint (FCP): how long until something is visible
  Largest Contentful Paint (LCP): how long until main content is visible
  Time to Interactive (TTI):    how long until user can interact
  Cumulative Layout Shift (CLS): how much the page jumps around while loading
  First Input Delay (FID):      how long until first user interaction is responsive
```

These are the **Core Web Vitals** — Google's standardized metrics for user experience quality, and a factor in search rankings.

RUM segments performance by:
- **Geography** — users in Asia might experience 3× longer load times than US users
- **Device type** — mobile users on 4G experience pages differently than desktop users on fiber
- **Browser** — a performance regression might only affect Chrome 119
- **Connection speed** — slow connections reveal what fast connections hide

```
Without RUM: "Our page loads in 1.2 seconds" (lab measurement)
With RUM: 
  US desktop / fiber: 0.8 seconds
  US mobile / 4G:     2.1 seconds
  Southeast Asia / mobile: 6.4 seconds  ← invisible without RUM
```

---

## 6. Crash Reporting for Mobile Apps

Mobile apps have a specific failure mode: crashes. Unlike a browser where an error might show a broken page, a mobile app crash closes the application entirely. The user has to reopen it. This is a severe experience failure.

Mobile crash reporting captures:
- The exception type and stack trace
- Device model and OS version
- App version
- Available memory at crash time
- Foreground/background state
- User actions leading up to the crash (session breadcrumbs)

```
Crash Report:
  Error: NullPointerException in CartViewModel.updateQuantity()
  App version: 4.2.1
  OS: Android 13
  Device: Samsung Galaxy S23
  Memory: 1.2 GB available of 8 GB
  Session: user was on cart screen for 45 seconds
  Breadcrumbs:
    - opened app
    - navigated to cart
    - changed item quantity from 2 to 3
    - CRASH
```

**Crash-free rate** is the key mobile health metric: what percentage of sessions end without a crash? A crash-free rate below 99.5% is generally considered poor for a production app.

---

## 7. Frontend Performance Metrics

Beyond errors, several performance metrics reveal frontend health:

**JavaScript Bundle Size** — larger bundles take longer to download and parse. Track bundle size over time; alert on significant increases (a new dependency was added that's much larger than expected).

**API Response Times (from client)** — the client measures how long its requests take end-to-end, including network transit. This differs from server-measured response time. The gap reveals network latency experienced by users.

```
Server-measured response time: 45ms
Client-measured response time: 280ms
Gap: 235ms of network transit time
→ Users in this region are far from your servers
→ Consider adding a CDN or regional endpoint
```

**Session Replay** — recording actual user sessions (mouse movements, clicks, scrolls, network requests) for debugging. When an error occurs, you can watch a replay of exactly what the user did. Privacy-sensitive — requires careful data handling and user consent.

---

## 8. Privacy and Data Sensitivity

Client-side monitoring presents unique privacy challenges because you're collecting data from users' browsers.

**What you must handle carefully:**
- Personal data in URLs (user IDs, email addresses in query strings)
- Form input data (users might type passwords in the wrong field)
- Credit card numbers accidentally logged
- Healthcare or financial data

**Best practices:**
- Scrub sensitive fields before sending to monitoring services
- Use data masking in session replay (blur form inputs)
- Respect Do Not Track browser settings
- Include monitoring in your privacy policy
- Use data residency controls if operating in regulated regions (GDPR, CCPA)

```javascript
Sentry.init({
  beforeSend(event) {
    // Scrub sensitive data before sending
    if (event.request?.data) {
      delete event.request.data.password;
      delete event.request.data.card_number;
    }
    return event;
  }
});
```

---

## 9. How Client-Side Monitoring Connects to Other Building Blocks

```
Server-Side Error Monitoring ────────────────────────────────────────────►
  Client errors often have a corresponding server error.
  Request IDs link client-side errors to server-side traces.
  Together they give complete visibility across the stack.

Distributed Tracing ─────────────────────────────────────────────────────►
  Client-side monitoring can initiate a trace.
  The trace ID flows from browser → API gateway → all backend services.
  Full end-to-end visibility for a single user interaction.

CDN ─────────────────────────────────────────────────────────────────────►
  Client-side resource load failures reveal CDN issues.
  If JS bundle fails to load → CDN miss or CDN outage.
  Server-side monitoring wouldn't see this (server served the HTML fine).

Distributed Monitoring ──────────────────────────────────────────────────►
  Client-side RUM metrics complement server-side metrics.
  Server: "latency is 50ms" — Client: "users wait 4 seconds"
  The gap reveals client-side rendering performance issues.
```

---

## 10. Self-Check

1. Why is server-side monitoring insufficient for catching all errors? Give two examples of errors that would be invisible to server-side monitoring.
2. What are breadcrumbs, and why are they useful for debugging client-side errors?
3. What is RUM, and how is it different from server-side latency metrics?
4. What is a crash-free rate, and what threshold is generally considered acceptable?
5. A JavaScript error is occurring for 5% of Safari users but 0% of Chrome users. Your server-side metrics look completely normal. How would you discover and diagnose this without client-side monitoring?
6. You add client-side error monitoring and immediately see 200 errors per minute. Your product manager asks if the app suddenly got worse. How do you explain this?
7. Why must client-side error monitoring be designed with privacy in mind? What types of data need to be scrubbed before sending to a monitoring service?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🔧 [Sentry Browser SDK](https://docs.sentry.io/platforms/javascript/) | The most widely used client-side error monitoring SDK |
| 📊 [Web Vitals Documentation](https://web.dev/vitals/) | Google's core web vitals explained — the metrics that matter for user experience |
| 📬 [ByteByteGo — Observability](https://bytebytego.com) | How client and server observability fit together |
| 📝 [MDN — Window.onerror](https://developer.mozilla.org/en-US/docs/Web/API/Window/onerror) | How browser error capture works at the platform level |

---

*⬅️ Previous: [Monitor Server-Side Errors](monitor-server-side-errors.md) &nbsp;|&nbsp; ➡️ Next: [Distributed Logging](distributed-logging.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Observability.</sub>