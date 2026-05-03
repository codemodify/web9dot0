# brief — and non-complete — technical info on violations

A reference catalog of techniques used today to track and collect data about users on the web, on mobile, and in email. Organized by mechanism. None of this is exhaustive — vendors invent new combinations every quarter — but it covers the load-bearing categories as of mid-2026.

The intent is descriptive: to name the techniques so a reader can recognize and refuse them. Inclusion here does not imply endorsement.

---

## 1. Storage-based identifiers (the user's own browser holds the ID)

### 1.1 HTTP cookies
- **First-party cookies** — set by the site the user is visiting. Survived all recent browser policy changes; they are the backbone of session/login plus most modern analytics.
- **Third-party cookies** — set by a domain other than the one in the address bar. Blocked by default in Safari (ITP) and Firefox (ETP). In Chrome they were scheduled for deprecation but Google reversed course in July 2024 and again in October 2025, retiring most of the Privacy Sandbox replacement APIs and keeping third-party cookies enabled by default. They remain the default cross-site identifier on Chrome/Edge.
- **CHIPS (Cookies Having Independent Partitioned State)** — a partitioning model where a third-party cookie is keyed to the top-level site, so the same embedded service cannot stitch a user across different parent sites. Surviving Privacy Sandbox feature.
- **SameSite=None / Lax / Strict** — controls whether the cookie is sent on cross-site requests. `SameSite=None; Secure` is what cross-site trackers require.

### 1.2 Web storage / databases
- **localStorage**, **sessionStorage** — JS key-value stores. Persist IDs that survive cookie clears unless the user wipes site data.
- **IndexedDB** — larger structured database in the browser. Used for offline apps and as a regen surface for evercookie-style tracking.
- **Web SQL** (deprecated, still present in some embedded WebViews).
- **Cache Storage / Service Worker caches** — content cached by a service worker can encode an ID (e.g., the URL of a precached asset).
- **File System Access API** — origin-private filesystems (OPFS) provide another persistence vector.

### 1.3 Cache- and metadata-based identifiers (no storage permission needed)
- **ETag tracking** — server returns a unique `ETag` per visitor; the browser sends it back in `If-None-Match` on the next request.
- **Last-Modified** — same idea, different header.
- **HSTS supercookies** — encode bits of an ID across many subdomains by toggling HSTS pins.
- **HTTP/2 and HTTP/3 connection coalescing / 0-RTT tickets** — session resumption tickets can carry an identifier across connections.
- **TLS session tickets / session IDs** — same family.
- **Favicon cache** — variants of the favicon URL persist across incognito sessions in some browsers.
- **Browser back/forward cache, prefetch state** — secondary surfaces for ID re-injection.

### 1.4 Service workers and background scripts
A registered service worker runs across visits, intercepts network requests, and can re-establish identity even after cookies are cleared. Combined with Cache Storage and Background Sync, it turns the browser into a tracker host.

### 1.5 Supercookies and evercookies
- **Evercookie** (open-source PoC by samy.pl) writes the same identifier to ~15 different storage mechanisms (cookies, localStorage, IndexedDB, ETag, HSTS, Flash LSO historically, Silverlight, window.name, etc.) and respawns it from any surviving copy. Its descendants live on as commercial fraud-prevention SDKs.
- **ISP supercookies / UIDH (Unique Identifier Header)** — the network operator (Verizon was the famous case) injects a unique header into outbound HTTP requests at the carrier. The user cannot remove it from the browser; it lives on the ISP's edge.
- **Zombie cookies** — generic term for any cookie that respawns after deletion via one of the above mechanisms.

---

## 2. Browser fingerprinting (no storage; "the browser is the ID")

The browser's hardware + software configuration is read out and hashed. Fingerprints can re-identify a user with 80–95% accuracy across cookie clears and incognito sessions, and ML systems now flag anti-fingerprint countermeasures by their inconsistency patterns.

### 2.1 Graphics / rendering
- **Canvas fingerprinting** — render text and shapes to an off-screen 2D canvas, hash the pixel output. GPU + driver + font hinting + AA settings each shift pixels deterministically.
- **WebGL fingerprinting** — read GPU vendor/renderer strings, shader precision, supported extensions, and run shader programs whose output varies by GPU model.
- **WebGPU fingerprinting** (newer) — the WebGPU adapter exposes more granular capability flags than WebGL; emerging high-entropy signal.
- **Font enumeration** — measure rendered glyph metrics for each font in a list to detect which fonts are installed.
- **Emoji rendering** — variant per OS/font version.

### 2.2 Audio
- **AudioContext fingerprinting** — process a known waveform through `OfflineAudioContext`, hash the output. Differs by audio stack version. Safari 17+ injects randomization in Private mode.

### 2.3 System/device
- `navigator.userAgent`, **User-Agent Client Hints** (`Sec-CH-UA-*`) — browser, version, platform, mobile flag, architecture, model, full version list.
- `navigator.hardwareConcurrency` — logical CPU count.
- `navigator.deviceMemory` — RAM bucket.
- `navigator.platform`, `navigator.oscpu`.
- `screen.width/height/availWidth/colorDepth`, `window.devicePixelRatio`.
- `Intl.DateTimeFormat().resolvedOptions().timeZone` and `Date.getTimezoneOffset()`.
- `navigator.languages`.
- **Battery Status API** (deprecated/removed in most browsers) — formerly used to correlate visits within ~30 minutes.
- **Network Information API** — connection type, downlink.
- **Permissions API** — query state for camera/mic/geo/notifications adds entropy.
- **Media devices** — `enumerateDevices()` returns deviceIds (per-origin hashed) and device labels once permission is granted.
- **Speech synthesis voices** — `speechSynthesis.getVoices()` reveals OS voice list.
- **Plugins / MIME types** — mostly empty in modern Chromium but still reads in some browsers.
- **Math/floating-point quirks** — small differences in `Math.tan`, `Math.exp` across CPUs/runtimes.
- **CSS feature detection** — `@media`, `prefers-color-scheme`, `prefers-reduced-motion`.
- **Sensor APIs** — Generic Sensor API exposes accelerometer, gyroscope, magnetometer, ambient light, orientation. Mobile-specific. Activity inference (walking/running/typing) is feasible.

### 2.4 Behavioral fingerprint additions
- **Mouse-movement curves** and **touch-event timing** carry per-user entropy beyond the static config.

---

## 3. Network- and protocol-level fingerprinting

These work even if the browser is locked down, because they happen below the JS layer.

- **IP address** — coarse geolocation, ASN, VPN/datacenter detection. Combined with a small fingerprint, often unique.
- **TCP/IP stack fingerprinting** (p0f-style) — TTL, window size, options ordering.
- **TLS JA3 / JA3S** — hash of the ClientHello (cipher list, extension order, EC curves, EC point formats). Largely superseded.
- **TLS JA4 / JA4+** — Cloudflare/AWS/VirusTotal industry standard. Sorts ciphers and extensions deterministically before hashing, which defeats the per-connection randomization Chrome and Firefox introduced. Used at the edge to flag headless browsers, automation libraries, and malware-family TLS stacks.
- **HTTP/2 fingerprinting** (Akamai) — SETTINGS frame values, window-update sizes, header order, priority frames. Headless browsers and HTTP libraries each have a recognizable pattern.
- **HTTP/3 / QUIC fingerprinting** — initial packet structure, transport parameters.
- **DNS-side observables** — the resolver sees every domain the user visits if not using DoH/DoT to a third party.
- **CNAME cloaking** — site sets up `analytics.example.com` and CNAMEs it to a tracker (Adobe, Eulerian, Pardot, etc.). The browser sees a first-party request, so cookies set on that subdomain dodge third-party-cookie blocking. Studies put adoption around 10–15% of EU enterprise sites. Safari (WebKit) caps the lifetime of cookies set on CNAME-cloaked responses to 7 days; Brave and Firefox uncloak via tracker lists.
- **WebRTC ICE candidate leak** — `RTCPeerConnection` enumerates local network IPs (192.168.x.x, etc.) and the public IP from STUN, regardless of VPN state, unless WebRTC is disabled or the browser is hardened (Brave, Tor, recent Firefox with `media.peerconnection.ice.proxy_only`).
- **WebSocket and EventSource** timing — long-lived connections with per-user IDs.
- **DNS prefetch / preconnect** — leaks intent before the user even clicks.

---

## 4. Behavioral and interaction telemetry

Captured by JS embedded on the page and sent to a backend.

- **Pointer telemetry** — mousemove velocity, acceleration, jitter, click coordinates, hover dwell. Used for heatmaps and bot detection alike.
- **Scroll depth, scroll velocity, rage-scroll detection.**
- **Click maps and rage-click detection.**
- **Keystroke dynamics** — flight time (key-up to next key-down) and dwell time (key-down to key-up). Per-user identifying at high accuracy. Also used for emotion/fatigue inference. Recent research reports >99% accuracy distinguishing humans from bots from keystroke and mouse traces.
- **Form interaction telemetry** — captures characters as they are typed, field focus order, abandoned fields. The "submit" button is irrelevant to the recorder.
- **Session replay** — full DOM-mutation recording (FullStory, Microsoft Clarity, Hotjar, LogRocket, Mida, UXCam, Glassbox). Reconstructs the visit as a video. Documented leaks of credit cards, passwords, and PHI when masking is misconfigured. Drove a wave of ~1,500 CIPA wiretap lawsuits in the US in the 18 months before mid-2025.
- **Engagement / dwell** — visibility-API-based time-in-view per element.
- **Copy/paste, right-click, print, screenshot detection** (the latter via UI-state heuristics).
- **Battery-drain and device-thermal inferences** (where APIs allow).

---

## 5. Cross-site stitching

Mechanisms that link the same user across multiple sites.

- **Tracking pixels** — 1×1 image (or invisible iframe, or `<script>`) loaded from a third-party host that drops or reads its own cookie. Examples: Meta Pixel, Google Tag, TikTok Pixel, LinkedIn Insight Tag, Pinterest Tag, Snap Pixel, Reddit Pixel, X/Twitter Pixel.
- **Tag managers** — Google Tag Manager, Tealium, Adobe Launch — hot-load arbitrary trackers without redeploying the site.
- **iframe + postMessage choreography** — embedded login/share widgets read their own first-party cookie inside the iframe, then talk to the parent via `postMessage` to confirm identity.
- **Bounce tracking / redirect tracking** — link rewriter sends the click through `t.example-tracker.com/?to=...`, which sets a first-party cookie there before redirecting. WebKit's "Bounce Tracking Protection" purges state for domains visited only as redirects.
- **Link decoration** — query/fragment parameters that carry an ID across the click: `gclid` (Google), `fbclid` (Meta), `ttclid` (TikTok), `msclkid` (Microsoft), `igshid`, `mc_eid`, `epik` (Pinterest), and the full `utm_*` family. Brave and Firefox strip many of these on outbound clicks; Safari strips known click IDs in Private mode and via Link Tracking Protection in Mail/Messages on iOS 17+.
- **window.name** — read/written across navigations on the same tab.
- **postMessage broadcasting from a shared third-party origin** — when the same iframe is embedded on many sites, it can use BroadcastChannel within its own origin to coordinate.
- **Cross-origin opener / referrer leakage** — `Referer` header and `Sec-Fetch-Site` reveal the previous page.
- **OAuth / "Sign in with X"** — turns every consenting site into a first-party graph node for the IdP.

---

## 6. Identity resolution and identity graphs

The cookie-replacement industry. The goal is a stable cross-device, cross-site, cross-channel identifier rooted in something the user provided once.

- **Hashed email (HEM)** — SHA-256 (and MD5 / SHA-1 historically) of a normalized email address. Treated as pseudonymous in industry, but it is deterministic — anyone who has the email and the same normalization can compute the hash.
- **Unified ID 2.0 (UID2)** — open-source framework. The user's email is salted, hashed, and encrypted into a token; tokens rotate. Promoted by The Trade Desk; widely integrated by SSPs, DSPs, and CDPs.
- **ID5** — probabilistic + deterministic universal ID for ad bidstream.
- **LiveRamp RampID / ATS (Authenticated Traffic Solution)** — RampID is a pseudonymous person-level identifier resolved from offline PII (name, postal address, email, phone) via LiveRamp's AbiliTec graph. ATS exchanges a publisher-side hashed email for a RampID at page load. Cross-walks to UID2 are standard.
- **Other universal IDs** — The Trade Desk's EUID (EU-only), Criteo ID for Exchanges, Lotame Panorama ID, Merkle Merkury, Epsilon CORE ID, Neustar Fabrick.
- **Probabilistic device graphs** — match devices by IP + UA + visit timing without a deterministic key. Mostly a fallback now.
- **Customer Match / Audience Match** — advertiser uploads hashed emails/phones; the platform matches against its logged-in user base. Available on Google, Meta, LinkedIn, X, TikTok, Amazon, Snap.
- **Data clean rooms** — Google Ads Data Hub, Amazon Marketing Cloud, Snowflake/LiveRamp/Habu rooms — two parties join their identity graphs without exporting raw PII to each other.

---

## 7. First-party / account-based collection

The strongest signal is the one the user volunteers.

- **Login identity** (email, phone, social ID).
- **Newsletter and account signup forms.**
- **Loyalty programs** — barcode/email at point of sale ties offline purchases to the online graph.
- **CRM / CDP ingestion** — Salesforce, Segment, Rudderstack, mParticle, Hightouch.
- **Customer support transcripts, chat logs, voice calls** (now often transcribed and embedded).
- **Survey/quiz funnels** — gather declared preferences + email.
- **In-product telemetry** — feature flags, A/B-test cohort, error reports.

---

## 8. Server-side and "post-cookie" tracking

The shift since ~2022. Instead of the browser pinging the ad network, the site's own server pings the ad network's API.

- **Meta Conversions API (CAPI)** — server-to-server event ingestion. Standard practice in 2026 alongside the Pixel; the two are deduplicated by `event_id`. Recovers events that ad blockers and ITP suppress.
- **Google Enhanced Conversions / Google Ads API / GA4 Measurement Protocol** — same pattern.
- **TikTok Events API**, **LinkedIn Conversions API**, **Pinterest Conversions API**, **Snap CAPI**, **X/Twitter CAPI**, **Reddit CAPI** — every major ad network now has one.
- **Server-side Google Tag Manager (sGTM)** — a customer-controlled tagging server (often on a first-party subdomain) that receives browser events and fans them out to ad networks server-side. Also used to set first-party cookies with longer lifetimes than browser-set ones.
- **Reverse proxy + first-party tag hosting** — `cdn.site.com/track.js` proxied to a vendor's CDN to dodge blocklists.
- **Data warehouse activation** — Reverse-ETL (Hightouch, Census) syncs Snowflake/BigQuery audiences directly to ad platforms via Customer Match.
- **Event collectors** — Snowplow, Jitsu, RudderStack — self-hosted analytics stacks that the site fully controls.

Reported industry numbers: ecommerce sites adding server-side tracking recover ~30–40% additional "tracked" events that the browser path loses to ad blockers, ITP, MPP, and consent gates.

---

## 9. Mobile / app tracking

The mobile stack is its own world; the web techniques largely apply, plus:

- **IDFA (iOS Identifier for Advertisers)** — opaque user ID. Since iOS 14.5 (App Tracking Transparency), apps must prompt; opt-in rates collapsed to roughly 25% globally and IDFA is mostly unavailable.
- **GAID / AAID (Google Advertising ID)** — Android equivalent. Still broadly available in 2026; no firm deprecation date. Users can reset or opt out, but defaults are permissive.
- **SKAdNetwork (SKAN) / AdAttributionKit (AAK)** — Apple's privacy-preserving on-device attribution. AAK replaced SKAN's planned 5.0 in iOS 18.4 and has expanded with country-level postbacks under crowd-anonymity thresholds. Reporting is delayed and aggregated.
- **Postbacks** — server-to-server callbacks from the ad network or MMP confirming an install/event.
- **MMPs (Mobile Measurement Partners)** — AppsFlyer, Adjust, Branch, Singular, Kochava — sit between apps and ad networks, deduplicate, and orchestrate SKAN/AAK.
- **Probabilistic / fingerprint attribution on iOS** — explicitly forbidden by Apple's policy but widely practiced; uses IP + device-class + timing.
- **SDKs in apps** — analytics SDKs (Firebase, Amplitude, Mixpanel), ad SDKs (AdMob, Meta Audience Network, Unity Ads), location SDKs (X-Mode/Outlogic, Cuebiq, Veraset, SafeGraph, Foursquare, Gravy Analytics), and bug-reporting SDKs (Crashlytics, Bugsnag, Sentry, Instabug — many include session replay).
- **Location resale via SDK or server-to-server feed** — the most regulated subspace; FTC actions in 2024 against InMarket, Mobilewalla, X-Mode/Outlogic. Government/LE buyers (ICE, CBP, military) procure the same data.
- **Real-time bidding (RTB) bidstream** — every ad request leaks IP, geo, device IDs, and inferred attributes to dozens-to-hundreds of bidders, who archive it. The bidstream is itself a tracking dataset.
- **Cross-app linking** — Universal Links, App Links, deferred deep links.
- **Push tokens (APNs/FCM)** — long-lived per-app per-device identifiers.
- **WebView abuse** — apps embedding a WebView for "login with X" can read cookies / inject JS.

---

## 10. Email tracking

- **Open pixels** — 1×1 image fetched from the sender's tracker on render. Records IP, User-Agent, time, geo.
- **Click rewriting** — every link in the email is replaced with `track.example.com/?u=...&id=...`, redirecting to the destination after logging.
- **CSS-based opens** — background-image in a `<style>` block, used to dodge naive image blockers.
- **Read-receipts** standardized in some clients (Outlook, ProtonMail) and disabled-by-default elsewhere.
- **Apple Mail Privacy Protection (MPP)** — preloads images via Apple's proxy when the message arrives, so all opens look identical from a fixed Apple IP. As of 2025, ~49% of all reported email "opens" come from MPP, inflating apparent open rates by 15–20 points and rendering open-rate-as-engagement-signal mostly useless.
- **Gmail image proxy** — caches images server-side; defeats geo and User-Agent leakage but does not affect click tracking. Click tracking remains fully effective in Gmail.
- **Form-in-email AMP** — Gmail's AMP-for-Email allows live forms inside the inbox, which transmit interactions in real time.
- **Tracker-aware mail clients** — HEY, Proton, DuckDuckGo Email Protection rewrite, neutralize, or proxy known trackers.

---

## 11. Privacy-Sandbox and "privacy-preserving" replacement APIs

Most have been retired or descoped, but a few remain in Chrome/Edge as of late 2025:

- **CHIPS** — partitioned third-party cookies (above).
- **FedCM (Federated Credential Management)** — browser-mediated federated login, replaces third-party-cookie-based SSO flows.
- **Private State Tokens** (formerly Trust Tokens) — unforgeable, unlinkable anti-fraud tokens issued by an issuer and redeemed elsewhere.
- **Topics API** — the browser locally derives a small set of interest topics from browsing history and shares them with sites. (Retired in October 2025 in Chrome's most recent Privacy Sandbox descope.)
- **Protected Audience API / FLEDGE** — on-device interest-group ad auctions. (Retired.)
- **Attribution Reporting API** — aggregated, noised conversion measurement. (Retired/descoped.)
- **Shared Storage**, **Storage Access API** — user-mediated cross-site state access.

The retreat of the Sandbox means cross-site tracking on Chrome continues mostly through third-party cookies, fingerprinting, server-side ingestion, and identity graphs.

---

## 12. Server- and infrastructure-side observation

Even with no third-party tags, the site's own infrastructure logs everything.

- **Web access logs** (Nginx/Apache/Caddy) — IP, UA, referer, cookie, path, timing.
- **CDN edge logs** (Cloudflare, Fastly, CloudFront, Akamai) — same plus TLS fingerprint, HTTP/2 fingerprint, country, ASN.
- **Application telemetry / Real User Monitoring (RUM)** — Datadog, New Relic, Sentry, Honeycomb, Grafana Faro. Often capture URL paths (which can contain PII), UA, IP, and stack traces with local-variable values.
- **APM and tracing headers** — distributed-trace IDs propagate user identity through the backend.
- **Load balancer / WAF logs** — same IP/UA/JA4 axis at the edge.
- **DNS server logs** — at the resolver (ISP, Cloudflare 1.1.1.1, Google 8.8.8.8, Quad9).

---

## 13. Machine-learning de-anonymization across signals

Modern trackers don't choose one signal — they fuse them.

- **Stable-fingerprint stitching** — combine a soft fingerprint (Canvas + WebGL + UA-CH + timezone) with a re-identifying signal (IP + behavior) and a deterministic signal (login or HEM). The fingerprint anchors the user across short sessions; the deterministic key anchors them across devices.
- **Cohort/lookalike modeling** — even without an ID, an ML model can predict "user-class" attributes (income, age band, intent) from browsing patterns and place users in addressable cohorts.
- **Anti-anti-fingerprinting detection** — randomization libraries leak entropy through their inconsistencies; ML classifiers spot the noise pattern.
- **Re-identification across pseudonymous datasets** — joining a "de-identified" dataset against another with overlapping attributes routinely re-identifies individuals (Netflix Prize, Strava heatmap, NYC taxi data are canonical examples).

---

## 14. The data flow downstream

Where collected data ends up:

- **Ad networks and DSPs** — for retargeting and bidding.
- **Data brokers** — Acxiom, Epsilon, Experian, Equifax, Oracle Data Cloud (winding down), TransUnion TruAudience, LiveRamp Data Marketplace, Lotame, Bombora.
- **Location-data brokers** — sell precise geolocation traces; FTC has reached settlements with multiple operators since 2024.
- **Government procurement** — federal agencies (US: ICE, CBP, IRS-CI, DoD components) buy commercial location and identity data without warrants. Reporting through 2026 documents ongoing contracts.
- **Real-time-bidding archives** — third parties scrape the bidstream itself as a data product.
- **Credit / insurance / employment scoring** — alternative-data feeds derived from web behavior.

---

## 15. What does *not* count as "no tracking"

A site claiming "no tracking" but doing any of the following is still tracking:

- "Privacy-friendly" analytics that hash IP + UA into a daily "visitor ID" (this is fingerprinting with a rotating salt — it still re-identifies within a day).
- First-party-only cookies used for cross-page behavioral profiling.
- Server-side conversion APIs with full event payloads (`fbp`, `external_id`, hashed email).
- "Anonymous" RUM that captures full URLs, which often contain account IDs, search queries, or order numbers.
- CDN logs sent to a third party for "DDoS protection" that include cookies and full URLs.
- Embedded YouTube / Vimeo / Twitter / reCAPTCHA / Google Fonts / Gravatar — each is a third-party request that hands the user's IP and Referer to that vendor.
- Outbound links to social networks without `rel="noreferrer"`.
- "Login with Google/Apple/GitHub" buttons — even before the click, the embedded SDK may fingerprint.

A genuine "no tracking" posture means: no third-party requests by default, no analytics (or strictly aggregate, IP-truncated, no-cookie), no fingerprinting, no behavioral telemetry, no server-side event forwarding, no CDN that logs cookies, no embedded social/media widgets that phone home, and no email pixels or click rewriting.

---

## Sources (selected)

- Browser fingerprinting overview — https://arxiv.org/html/2411.12045v1
- Canvas / WebGL / Audio fingerprinting deep-dive — https://blog.octobrowser.net/canvas-audio-and-webgl-an-in-depth-analysis-of-fingerprinting-technologies
- Fingerprinting techniques with code — https://www.thumbmarkjs.com/content/browser-fingerprinting-techniques/
- Privacy Sandbox status update — https://privacysandbox.google.com/blog/update-on-the-plan-for-phase-out-of-third-party-cookies-on-chrome
- Privacy Sandbox October 2025 retirement — https://segwise.ai/blog/google-privacy-sandbox-shutdown-reason
- CNAME cloaking primer — https://medium.com/nextdns/cname-cloaking-the-dangerous-disguise-of-third-party-trackers-195205dc522a
- WebKit on CNAME cloaking and bounce tracking — https://webkit.org/blog/11338/cname-cloaking-and-bounce-tracking-defense/
- Brave on CNAME trickery — https://brave.com/privacy-updates/6-cname-trickery/
- CNAME cloaking research (IEEE) — https://ieeexplore.ieee.org/document/9403411/
- TLS JA4 / bot detection — https://auth0.com/blog/strengthening-bot-detection-ja4-signals/
- Cloudflare JA3/JA4 docs — https://developers.cloudflare.com/bots/additional-configurations/ja3-ja4-fingerprint/
- Evercookie (samy.pl) — https://github.com/samyk/evercookie
- Evercookie / supercookie reference — https://en.wikipedia.org/wiki/Evercookie
- Supercookie taxonomy — https://panopticlick.org/anatomy/supercookies/
- Session-replay data exfiltration (Princeton/EFF) — https://privacyinternational.org/examples/1918/no-boundaries-exfiltration-personal-data-session-replay-scripts
- Session replay & CIPA wave — https://seresa.io/blog/privacy-compliance/session-replay-tools-are-the-next-cipa-wave-on-woocommerce
- Meta CAPI guide — https://www.dinmo.com/third-party-cookies/solutions/conversions-api/meta-ads/
- Server-side tracking guide — https://ingestlabs.com/blogs/the-complete-guide-to-server-side-tracking-2026/
- Identity graph (The Trade Desk) — https://www.thetradedesk.com/resources/how-identity-graphs-are-built-the-present-and-the-future
- Cracked Labs report on LiveRamp — https://crackedlabs.org/dl/CrackedLabs_IdentitySurveillance_LiveRamp.pdf
- UID2 / LiveRamp ATS integration — https://unifiedid.com/docs/guides/integration-liveramp-tips
- WebRTC leak test — https://browserleaks.com/webrtc
- Sensor / WebRTC privacy — https://www.praetorian.com/blog/javascript-sensor-api-new-browser-features-webrtc-raise-privacy-concerns/
- Email tracking pixels in 2026 — https://www.gblock.app/articles/email-tracking-pixels-2026-gmail-proxy
- Apple Mail Privacy Protection impact — https://sparkle.io/blog/email-tracking-pixels/
- SKAdNetwork / AdAttributionKit — https://www.rocketshiphq.com/what-is-skadnetwork-skan-how-it-works/
- Mobile attribution 2026 — https://www.moburst.com/blog/mobile-attribution-in-2026-what-marketers-actually-need-to-know/
- EFF on location data brokers — https://www.eff.org/issues/location-data-brokers
- Government purchase of broker data — https://www.npr.org/2026/03/25/nx-s1-5752369/ice-surveillance-data-brokers-congress-anthropic
- Behavioral biometrics survey — https://www.sciencedirect.com/science/article/pii/S0167404825004201
- Bot detection via TLS handshake — https://arxiv.org/html/2602.09606v1
- Safari 26 link-tracking changes — https://taggrs.io/safari-26-tracking-changes/
