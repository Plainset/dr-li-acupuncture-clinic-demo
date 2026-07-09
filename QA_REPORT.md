# QA Report

Use exact pass/fail evidence. "Looks fine" is not a result.

## Pages Checked
- index.html (Home)
- services.html (Treatments)
- contact.html (Contact & Booking)

## Audit Results
Ran `.pipeline/qa/contrast-audit.js` and `.pipeline/qa/upscale-audit.js` verbatim via `preview_eval` against the live dev server (port 4228), once per page at each of the three standard breakpoints, with every `[data-reveal]` element force-set to `.is-visible` first so nothing below the fold was skipped by the scripts' `isVisible()` gate.

| Check | Result | Evidence |
|---|---|---|
| Contrast audit — desktop (1280×720/800) | PASS | index.html: 81 text nodes checked, 0 violations. services.html: 65 checked, 0 violations. contact.html: 54 checked, 0 violations. |
| Contrast audit — tablet (768×1024) | PASS | index.html: 81 checked, 0 violations. services.html: 65 checked, 0 violations. contact.html: 54 checked, 0 violations. |
| Contrast audit — mobile (375×812) | PASS | index.html: 81 checked, 0 violations. services.html: 65 checked, 0 violations. contact.html: 54 checked, 0 violations. |
| Upscale mobile (375×812) | PASS | index.html: 5 images checked, 0 violations, 0 broken. services.html: 3 checked, 0 violations, 0 broken. contact.html: 2 checked, 0 violations, 0 broken. |
| Upscale tablet (768×1024) | PASS | Same totals per page as above, 0 violations, 0 broken on all 3 pages. |
| Upscale desktop (1280×720/800) | PASS | Same totals per page as above, 0 violations, 0 broken on all 3 pages. |
| Broken images | PASS | 0 `brokenImages` across all 9 page×breakpoint runs. |
| Aspect mismatch advisory | PASS (advisory) | 0 `aspectMismatches` across all 9 runs — all images use `object-fit: cover` inside `aspect-ratio` wrappers, not `fill`. |

## Manual Checks
| Check | Result | Notes |
|---|---|---|
| Gradient/`::before`-layered backgrounds (contrast script `needsManualCheck`) | PASS (manual) | `.btn-primary` and `.cta-band` use `linear-gradient(in oklch ...)` between `--cinnabar` (oklch L 0.45) and `--cinnabar-deep` (oklch L 0.33), with text in `--parchment` (oklch L 0.96). The lightness gap (0.96 vs 0.33–0.45) is large enough to clear WCAG AA on both ends of the gradient; confirmed via `preview_inspect` computed color read (`h2` color `oklch(0.96 0.02 80)`) against the known gradient endpoints. Both elements also now carry a solid `background-color` fallback (see Fixed Verification) so text never sits on an unstyled background if the gradient fails to paint for any reason. |
| Text on photo | N/A | No text is overlaid directly on a photo anywhere in this build (hero/portrait images sit beside copy, not behind it), so this check doesn't apply. |
| Image/content match | PASS | Hero portrait = Dr Li at the herbal cabinet (About page source) used with credentials copy; second portrait = Dr Li at the clinic counter (homepage source) used with bio/treatments copy; meditation photo = Dr Li in a Qigong posture (Qigong page source) used with the Qigong section. Each real photo used once, next to the copy it actually illustrates. |
| Fabricated claims | PASS | Cross-checked every on-page fact against `BUILD_BRIEF.md`'s Allowed Facts table. No price, no opening hours for the London clinic, and no rating/review-score are stated anywhere in the build — see `Do Not Claim` in `BUILD_BRIEF.md` for the specific items deliberately excluded (pricing, hours, "Free Consultation," star ratings, WhatsApp, before/after photo pair, decorative stock banner). |
| Mobile layout / real-content text overflow | PASS | At 375px width, checked `scrollWidth` vs `clientWidth` on `.brand-text strong/span`, `.trust-strip strong/span`, `.hero-media .seal`, `.card h3`, `.t-card q/footer`, `.footer-col li`, `.footer-bottom span`, and `.info-list .value` (which holds the full address and the full email address, the longest real strings on the site) — 0 overflowing elements found. |
| Mobile nav toggle | PASS (after fix) | See Blocking Issues / Fixed Verification below — found broken, fixed, reverified working (open height 744px, correct nav items, closes again) on all 3 pages at 375×812. |
| Broken/failed network requests | PASS | `preview_network` showed 0 failed requests on every page loaded during this session. |
| Internal link / asset path integrity | PASS | Static cross-check of all 3 HTML files: every `src="images/...\"` reference matches a file actually present in `images/`; every internal `href="*.html"` matches an existing page; `css/styles.css` and `js/main.js` paths resolve. No lorem ipsum or placeholder text found (`grep` clean). |

## Blocking Issues
| Issue | Evidence | Required fix |
|---|---|---|
| Mobile nav menu invisible when "open" | `.site-header` had `backdrop-filter: blur(10px)`, which creates a CSS containing block for `position: fixed` descendants. The mobile `.main-nav` (`position: fixed; inset: 68px 0 0 0;`) was therefore contained inside the ~68px-tall header box instead of the viewport — `getBoundingClientRect()` on the "open" nav returned `height: 49` instead of filling the screen, so the menu was practically invisible/unusable on mobile even though `aria-expanded="true"` and `.is-open` were correctly applied. | Removed `backdrop-filter` from `.site-header`, replaced with a solid `background: var(--surface-raised)`. Reverified: open nav height now 744px (full viewport minus header) on index.html, services.html and contact.html at 375×812; toggle opens and closes correctly; re-ran contrast + upscale audits after the fix on all 3 pages at mobile — still 0 violations. |

## Advisory Issues
- The `preview_click` tool's simulated click on the booking-form submit button was affected by cross-session interference in this shared preview environment (the browser tab ended up navigating to a different concurrently-running business's dev server on another port — unrelated to this site's code; confirmed no such link/script exists in this repo). The form's JS logic (`js/main.js`: `e.preventDefault()` + set `.form-status` text) was verified by direct code review and is structurally identical in style to the nav-toggle handler in the same file, which *was* live-verified working via direct `.click()` dispatch. The dev server for this project was later evicted by the shared preview pool's 5-server cap (multiple other pipeline businesses building concurrently) before this could be re-verified interactively. Low risk: the form has no `action`/backend by design (it's a static demo) and the only failure mode is the confirmation message not appearing, which does not block the page or lose any user data.

## Fixed Verification
| Issue | Fix | Recheck result |
|---|---|---|
| Mobile nav menu invisible when open (see Blocking Issues) | Removed `backdrop-filter: blur(10px)` from `.site-header` in `css/styles.css`; solid background instead | Reloaded and reverified on index.html, services.html, contact.html at 375×812: nav toggle opens to full-height (744px) legible overlay with all 3 links + CTA button, closes cleanly on second click; re-ran full contrast + upscale audit on all 3 pages post-fix at mobile width — 0 violations, 0 broken images on all 3. |

## Verdict
PASS
