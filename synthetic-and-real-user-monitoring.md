---
title: CloudWatch Synthetics, RUM & Evidently
summary: Practical setup guide for CloudWatch's outside-in (Synthetics), client-side (RUM), and former experimentation (Evidently, now discontinued) monitoring tools
last_verified: 2026-08-18
categories: [Amazon Web Services, Cloud monitoring, Observability, Frontend monitoring]
artifact: https://claude.ai/code/artifact/5e8421c4-d645-4a97-847e-12e02ac0cffa
---

# CloudWatch Synthetics, RUM & Evidently

This article covers the three CloudWatch tools that watch an application from the *outside* or from the *end user's browser*, rather than from inside AWS infrastructure: **Synthetics** (scripted canaries), **RUM** (real user telemetry), and **Evidently** (feature flags/experiments — now discontinued, see below). See [Amazon CloudWatch](amazon-cloudwatch.md) for the full service family overview.

## CloudWatch Synthetics

Synthetics runs headless-browser or HTTP scripts ("canaries") on a schedule from AWS-managed Lambda infrastructure, so you get availability/latency signal even with zero real traffic — useful for pre-launch checks, low-traffic endpoints, or catching a break before a real user does.

### Canary blueprints

When creating a canary in the console, you start from a blueprint. Each blueprint generates a Node.js (or Python) script you can then edit directly:

| Blueprint | What it does |
|---|---|
| **Heartbeat monitoring** | Loads a URL, stores a screenshot + HAR file + accessed-URL log. `syn-nodejs-puppeteer-3.1+` supports monitoring *multiple* URLs in one canary, with per-URL status/duration/screenshot in the run report. |
| **API canary** | Multi-step REST API testing (GET/POST/PUT/PATCH/DELETE) — each step hits a different endpoint with its own headers/body. Integrates directly with API Gateway (pick an API+stage, or upload a Swagger template for cross-account/region testing). Not supported on Playwright runtimes. |
| **Broken link checker** | Crawls all `<a>` tags on a page (`document.getElementsByTagName('a')`) up to N links, reports 404s, invalid hostnames, malformed URLs, timeouts, dropped connections — with annotated source/destination screenshots. Not supported on Playwright runtimes. |
| **Visual monitoring** | Diffs screenshots against a baseline run (`syntheticsConfiguration.withVisualCompareWithBaseRun(true)`); fails the canary if the difference exceeds a set threshold. You can mask regions to ignore. Powered by ImageMagick. Requires `syn-puppeteer-node-3.2+`; not supported for Python/Selenium or Playwright. |
| **Canary recorder** | Record clicks/typing on a site with the CloudWatch Synthetics Recorder (a Chrome extension) and auto-generate the equivalent Node.js canary script. Not supported on Playwright. |
| **GUI workflow builder** | Scripts a UI flow (click, verify selector, verify text, input text, click-with-navigation) — e.g. fill a login form and confirm a "Welcome" message appears. Element selectors use CSS-style `[id=]`/`a[class=]` in Node.js or XPath in Python. |
| **Multi checks blueprint** | Up to 10 sequential HTTP/DNS/SSL/TCP checks defined via simple JSON, with assertions per check and Secrets-Manager-backed auth (Basic, API key, OAuth, SigV4) — the cheapest way to do basic uptime/cert-expiry checks without a full browser. |

### Runtimes

Canaries run on versioned runtimes named `syn-<language>-<framework>-<major>.<minor>`, e.g. `syn-nodejs-puppeteer-13.1` (Node.js + Puppeteer) or the `syn-python-selenium-*` family. AWS ships new runtime versions periodically (Puppeteer 6.x/Python-Selenium 2.x in Feb 2024, runtime-v7/Python-v3 in Mar 2024, etc.) — always prefer the newest runtime for a new canary; upgrading an existing canary off `syn-1.0` requires deleting and recreating it, since in-place updates don't retroactively add newer-runtime features.

### Minimal heartbeat canary (Node.js/Puppeteer)

```javascript
const synthetics = require('Synthetics');
const log = require('SyntheticsLogger');

const pageLoadBlueprint = async function () {
    const page = await synthetics.getPage();
    const response = await page.goto('https://example.com', { waitUntil: 'domcontentloaded', timeout: 30000 });
    await synthetics.takeScreenshot('loaded', 'success');
    if (!response || response.status() !== 200) {
        throw new Error(`Failed with status ${response ? response.status() : 'unknown'}`);
    }
};

exports.handler = async () => {
    return await pageLoadBlueprint();
};
```

### Scheduling

Canaries are scheduled with an `Expression`, e.g.:

```json
{"Expression": "rate(5 minutes)"}
```

Rate expressions (`rate(N minutes|hours|days)`) or full cron expressions are both supported; `rate(0 minutes)` / a one-time run is used for a single ad hoc execution.

### Artifacts and alarms

Every run's screenshots, HAR files, and logs land in an S3 bucket you designate at canary creation. Each canary automatically gets an associated CloudWatch metric (`SuccessPercent`, `Duration`, etc. in the `CloudWatchSynthetics` namespace) — attach a standard CloudWatch **Alarm** to `SuccessPercent` (e.g. alarm when it drops below 100% over N consecutive periods) to page on-call when a canary starts failing.

## CloudWatch RUM (Real User Monitoring)

RUM captures telemetry from *actual* visitor browsers rather than synthetic scripts — the only way to see real device mix, real network conditions, and real geographic latency.

### Setting up an app monitor

1. **CloudWatch console → Application Signals → RUM → Add app monitor.**
2. Set an **app monitor name** and platform (**Web**).
3. Set **Application domain list** — the domain(s) allowed to send data (wildcards like `*.example.com` supported).
4. Choose what to collect:
   - **Performance telemetry** — page load and resource load timing.
   - **JavaScript errors** — unhandled JS exceptions (optionally unminified via uploaded source maps in S3).
   - **HTTP errors** — failed HTTP requests made by the page.
   - If none of these are selected, RUM still records session-start events and page IDs (OS, browser, device type, location breakdowns) — useful traffic-shape data with the lowest possible event volume/cost.
5. Optionally allow the web client to **set cookies**, which lets you tie events to a persistent (RUM-generated) user ID and session ID.
6. Set **Session samples** — the percentage of sessions RUM actually instruments. Default is 100%; lower it to cut cost on high-traffic sites.
7. Choose **authorization**: either a public resource-based policy (no signing needed) or a **Cognito identity pool** (new or existing) that grants an unauthenticated IAM role permission to call `rum:PutRumEvents` — this is what lets an anonymous browser client push telemetry without embedding real AWS credentials in your page.
8. Optionally enable **Active tracing** to connect RUM sessions to AWS X-Ray (traces `fetch`/`XMLHttpRequest` calls made during sampled sessions, and surfaces those sessions inside Application Signals as "client pages").
9. Save, then copy the generated code snippet.

### The embed snippet

The console gives you an IIFE snippet to paste before `</head>`:

```html
<script>
(function(n,i,v,r,s,c,x,z){x=window.AwsRumClient={q:[],n:n,i:i,v:v,r:r,c:c};window[n]=function(c,p){x.q.push({c:c,p:p});}})(
  'cwr',
  '00000000-0000-0000-0000-000000000000', // app monitor id
  '1.0.0',
  'us-east-1',
  'https://client.rum.us-east-1.amazonaws.com/1.x/cwr.js',
  {
    sessionSampleRate: 1,
    guestRoleArn: "arn:aws:iam::123456789012:role/RUM-Monitor-unauth-role",
    identityPoolId: "us-east-1:00000000-0000-0000-0000-000000000000",
    endpoint: "https://dataplane.rum.us-east-1.amazonaws.com",
    telemetries: ["performance","errors","http"],
    allowCookies: true,
    enableXRay: false
  }
);
</script>
```

You can install `aws-rum-web` via NPM instead of the CDN snippet — the console offers a JS/TS module version, avoiding the risk of ad-blockers dropping the CDN script.

### What's auto-collected

- **Performance**: page load phases (via the Navigation Timing / Resource Timing APIs) plus **Core Web Vitals** — LCP, FID/INP, CLS.
- **Errors**: uncaught JS exceptions (`window.onerror`/`unhandledrejection`).
- **HTTP**: XHR/fetch calls, with status, latency, and (if `enableXRay`/`addXRayTraceIdHeader` set) an X-Ray trace header for end-to-end tracing into downstream services.
- **Session/page metadata**: browser, OS, device type, country — collected even with all three telemetries turned off.

Collected end-user data is retained 30 days by default; you can additionally mirror events into a CloudWatch Logs log group (with its own configurable retention) if you need longer-term storage.

## CloudWatch Evidently — discontinued

> **Service retired.** AWS announced that **support for CloudWatch Evidently ended on 16 October 2025**. As of that date the Evidently console and API are no longer accessible for new work; only critical security patches continued briefly afterward, and limit-increase requests are no longer honored. **This service should be treated as fully gone as of this writing (Aug 2026).**
>
> AWS's stated migration path: move feature-flag/launch use cases to **AWS AppConfig** (a Systems Manager feature), which already underpins Evidently's client-side evaluation internally. Evidently's A/B **experiment** capability (statistical A/B testing with p-values/confidence intervals) has no direct 1:1 AWS replacement — AWS does not name a managed successor for that half of Evidently; teams doing statistical experimentation need a third-party tool or to build on top of AppConfig flags plus their own metrics/stats pipeline.

For historical reference, Evidently's model was:

- **Project** — a container for a set of features/launches/experiments for one application.
- **Feature** — a piece of functionality with 1+ **variations** (e.g. `control` vs `treatment`), evaluated per user/session.
- **Launch** — a staged rollout: allocate a growing percentage of traffic to a new variation over time while watching a chosen metric, without a hard A/B statistical framework.
- **Experiment** — a true A/B test: split traffic across up to 5 variations, define metric goals, and let Evidently's statistical engine report anytime p-values and confidence intervals to tell you when a result is significant (conventionally p < 0.05).

## See also

- [Amazon CloudWatch](amazon-cloudwatch.md)

## References

1. "Using canary blueprints" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries_Blueprints.html>
2. "Runtime versions using Node.js and Puppeteer" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Library_nodejs_puppeteer.html>
3. "Set up a web application to use CloudWatch RUM" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM-get-started.html>
4. "Creating a CloudWatch RUM app monitor for a web application" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM-get-started-create-app-monitor.html>
5. aws-observability/aws-rum-web — official RUM web client source and configuration docs. <https://github.com/aws-observability/aws-rum-web>
6. "Perform launches and A/B experiments with CloudWatch Evidently" — Amazon CloudWatch User Guide. <https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Evidently.html>
7. "Support for Amazon CloudWatch Evidently ending soon" — AWS Cloud Operations Blog. <https://aws.amazon.com/blogs/mt/support-for-amazon-cloudwatch-evidently-ending-soon/>

---
*Last verified 2026-08-18 against the sources above.*
