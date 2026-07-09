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

## Reviewer Verification (independent, REVIEW phase)

Re-ran both shared audit scripts (`contrast-audit.js`, `upscale-audit.js`) verbatim via `preview_eval` against the live dev server (port 4228), independently, once per page at all three breakpoints (desktop 1280x800, tablet 768x1024, mobile 375x812), with `[data-reveal]` force-set to `.is-visible` first.

| Check | Result | Evidence |
|---|---|---|
| Upscale audit — all 3 pages × 3 breakpoints (9 runs) | PASS | index.html: 5 images/run, services.html: 3 images/run, contact.html: 2 images/run. 0 violations, 0 brokenImages, 0 aspectMismatches on every run. |
| Contrast audit — all 3 pages × 3 breakpoints (9 runs) | PASS | index.html 81 checked/run, services.html 65/run, contact.html 54/run. 0 violations on every run. |
| Gradient manual check (`.btn-primary`, `.cta-band`) | PASS | Sampled both gradient endpoints (`--cinnabar` oklch 45% and `--cinnabar-deep` oklch 33%) against `--parchment` text (oklch 96%) using the browser's own OKLCH→sRGB conversion. Ratios: 7.28:1 and 10.87:1 — both clear WCAG AA (4.5:1) with margin. |
| Mobile nav toggle — independently re-verified, not trusted from build notes | PASS | `preview_click` (the automation tool) did not reliably register clicks on `.nav-toggle` in this session — 0/2 attempts fired the handler despite reporting "Successfully clicked" (see Reviewer Tooling Notes below). Switched to a full synthetic pointer/mouse event sequence (`pointerdown`→`mousedown`→`pointerup`→`mouseup`→`click`) dispatched at the toggle's real screen coordinates, which is a much closer approximation of a genuine user click. Result: nav opens to `position:fixed`, full viewport width (375px) and height (744px, viewport minus header), solid `oklch(0.96 0.02 80)` background, `backdrop-filter:none` confirmed on both `.site-header` and `.main-nav`, `pointer-events:auto`, all 3 nav links + CTA visible and correctly positioned within the open panel. Closes correctly on a second toggle click (`is-open` removed, `pointer-events:none`). Also visually confirmed via screenshot (see below). Fix from the prior build session holds up under independent re-test. |
| Booking form submit handler — independently click-tested, not just code-reviewed | PASS | Filled `#name`/`#email`, then dispatched the same full pointer/mouse event sequence on the submit button (not `.click()`/`requestSubmit()`, to exercise the real event path). Result: `location.href` unchanged after submit (confirms `preventDefault()` fired — a real GET submit would have reloaded/reset the page and cleared the field), `#name` value persisted ("Jane Test"), and `.form-status` populated with the exact expected text ("Thanks — this demo form doesn't send yet. Please call 07540 776059 or email drliacupunctureclinic@gmail.com to book."). The advisory gap in the original QA_REPORT is resolved — this is now a live-tested PASS, not a code-review-only inference. |
| Two-location business scope — independently re-verified against the live business site, not just BUILD_BRIEF | PASS | Fetched `acupuncturelondonclinic.com/copy-of-london` and `/copy-of-visit-us` directly (not relying on BUILD_BRIEF's cached research). Confirmed: real second clinic at 4a Church Road, Welwyn Garden City AL8 6NE, Mon & Thu 9am–5pm, reachable via a live "Visit Us" submenu alongside the Belsize Park address. On the demo site: title tag, header subtitle ("Belsize Park · NW3"), and hero copy all scope explicitly to Belsize Park; the Welwyn clinic is honestly disclosed (not hidden) with its own address/hours in a dedicated Contact-page info block, a footer note on every page, and as a selectable option in the booking form's "Preferred clinic" dropdown. Reads clearly as "a redesign of the Belsize Park clinic's page," not a full company site erasing the second location. |
| Fabricated facts — independently re-verified against live source, not just BUILD_BRIEF's table | PASS | Fetched `acupuncturelondonclinic.com/about` directly and cross-checked every credential/timeline claim on the demo (Chengde Medical University 1990–1995, Dacheng Hospital TCM doctor 1995–2002, Tianjin Cancer Hospital chemotherapy certificate 2002–2003, Sheffield clinic 2004–2010, ATCM member since 2005, Belsize Park clinic since 2010, Qigong training from 1990/Master status by 1995) against the live page. All match exactly. No price, no London opening hours, and no star rating are stated anywhere on the demo, consistent with BUILD_BRIEF's Do Not Claim list. |
| Text overflow (contact.html real content, mobile+tablet) | PASS | `scrollWidth` vs `clientWidth` on `.brand-text strong/span`, `.info-list .value` (full address + full email), `.footer-col li`, `.footer-bottom span` — 0 overflowing elements at 375px and 768px. |

### New blocking issue found (missed by build-time QA)

**Image/copy mismatch on `services.html`, `.bio` section.** The section's eyebrow/heading/body copy is entirely about Qigong and meditation ("Qigong & meditation training" / "Beyond the treatment room." / "Alongside acupuncture and herbal medicine, Dr Li teaches Qigong — a moving-meditation practice..."), but the adjacent photo is `images/dr-li-portrait-clinic-counter.jpg` — a generic portrait of Dr Li at the clinic counter/herbal cabinet, with alt text "Dr Li at the clinic counter in front of the herbal medicine cabinet at Winchester Road." The photo has no connection to Qigong or meditation. Confirmed on the live rendered page via DOM inspection (image src, alt, and adjacent heading/body text read directly from `.bio` on `services.html`), not just source reading.

This appears to be a copy-paste artifact: the same `.bio` section markup is reused correctly on `index.html` (clinic-counter portrait paired with "About Dr Li" / training-history copy — a good match), but on `services.html` the copy was swapped to Qigong-specific text without swapping or reconsidering the image. BUILD_BRIEF.md's own Asset Manifest states this image's "Intended section" was "Services page practitioner portrait" matching "bio/treatments intro copy" — the copy that actually landed there is Qigong copy, not treatments-intro copy, so the build deviated from its own plan.

Required fix (per AGENTS.md step 5's image/copy-match rule) — pick one:
1. Rewrite the `services.html` `.bio` section copy to be about Dr Li's training/qualifications (matching the image), and move the Qigong-specific copy elsewhere or drop it from this page (it's already covered on the homepage's Qigong section with the correct meditation photo).
2. Drop the image from this section on `services.html` and run it as text only, per AGENTS.md's "if nothing in the real asset library fits a section, cut the image" guidance — the meditation photo is already deliberately used once (homepage) per BUILD_BRIEF's "only 3 photos, each used once" design decision, so reusing it here recreates the repetition problem the build was trying to avoid.
3. Source a new, genuinely Qigong/meditation-relevant photo for this section if one exists in official channels not yet checked.

### Reviewer tooling notes (not site bugs — logged so the next reviewer doesn't waste time)
- `preview_resize` with the `desktop` **preset** returned a broken viewport in this session (`window.innerWidth`/`document.documentElement.clientWidth` both read `0`, causing a false-positive-looking `width:0` on the header logo `<img>`). Passing explicit numeric `width`/`height` (e.g. 1280×800) fixed it immediately. Confirmed this was a tooling artifact, not a real layout bug, by re-measuring the same element after the explicit resize (correct 42×42).
- `preview_click` did not reliably dispatch working click events against this page in this session (confirmed with an instrumented listener: two separate `preview_click` calls on `.nav-toggle` produced zero listener firings, while a same-target synthetic `pointerdown/mousedown/pointerup/mouseup/click` sequence via `preview_eval` worked every time). This matches what the builder reported for the booking-form submit click. Recommend future reviewers default to the synthetic event-sequence pattern for click verification in this shared preview pool rather than re-trusting `preview_click`'s "Successfully clicked" response at face value.

## Verdict
BLOCKED — one blocking image/copy mismatch found on `services.html` (see above). Everything else independently re-verified as PASS: contrast, upscale/broken images (all 3 pages × 3 breakpoints), text overflow, fabricated-facts cross-check against live source, two-location scoping against live source, mobile nav fix, and booking form submit handler (both live-click-tested, not just code-reviewed).

## Fix verification (FIX phase, 2026-07-09)

| Check | Result | Evidence |
|---|---|---|
| `services.html` `.bio` image/copy relationship | PASS | Replaced the Qigong-specific eyebrow, heading, body and CTA with verified clinical-background copy. The section now shows the official clinic-counter practitioner portrait beside `Dr Li's background` / `The training behind the treatment room.` and a sourced training timeline. The Qigong copy remains once only on the Home page, beside the official Qigong-meditation photo. |
| Facts introduced by the rewrite | PASS | Every timeline item is in `BUILD_BRIEF.md` Allowed Facts: Dacheng Hospital (1995–2002), Tianjin Cancer Hospital certificate (2002–2003), Sheffield practice (2004–2010), Belsize Park clinic since 2010, and ATCM membership since 2005. No treatment price or London hours introduced. |
| Image and contrast regression risk | PASS | The edited section retains the previously independently audited image element, wrapper, classes and CSS without alteration. The REVIEW pass ran the shared contrast and upscale scripts on `services.html` at 375×812, 768×1024 and 1280×800: 0 contrast violations; 3 images per run; 0 upscale violations, broken images, or aspect mismatches. This fix changes only inherited solid-background text and does not introduce a new image, colour, gradient, or layout rule. |
| Browser re-run availability | N/A | The local browser runtime had no available browser binding in this FIX session, so a duplicate rendered audit could not be started. This is recorded rather than represented as a live re-run; the full independent review evidence above remains the live gate record for the unchanged rendering rules. |

## Final verdict
PASS — the sole blocker is fixed. The services-page portrait now illustrates adjacent practitioner-background copy, while Qigong content remains paired with the Qigong image on the Home page. No new factual, image-quality, or contrast risk was introduced.
