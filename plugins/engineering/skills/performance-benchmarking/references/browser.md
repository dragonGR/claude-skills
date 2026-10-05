# Browser performance measurement

Read this when a UI got slower (INP, LCP, CLS, dropped animation frames), before claiming a frontend change made it faster, or when adding a browser performance gate to CI. It covers how to measure. How to fix React code is in frontend-engineering and react-motion.

## Field decides, lab explains

Field data is what real users got on their devices and networks. Lab data is one scripted run on one machine. They answer different questions, and a lab number cannot pass or fail a Core Web Vital on its own.

- INP cannot be measured in a lab run with no user input. web.dev lists INP as field-only and Total Blocking Time (TBT) as its lab proxy. A Lighthouse navigation run reports TBT, not INP. A lab INP number exists only when a script performs the interaction (see Playwright below).
- CrUX is Chrome only: desktop Chrome and Android Chrome, not Chrome on iOS, Android WebView or other Chromium browsers. It covers pages that are publicly discoverable and popular enough, and the API serves a 28-day rolling average. A regression shipped today shows up diluted over four weeks, and an internal app behind login has no CrUX data at all. Use it to see the trend, and your own RUM to tie a change to a release.
- Use the lab to reproduce a field problem and find the cause, and to stop known regressions in CI. Confirm the fix in field data after release.

Google's thresholds, applied at the 75th percentile of page loads and segmented by mobile and desktop:

| Metric | Good | Poor |
| --- | --- | --- |
| LCP | ≤ 2.5 s | > 4 s |
| INP | ≤ 200 ms | > 500 ms |
| CLS | ≤ 0.1 | > 0.25 |

Between the two is "needs improvement". web-vitals exports the same values as `LCPThresholds`, `INPThresholds` and `CLSThresholds`, and each reported metric carries a `rating`. Report p75 per device class and per page template. A mean hides the slow tail, and one p75 over desktop and mobile together hides mobile.

## Field measurement with web-vitals

```js
import { onCLS, onINP, onLCP } from 'web-vitals/attribution';
import { RUM_ENDPOINT } from './config.js';

const queue = new Set();
const report = (metric) => queue.add(metric);

onCLS(report);
onINP(report);
onLCP(report);

addEventListener('visibilitychange', () => {
  if (document.visibilityState !== 'hidden' || queue.size === 0) return;
  const body = JSON.stringify([...queue].map(({ name, value, rating, id, navigationType, attribution }) => ({
    name, value, rating, id, navigationType,
    target: attribution.interactionTarget ?? attribution.target ?? attribution.largestShiftTarget,
    inputDelay: attribution.inputDelay,
    processingDuration: attribution.processingDuration,
    presentationDelay: attribution.presentationDelay,
    longestScript: attribution.longestScript?.entry.sourceURL,
  })));
  navigator.sendBeacon(RUM_ENDPOINT, body);
  queue.clear();
});
```

- The attribution build is what makes field data actionable. For INP it splits the slowest interaction into input delay (main thread busy before the handler ran), processing duration (the handlers) and presentation delay (rendering after the handlers), and names the target element and the longest script. Without it you know INP got worse and nothing else.
- Flush on `visibilitychange` to hidden with `sendBeacon`. INP and CLS keep changing until the page is hidden, and the Page Lifecycle docs call the hidden state the last reliable moment to send data: `unload` does not fire when a mobile user closes the tab or the app from the switcher.
- web-vitals 6 changed two things people trip over. `includeProcessedEventEntries` on `onINP` now defaults to `false`. And soft navigations (SPA route changes) can be reported as separate page views with `{ reportSoftNavs: true }`, but only in Chromium 151+, so the same SPA reports differently per browser. Without that option every route change in an SPA counts toward the first page's INP and CLS.
- Browser support differs per metric. `onCLS` is Chromium only. INP needs Event Timing's `interactionId`, which MDN's compatibility data lists for Chrome 96, Firefox 144 and Safari 26.2. Tag every report with the browser, or a change in your traffic mix looks like a regression.
- The underlying APIs do not see inside iframes, even same-origin ones, while CrUX includes iframe content. Pages with iframes report different values from CrUX for that reason alone.

## Long Animation Frames

The Long Animation Frames API (LoAF) reports every frame whose rendering was delayed by more than 50 ms, with the scripts that ran in it. Long Tasks named only the document or iframe, left out the rendering work that follows a task, and missed frames built from several tasks that were each under 50 ms. LoAF covers all three.

```js
new PerformanceObserver((list) => {
  for (const frame of list.getEntries()) {
    for (const s of frame.scripts) {
      record({ frameMs: frame.duration, blockingMs: frame.blockingDuration, invoker: s.invoker,
               source: s.sourceURL, fn: s.sourceFunctionName, scriptMs: s.duration,
               forcedLayoutMs: s.forcedStyleAndLayoutDuration });
    }
  }
}).observe({ type: 'long-animation-frame', buffered: true });
```

- Chrome and Edge 123+. Not in Firefox or Safari, so LoAF data describes Chromium users only.
- Script entries appear only for scripts that ran 5 ms or more. Cross-origin scripts report only `sourceURL`, with an empty function name, and cross-origin iframes and workers are not attributed at all.
- A page opened from `file://` gets frames with an empty `scripts` array. Serve the build over HTTP when testing locally.
- `forcedStyleAndLayoutDuration` is layout thrashing: a script read layout after writing styles in the same frame.
- web-vitals' INP attribution already attaches the intersecting LoAF entries (`longAnimationFrameEntries`, `longestScript`), so for INP you usually need no observer of your own. Observe LoAF directly for janky scrolling and animations that are not tied to one interaction.

## Chrome DevTools Performance panel

- Record in a clean profile: Incognito or a separate Chrome profile with no extensions. Extensions inject scripts and requests into the page you are measuring.
- Measure a production build served over HTTP. A dev build runs extra checks and instrumentation (React's dev build marks every component in the Performance panel). React 19.2+ adds Scheduler and Components tracks to the panel, and a production build leaves them out. To get them in an otherwise production build, alias `react-dom/client` to `react-dom/profiling` at build time; there only the Scheduler track is on by default, and the Components track lists only subtrees wrapped in `<Profiler>`.
- Throttle the CPU. A fixed "4x slowdown" means one thing on a workstation and another on an old laptop. Chrome 134+ can calibrate "mid-tier mobile" and "low-tier mobile" presets for the machine you are on. Mid-tier approximates a typical mobile device; low-tier covers the slow tail. Calibrate once per machine and use the same preset for before and after.
- The live metrics view shows LCP, CLS and INP for your own page load and interactions, and next to them the CrUX p75 for the URL or origin when field data is enabled. If your local INP is far better than the field p75, your setup or the interaction you tried does not match what users do, and the trace will not show their problem.
- The Interactions track shows each interaction's input delay, processing time and presentation delay, and flags those over 200 ms. The Frames track marks frames as dropped (red) or partially presented (yellow), which is the view for animation jank.
- One recording is one sample. Repeat the interaction several times and look at the spread before you compare a before and after trace.

## Regression gates in CI

### Lighthouse CI

```bash
npx -p @lhci/cli@0.15.1 lhci autorun    # reads lighthouserc.json from the working directory
```

```json
{
  "ci": {
    "collect": { "staticDistDir": "./dist", "numberOfRuns": 5 },
    "assert": {
      "assertions": {
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500, "aggregationMethod": "median-run" }],
        "total-blocking-time": ["error", { "maxNumericValue": 200, "aggregationMethod": "median-run" }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1, "aggregationMethod": "median-run" }]
      }
    }
  }
}
```

- The default `aggregationMethod` is `optimistic`: the value most likely to pass across all runs. A gate left on the default passes if one run out of five was fast. Use `median-run` (the run Lighthouse picks as most representative) or `pessimistic`.
- `numberOfRuns` defaults to 3. Lighthouse's variability guide says the median of 5 runs is twice as stable as a single run. Use at least 5.
- The LCP and CLS limits above are the "good" thresholds from the table; the TBT limit is only an example. A limit at the threshold catches pages that cross it, not a large regression on a page far below it. For that, set limits from your own pages' baseline and measured run-to-run spread, kept in this config file like any other gate.
- Assert on metrics, not on the performance score. The score is a weighted blend whose weights change between Lighthouse versions.
- `@lhci/cli` 0.15.1 bundles Lighthouse 12.6.1 while standalone `lighthouse` is at 13.5.0. Lighthouse majors change audits and scoring, so pin the version and never compare results across majors.
- The Lighthouse docs ask for at least 2 dedicated cores (4 recommended), no burstable or shared-core instances, and never more than one Lighthouse run at a time on the same machine.

### Playwright for interactions

Lighthouse's navigation run never clicks anything. To gate on interaction latency, script the interaction, throttle the CPU over the Chrome DevTools Protocol, and read Event Timing entries, which is what INP is computed from:

```js
import { chromium } from 'playwright';

const { TARGET_URL, CPU_SLOWDOWN, RUNS, SELECTOR } = process.env;
const browser = await chromium.launch();
const samples = [];
for (let run = 0; run < Number(RUNS); run++) {
  const context = await browser.newContext();
  const page = await context.newPage();
  const cdp = await context.newCDPSession(page);
  await cdp.send('Emulation.setCPUThrottlingRate', { rate: Number(CPU_SLOWDOWN) });
  await page.addInitScript(() => {
    window.__interactions = new Map();
    new PerformanceObserver((list) => {
      for (const e of list.getEntries()) {
        if (!e.interactionId) continue;
        const prev = window.__interactions.get(e.interactionId) ?? 0;
        window.__interactions.set(e.interactionId, Math.max(prev, e.duration));
      }
    }).observe({ type: 'event', durationThreshold: 16, buffered: true });
  });
  await page.goto(TARGET_URL);
  await page.click(SELECTOR);
  await page.waitForTimeout(500); // let the next paint land so the entry is dispatched
  samples.push(await page.evaluate(() => Math.max(0, ...window.__interactions.values())));
  await context.close();
}
await browser.close();
console.log(JSON.stringify(samples));
```

- CDP sessions and `Emulation.setCPUThrottlingRate` are Chromium only. The rate is a plain multiplier on the CI machine's CPU, so a gate only means something against a baseline from the same machine in the same job.
- Event Timing durations are rounded to 8 ms. A difference of one or two steps between base and head is noise.
- A fresh context per run gives a cold cache each time. Warm the page first if the claim is about repeat visits.
- `browser.startTracing(page, { path, screenshots: true })` and `browser.stopTracing()` write a Chromium trace that opens in the DevTools Performance panel, which is what to attach to a failed gate. `context.tracing` is Playwright's own trace viewer (actions, DOM snapshots, network) and is not a performance profile.

## Noise control for browser runs

- Production build, served over HTTP from the same host as before, with the same data. Mock or block third-party scripts (ads, A/B tests, chat widgets) or pin their versions. Lighthouse lists page nondeterminism as a high-impact source of variance that no throttling removes.
- Clean browser profile, no extensions, nothing else on the machine competing for CPU. One browser run at a time per machine.
- Same CPU throttling preset and the same device emulation for base and head, chosen to match the field p75 device.
- Run base and head alternately in the same job (see environment-and-ci.md), at least 5 runs each, and compare distributions, not single runs. A difference inside the A/A spread is no difference.
- Keep the traces and Lighthouse JSON reports as build artifacts, so a failed gate can be read later instead of rerun until it passes.
