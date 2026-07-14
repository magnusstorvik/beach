# Vikaneset — Pre-Launch Design & Code Review
**Date:** 2026-07-14
**Scope:** `index.html` (authoritative), cross-checked against `27.06.2026 Arbeid med nettside.rtf` (prior owner review notes) and the current `assets/` tree.
**Brief:** Scandinavian, quiet luxury, boutique coastal retreat, architectural, timeless, restrained. Reference points: Aman, The Audo Copenhagen, Frama, Norwegian architecture. Goal: launch-ready polish, not a redesign.

---

## SECTION 1 — Executive Summary

The current `index.html` is materially more advanced than the RTF notes suggest. Nearly every open item from the 27 June review has already been resolved — often better than literally requested (e.g. "Explore" instead of a literal "Restaurant" nav button groups the site's attractions more editorially; the bus-connection uncertainty was resolved by removing the claim rather than guessing). The bilingual EN/NO system is complete and consistent, the custom cormorant ("skarv") scroll cue is a genuinely tasteful, restrained Norwegian motif, and the auto photo-swap infrastructure (`data-photo-slot` + `assets/images/carousels/`) provides a structured path for installing the final photography.

Given that, this pass was deliberately narrow: **performance, accessibility, and code-hygiene fixes with zero visual redesign.** Six changes were implemented, all low-risk and individually justified below.

> **Correction (added after owner clarification, same day):** the initial version of this review flagged the site's glassmorphism/frosted-glass system (~50 `backdrop-filter` rules) as a mismatch against the Aman/Audo/Frama reference points and floated flattening it in a future pass. The owner has since clarified the project's actual design rulebook (`27.06.2026 Arbeid med nettside.rtf` plus a set of explicit standing rules — see the Addendum at the end of this document): restrained glass effects, layered gradients, soft depth and atmospheric lighting are an *intentional* part of this site's visual language and should be preserved, not removed for the sake of matching a more minimal external reference. That original recommendation is retracted. See Section 3 for the corrected reasoning.

The stylesheet itself also carries significant technical debt worth flagging even though it wasn't broadly rewritten: 18 sequential "refinement" comment blocks stack on top of each other, redefining the same selectors repeatedly (`:root` appears 6 times with different values for the same custom properties; `.aerial` is redefined 8+ times; `.gallery`'s background 6+ times). The page renders correctly because CSS cascade order resolves the conflicts, but the file is far larger and more fragile than it needs to be. A late `!important` override on `.nav__links a` was found during QA and consolidated into the canonical navigation rule. See Section 4.

---

## SECTION 2 — Implemented Changes

### 1. Compressed `hero.jpg` and `aerial.jpg` (performance)
**Issue:** Both were unedited camera originals — 4000×3000px, 3.36MB and 3.45MB respectively — used as full-bleed CSS backgrounds (hero covers the entire viewport; the aerial shot spans the full page width edge-to-edge). Every visitor downloaded ~6.8MB of image data before seeing anything below the fold.
**Why it matters:** First paint speed *is* part of "quiet confidence" — a slow-loading navy blank screen undercuts the premium feel the brief asks for, and guests checking the site from a phone on the Nordmøre coast may not have fast connections.
**Fix:** Resampled to 2800px on the long edge (still sharp on large/retina displays at the sizes these are actually displayed) and re-encoded at JPEG quality 58 using macOS `sips`. Result: `hero.jpg` 3.36MB → 1.19MB, `aerial.jpg` 3.45MB → 1.22MB (~65% reduction each). Verified visually at full size — no visible banding or compression artefacts; both images were re-inspected directly after compression.
**Risk:** Low — this is a lossy re-encode, although direct inspection found no obvious artefacts. The filenames and paths are unchanged, and the Git-tracked originals remain recoverable.

### 2. Fixed Gallery section's reading order (accessibility — WCAG 1.3.2)
**Issue:** The Gallery `<section>` was physically the *last* section in the HTML (after Contact), but CSS gave it `order: 6` in a flex `<main>`, visually placing it *before* "Film" (`order: 7`) and "Contact" (`order: 8`). Sighted mouse users see: Practical → Gallery → Film → Contact. Keyboard and screen-reader users, who follow DOM order (not CSS `order`), would tab/hear: Practical → Film → Contact → Gallery — reaching the contact form and footer *before* the gallery even exists in their navigation model.
**Why it matters:** This is a textbook WCAG 1.3.2 "Meaningful Sequence" failure — the presented order and the programmatically-determined order must correspond. It's disorienting for exactly the users who rely on sequential navigation most.
**Fix:** Physically moved the `<section class="gallery">` block in the HTML source to sit between Practical and Film (matching its visual position), and deleted the now-redundant `order` CSS block (`.hero{order:1}` … `.contact{order:8}`) since natural DOM order alone now produces the correct sequence.
**Risk:** Low — the intended visual order is unchanged, while the source order now matches it. Confirmed via tag-balance check and section-order grep after the edit.

### 3. Rewrote alt text on placeholder carousel images (accessibility + forward-compatibility)
**Issue:** Stand-in images awaiting real photography were labelled `alt="Temporary Sjøhus photo"`, `alt="Temporary apartment photo"`, and (identically, six times over) `alt="Temporary gallery photo"`.
**Why it matters, and why it's more than cosmetic:** The auto-swap script (bottom of `<script>`) only updates `img.src` when a matching file appears in `assets/images/carousels/` — it never touches `alt`. Left as-is, the word "Temporary" would survive silently into the *finished* site with real photography, forever telling screen-reader users the professionally-shot final images are placeholders. Separately, six consecutive identical "Temporary gallery photo" announcements gave screen-reader users no way to distinguish six different images in the main Gallery carousel today.
**Fix:** Replaced with accurate, permanent descriptions matching what's actually shown: `"Sjøhus at Vikaneset"`, `"Leiligheter at Vikaneset"`, and for the six reused Gallery placeholders — `"Aerial view of Vikaneset"`, `"The harbour at Vikaneset"`, `"Klippfiskbrygga at Vikaneset"`, `"The amphitheatre at Vikaneset"`, `"Sauna at Vikaneset"`, `"View of Vikaneset by the sea"` — consistent with the alt-text style already used elsewhere on the page.
**Risk:** None — `alt` text has no visual rendering.

### 4. Raised footer address contrast (accessibility — WCAG AA)
**Issue:** `.footer__loc` (the "Kjørvegen 31, 6520 Frei, Norway" line) was set to `rgba(240,236,229,0.38)` against the footer's near-black background (`#091f2a`). Computed contrast ratio: **~3.2:1** — below the 4.5:1 WCAG AA minimum for text this size (0.72rem / ~11.5px, not "large text").
**Why it matters:** This is real, informational content (an address), not decorative fine print — it should be legible, including for low-vision visitors.
**Fix:** Raised opacity to 0.6 → computed contrast **~5.9:1**, comfortably passing AA. Deliberately kept a notch below the adjacent "Opening spring 2027" line (0.68 opacity) to preserve the existing quiet fine-print hierarchy between the two footer lines.
**Risk:** Minimal visual change (slightly more legible grey, same hue). No other rule overrides `.footer__loc` color later in the cascade (verified by grep).

### 5. Removed 7 orphaned i18n translation keys (code hygiene)
**Issue:** `heroNote`, `pdfTitle`, `pdfBody`, `pdfButton`, `pdfMeta`, `videoTitle`, `videoBody` existed in both the English and Norwegian translation dictionaries but had zero `data-i18n` references anywhere in the markup (confirmed by grep before removal) — leftovers from when the hero had a tagline paragraph and the Film section had a second "Download PDF" card, both since intentionally removed from the HTML.
**Why it matters:** Dead data in a shipped file reads as unfinished and risks confusing whoever edits this next (they may assume `pdfTitle` etc. is live and safe to reintroduce partially).
**Fix:** Deleted all 7 keys from both language objects.
**Risk:** None — confirmed zero references before deletion; this is inert data, not markup or logic.

### 6. Moved the active navigation accent to the established gold palette
**Issue:** The active navigation item used a blue accent over a predominantly blue photographic header, weakening the state distinction and bypassing the site's restrained brass accents.
**Fix:** Changed the canonical `.nav__links a.active` colour to the existing warm-gold accent. A redundant late override was removed during QA, so the result now comes from one source rule without `!important`.
**Risk:** Low — this changes only the active navigation colour and does not affect navigation logic or layout.

---

## SECTION 3 — Recommendations Deliberately Rejected

### From the 27 June 2026 owner review notes (RTF) — already resolved, no action taken
Cross-checked every note against the current `index.html`. The following were **already fixed**, several with better solutions than literally requested, so nothing was re-applied:
- Fewer top-nav buttons ("don't need both Practical, Information and Contact") — nav now shows only Stay / Explore / Sauna & beach.
- "Restaurant" / "Sauna by the sandy beach" nav additions — implemented as "Explore" (bundles restaurant, harbour, amphitheatre) and "Sauna & beach"; more editorial than a literal restaurant call-out, matches the boutique-not-generic brief better.
- 16 km distance to Kristiansund — corrected everywhere (intro stat callout, Practical/By car).
- Gjestehavn/harbour copy — matches the requested text, real harbour photo in place.
- Amfi copy (1,500 seats, Donna Bacalao/2008) — matches verbatim, real photo in place.
- Sauna copy — matches verbatim, real photo in place.
- Sauna and laundry moved out of "Included" into a separate "Available for a fee" list — done exactly as requested.
- Uncertain bus-connection info — resolved by *removing* the unverified claim rather than guessing; the Practical/Location list now only states what's confirmed.
- Uncertainty over whether to offer a downloadable PDF — resolved by removing the PDF card from the Film section entirely (see Section 2, item 5, for the resulting cleanup).

**Two open items from the notes remain genuinely open and are content/media tasks, not code:** a black-and-white historical photo treatment behind the Klippfiskbrygga history text (the note itself says "maybe" — no archival photo exists in `assets/` to use), and the video's abrupt ending (a video-editing task, not fixable in HTML/CSS).

### The glassmorphism / frosted-glass visual system — corrected assessment, not touched
**Retracted:** the original version of this review flagged the ~50 `backdrop-filter` rules across nav, buttons, form fields, unit cards, site-feature copy panels, and the gallery carousel as a mismatch against the Aman/Audo/Frama reference points, and floated flattening them toward more matte surfaces.

The owner has clarified that this project's design rulebook is the RTF notes plus a set of explicit standing rules (see Addendum), which state directly: *"The website should continue embracing subtle visual richness... restrained glass effects, layered gradients, soft depth, atmospheric lighting... are encouraged where they strengthen the experience."* Within that rulebook — which takes precedence over generic external reference points per the owner's own instructions — the glass/depth system is intentional, not a stray trend. **No removal or flattening is recommended.** I was comparing the execution to a generic reading of "Scandinavian quiet luxury" rather than to the project's own stated rules; the correct standard is the latter.

The one caveat directly from the same rule set: *"avoid effects that feel decorative rather than intentional."* I did not find, in this pass, a specific instance of a glass/depth effect that reads as merely decorative noise rather than purposeful — the ambient-wave background motifs and layered gradients are subtle (2–9% opacity) and thematically consistent with the coastal setting. If a future pass wants to audit for decorative-vs-intentional effects specifically, that's a legitimate exercise — but it should be judged against restraint and intentionality, not against removing glass effects as a category.

### Fabricating direct URLs for Lokalkokken and sauna booking — rejected
"Find Lokalkokken" and "Sauna reservations" currently link to Google search queries rather than a direct website. I could have pointed these somewhere, but I have no verified URL for either, and guessing risks sending guests to a wrong or dead page — worse than the current honest (if unpolished) stopgap. Left untouched; the client should supply the real URLs.

### Deleting the duplicate video and orphaned PDF — flagged, not executed
`assets/videos/vikaneset-film.mp4` (24MB) is a byte-identical duplicate of `assets/video/vikaneset-film.mp4` (confirmed via `md5`) — the page only references the singular `/video/` path, so the plural folder is dead weight. Similarly, `assets/docs/vikaneset-prospectus.pdf` (208KB) is no longer linked from anywhere in the page since the PDF card was removed. Both are safe-looking deletions, but deleting files from your repository is a one-way action I chose not to take without your sign-off — this was a design/code review, not a repo-cleanup mandate. Let me know if you'd like these removed.

### Wiring up a real contact-form backend — rejected as out of scope
The enquiry form submits via `action="mailto:..." enctype="text/plain"`, a client-side pattern that fails silently on many mobile browsers and any webmail-only setup (no configured desktop mail client). Properly fixing this needs a form-handling service (Formspree, Netlify Forms, or a small serverless function), which requires an account/infrastructure decision from you — not something to wire up unilaterally in a static HTML file. Flagged in Section 4.

### A full CSS de-duplication pass — rejected for this session
See Section 4 for the full description of the issue. Not attempted here: too large a surface (18 stacked revision layers, dozens of repeated/contradictory selectors) to safely rewrite without live visual regression testing, and explicitly beyond what was asked ("do not rewrite functioning code unnecessarily").

---

## SECTION 4 — Remaining Opportunities (Future Iterations)

1. **CSS consolidation — code health only, not a visual change.** The `<style>` block contains 18 sequential "refinement" comment markers (`/* Final layout refinements */`, `/* Editorial refinement */`, `/* Balanced visual system */`, … `/* Media-safe frosting */`), each re-declaring selectors already set earlier: `:root` is redefined 6 separate times (with genuinely different values for `--navy`, `--stone`, `--text`, etc. across passes), `.aerial` 8+ times, `.gallery`'s background 6+ times. The page works today because cascade order resolves it, but the stylesheet is roughly 2–3× the size it needs to be, and every future edit risks fighting an earlier layer instead of a clean rule. That cascade fragility had already encouraged a late `!important` navigation patch; QA consolidated that specific rule, but the broader debt remains. **Important:** any consolidation pass must reproduce the current computed styles exactly (including the glass/depth effects confirmed intentional above) — this is strictly a dead-rule removal exercise, not an opportunity to simplify the visual language. Recommend doing it with live visual-regression screenshots (unavailable in this session) so "no visual change" can actually be verified rather than assumed.
2. **Carousel replacement probes.** The current loader tries every `data-photo-slot` path even though `assets/images/carousels/` is not present yet, producing 32 failed requests in the current asset state. Before launch, either provide an explicit available-file manifest or enable this development convenience only while replacement photography is being installed.
3. **Real photography rollout.** Sjøhus, Leiligheter, and 6 of 14 Gallery slots still show generic stand-in photos (marked `placeholder-media`, auto-swapped once correctly-named files land in `assets/images/carousels/` per `PHOTO-NAMING.md`). Content/production dependency, not a code issue.
4. **Direct URLs** for Lokalkokken and sauna booking, replacing today's Google-search-query links, once you can confirm the correct destinations.
5. **Contact-form backend** (Formspree / Netlify Forms / serverless function) to replace the fragile `mailto:` submission — a real risk of silently lost enquiries on mobile.
6. **Duplicate/orphaned assets** — `assets/videos/vikaneset-film.mp4` (24MB duplicate) and `assets/docs/vikaneset-prospectus.pdf` (208KB, unlinked). Safe to delete once you confirm.
7. **Historic black-and-white photo treatment** behind the Klippfiskbrygga history copy, as tentatively suggested in the original review notes — contingent on an archival photo actually being available.
8. **Video's abrupt ending** (flagged in the original review notes) — a media-editing task, not a code fix.
9. **General image compression** on the remaining gallery photos (340KB–1MB each, all lazy-loaded) — lower priority than hero/aerial since they're not render-blocking above the fold, but a pass would still help overall page weight.
10. **No live browser verification was possible in this session** (no headless Chrome/Playwright available, no screen-recording permission for a real screenshot). Every change above was verified through direct visual inspection of the compressed images and careful static HTML/CSS-cascade tracing, but a manual pass in an actual browser — both languages, desktop and mobile widths — is recommended before publishing.

---

### What's working well and was deliberately left unchanged
- The bilingual EN/NO system: complete, consistent, no missing keys (after cleanup).
- The cormorant ("skarv") scroll-cue animation — a genuinely restrained, specific-to-place motif; resist the urge to replace it with a generic chevron.
- The reduced top nav (Stay / Explore / Sauna & beach) — correctly implements the "fewer choices" note.
- The auto photo-swap infrastructure (`data-photo-slot` + `PHOTO-NAMING.md`) — a low-maintenance way to let real photography land without further engineering.
- The current Booking.com, Lokalkokken and sauna destinations were preserved as placeholders pending confirmed production URLs.
- The restrained glass/depth visual system (tinted overlays, layered gradients, glass effects, soft shadows) — confirmed intentional by the owner; not a candidate for removal or flattening (see Section 3 correction and Addendum below).

---

## ADDENDUM — Standing Project Rules (received same day, after initial review)

The owner supplied additional ground rules that govern this project going forward and take precedence over generic web-design conventions or external reference points (Aman/Audo/Frama) wherever the two conflict. Recorded here so future edits — by any agent — start from the same rulebook rather than re-deriving it:

1. **`27.06.2026 Arbeid med nettside.rtf` is the project's design rulebook.** Its principles take precedence over general conventions unless they clearly conflict with usability or accessibility.
2. **Exercise restraint editing existing copy.** Assume the current writing has been carefully considered. Only edit text for a clear improvement in clarity, rhythm, accuracy, or professionalism — never a rewrite just because it "could" read differently. Avoid marketing language, exaggerated claims, or generic luxury vocabulary.
3. **Avoid unnecessary repetition** of the same attraction/selling point across multiple sections without a compelling reason (e.g. Donna Bacalao should only appear in the Amfi section). **Checked in this pass — see below: already compliant.**
4. **Preserve tasteful, intentional visual effects** — tinted overlays, layered gradients, restrained glass effects, soft depth, gentle transitions, atmospheric lighting, careful masking, refined hover states. Do not strip these for the sake of minimalism. Only flag an effect if it reads as *decorative rather than intentional*.
5. **Controlled complexity, not mechanical cleanliness.** Slight asymmetry, gentle overlap, and controlled visual tension are preferable to excessive symmetry where they'd feel more natural.
6. **Professional confidence.** Don't redesign a section that already communicates effectively just to demonstrate activity — if a section works, say so explicitly and leave it.
7. **Respect project maturity.** This site is in its final refinement stage. Prioritise polish over invention; small, consistent improvements beat introducing new concepts.
8. **Avoid overcorrection.** Don't maximise consistency for its own sake — some of the strongest design decisions are carefully judged imperfections and variation in rhythm. Optimise for harmony, not uniformity.

**Verification performed against rule 3 (repetition):** grepped the full file for "Donna Bacalao," "1,500 seats," and "premiere/urpremiere." All three appear exactly once in the markup (Amfi feature body) plus once in each of the EN/NO translation-dictionary entries for that same string — which is the i18n system's normal one-fact-two-languages structure, not cross-section repetition. No other section (Site intro, Hero, Practical) restates the opera/amphitheatre fact. **Confirmed compliant — no change needed.**
