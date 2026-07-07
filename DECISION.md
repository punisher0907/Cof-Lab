# DECISION.md — Coffee Lab design & build decisions

A log of every consequential choice made while turning the wireframes + spreadsheet into a working site, and the reasoning behind each.

---

## 1. Product framing

**Decision:** Position the site as *"one place to know everything about coffee in India"* with three jobs, straight from the wireframes: **browse** (directory), **match** (quiz), **learn** (Coffee 101 + stories).

**Why:** The wireframes' nav (Coffee Directory · Find my Match · Know your coffee · Blogs/Stories) already encodes this. Every page and the homepage's section order follow that hierarchy: search first, browse second, self-identify third ("how do you like your coffee?"), learn last.

## 2. Architecture: zero-build static site

**Decision:** Plain HTML + CSS + vanilla JS. The dataset is compiled into `js/data.js` (a `const COFFEES = [...]`) rather than fetched as JSON.

**Why:**
- "Production-quality **local** website" means it must run with zero setup. Embedding data as a script sidesteps `fetch()`/CORS restrictions on `file://`, so double-clicking `index.html` works — no server, no build, no dependencies.
- Vanilla JS at this scale (~99 records, 7 pages) is faster and more maintainable than a framework; there is nothing a framework would earn here.
- Detail and article pages use query params (`coffee.html?id=…`, `article.html?id=…`) instead of one page per coffee — 99 coffees would mean 99 files to regenerate on every data change.

**Trade-off accepted:** No SSR/SEO per coffee page. For a local directory tool this is irrelevant.

## 3. Data pipeline

**Decision:** `build_data.py` holds the raw sheet rows and derives structured fields: `roastLevel` (1–5), `states[]`, `families[]` (flavour families), `styles[]` (milk/black/cold suitability), `brews[]` (suggested methods), stable `id` slugs.

**Why:** The sheet is human-authored (e.g. "Natural + Aged", "Chikkamagaluru, Kodagu, Karnataka", flavours separated by •). Filters and the quiz need normalised axes. Deriving them in one script keeps the site JS dumb and the mapping auditable/regenerable.

**Derivation rules (documented so they can be argued with):**
- `roastLevel`: Light=1, Light Medium=2, Medium=3, Medium Dark=4, Dark=5, Vienna Roast=5.
- `styles`: roast ≥ Medium → suits milk; roast ≤ Medium Dark → suits black; fruity/natural/light-medium coffees → suit cold brew. These are the standard cupping heuristics, also explained to users in the "Roast levels" guide so the logic is transparent, not magic.
- `families`: keyword match over flavour notes into six families (fruity, floral, chocolatey, nutty, sweet, spicy). Drives filters, similar-coffee scoring and the generated artwork palette.
- Two Alchemist rows had empty flavour cells in the sheet; they were given minimal honest descriptors ("Bold • Rich" etc.) rather than left blank, since empty flavour rows break the card layout's information promise. Bili Hu's three distinct "Balur Estate" rows and two "100% Arabica" rows were disambiguated in the name (e.g. "Balur Estate Honey (Dark)") because identical names with different specs are a UX bug, not a data feature.

**Known limitation:** The sheet's public HTML render caps at 100 rows, so the dataset covers the roasters A–B (729 Grams → Bloom, 99 coffees, 9 roasters, 5 states). The pipeline is one script; appending further rows and re-running `python3 build_data.py` extends the site with no code changes.

## 4. Visual direction — "estate ledger meets modern tasting room"

**Decision:** A deep-green/roast-brown/marigold system on pale leaf-tinted paper, instead of the default "specialty coffee site" look (cream background, terracotta accent, all-serif).

**Why:** The subject supplies its own palette: the Western Ghats canopy (`--canopy #1F3D2B`), the roasted bean (`--roast #2B1D14`), marigold/turmeric (`--marigold #DFA126`) as the Indian accent, and coffee-cherry red (`--berry`) used only for destructive/removal affordances. Green-forward coffee sites are rare, which makes the direction ownable; it also honestly reflects that Indian coffee is *shade-grown forest coffee* — the site's editorial repeatedly makes this point, so the design should too.

**Typography:** Fraunces (display serif with real character at heavy weights) for headlines; Archivo (grotesque) for UI/body; IBM Plex Mono for the "ledger" voice — spec labels, counts, prices, eyebrows. The mono face is doing the brand work: coffee bags and cupping sheets are full of small technical type, and the site borrows that vernacular. Fonts load from Google Fonts with full system fallbacks, so the site still works offline.

**Signature elements (the one place boldness is spent, twice):**
1. **The roast meter** — five beans filled light→dark, used identically on cards, filters, quiz and detail pages. It turns the dataset's most decision-relevant field into a glanceable, language-free scale.
2. **Generated bag art** — every coffee gets a deterministic SVG "coffee bag" whose colours come from its lead flavour family and whose texture pattern (dots/ridges/leaves) is hashed from its id. Reason: the sheet has no product photos, hotlinking roaster images would be fragile and legally murky, and placeholder grey boxes would sink the whole design. Generated art keeps cards visually distinct *and meaningful* (fruity coffees look warm-red, spicy ones green, sweet ones golden) with zero image assets.

## 5. Homepage

**Decision:** Hero = thesis headline + search + live dataset stats, over an SVG Western-Ghats ridge line; then Explore (8 featured), "How do you like your coffee?" (3 style cards), Coffee 101 (3 guides).

**Why, per wireframe:** This mirrors Desktop-1's exact section order. Deviations:
- Stats (99 coffees · 9 roasters · 87 estates · 5 states) are computed from the data at runtime — honest numbers, never stale.
- Featured picks rotate daily (seeded by date) and are drawn one-per-roaster so no brand dominates the shelf.
- The wireframe's card tags ("roast/intensity") became the roast meter + process/sourcing tags, which carry more information in the same space.

## 6. Directory

**Decision:** Persistent filter sidebar (desktop) / full-screen filter sheet (mobile), with facet counts, active-filter chips, sort, live result count, and URL-driven entry points (`?q=`, `?style=milk`, `?sample=1`).

**Why:** The Filter.png wireframe shows checkbox groups; the additions are standard directory ergonomics that the wireframe implies but doesn't draw:
- **Counts next to every option** answer "is it worth ticking this?" before the click, and options with zero remaining results hide themselves.
- **Chips** make the current query legible and individually removable — the wireframe's filter state was otherwise invisible once the panel closed.
- **"Your cup" (milk/black/cold) is the first facet**, above roast — beginners think in cups, not roast curves. The homepage style cards deep-link into it.
- Price is a single "max price" slider rather than min+max: the realistic question is "what's my ceiling?", and every price is normalised to ₹/100 g (a decision inherited from the sheet, surfaced in the label).

## 7. Coffee detail page

**Decision:** Two-column layout: sticky generated art left; name, roast meter, flavour pills, price, and an **"Estate ledger"** spec table right; "Taste neighbours" (4 similar coffees) below.

**Why:** The spec table is titled and styled as a ledger (mono labels, dashed rules) — this is the signature direction doing functional work: eight fields of provenance data (estate, district, elevation, varietal, process, fermentation…) presented as the proud technical document it is, instead of an apologetic bullet list. Missing sheet values render as "—" rather than being hidden, because in a directory an explicit unknown is information.

**Similar-coffee scoring:** shared flavour family (+2), same roast level (+2, adjacent +1), identical flavour note (+3), different brand (+1, to encourage discovery). Transparent, tunable, no ML pretensions.

## 8. Find my match quiz

**Decision:** Five questions (cup style → gear → flavour direction → adventurousness → budget), one screen each with progress beans, then a "match report": top 3 with rank badges and a *"Why"* line, plus 4 runners-up. Max two picks per brand.

**Why:**
- The questionnaire wireframe shows exactly this pattern (icon option cards, one question per screen, back/next).
- **Every match explains itself** ("holds up beautifully in milk; leads with dark chocolate and maple syrup"). Recommendation UIs earn trust through reasons, not scores; the reasons are assembled from the same rule hits that produced the score, so they can't drift from the truth.
- Budget is a hard-ish constraint (heavy penalty, not exclusion) so a spectacular ₹20-over coffee can still surface.
- "Adventurousness" maps to the dataset's genuinely distinguishing axis — India's experimental fermentation scene (anaerobic, barrel-aged, kombucha) versus classic washed lots — rather than a generic "mild/strong" question.

## 9. Learn & Stories

**Decision:** Six original guides (roast, process, species, brewing, regions, label-reading) + three stories (Baba Budan, Monsooned Malabar, forest coffee), all rendered from a single content file through one article template with prev/next chaining.

**Why:** The wireframes call for "Know your coffee" and "Blogs/Stories" as separate nav items with the same reading experience. One `ARTICLES` array with a `type` field keeps that true with zero duplicated code. Content is written to serve the directory: every guide explains a field that appears on coffee cards (roast meter, process tag, varietal line, ₹/100 g), so learning loops back into browsing.

## 10. Accessibility & quality floor

- Semantic landmarks, real `<button>`/`<a>` semantics, `aria-current` nav state, `role="radiogroup"` in the quiz, `aria-label`s on icon-only controls, visible `:focus-visible` outlines.
- Roast meters carry `role="img"` + text labels; generated art has descriptive `aria-label`s.
- `prefers-reduced-motion` disables all transitions/animations.
- Responsive to ~360 px: nav collapses to a toggle, filters become a sheet, detail stacks, quiz options go single-column.
- All user-influenced strings pass through an HTML-escaping helper before rendering.

## 11. Things deliberately *not* built

- **Cart/checkout** — the sheet has "LINK" placeholders, not real product URLs; a fake buy flow would be dishonest. "More from {roaster}" search stands in.
- **Framework, bundler, package.json** — nothing here needs them; their absence *is* the local-first feature.
- **Hotlinked roaster logos/photos** — fragile, unlicensed; replaced by the generated-art system.

## 12. Revision round 1 (user feedback)

- **Header CTA:** "Find my match" moved out of the plain nav into a persistent marigold button in the header on every page (visible on mobile too, next to the menu toggle) — it is the site's primary action and now reads as one.
- **Mobile home shelf:** the featured coffee listing becomes a snap-scrolling horizontal shelf under 720 px (`.scroll-mobile`), cutting the homepage's mobile length by ~8 card heights while keeping all 8 picks reachable.
- **Detail-page CTAs:** "Buy from website" is now the primary button, "More from {roaster}" demoted to secondary. The sheet's link column isn't exposed in its public render, so the buy button opens a precise web search for the exact roaster + coffee (new tab) — honest routing to the real product page without inventing URLs. If real product links become available in the data, `detail.js` swaps them in at one line.

## 13. Revision round 2 — cinematic minimal homepage

**Brief:** minimalist + bento + gradients + parallax; Apple-style scroll-scrubbed hero from the supplied bean photograph; forhers.com as the tonal reference; quiet copy.

- **Frame sequence:** the scrub needs frames, and one photo was supplied — so `gen_frames.py` renders 72 cinematic frames (1366×768, ~6 MB total) from `HeroImage.jpg`: a smoothstep-eased push-in from a wide, dark establishing crop into the heart of the bean pile, with an exposure/saturation ramp. Regenerable in one command.
- **Scrub engine (`js/home.js`):** 340vh pinned section, sticky full-viewport canvas, scroll progress → frame index, cover-fit drawing, DPR-aware, draws only when scroll dirties the frame (single rAF loop). Contiguous-preload strategy shows the nearest loaded frame while the sequence streams in.
- **Text choreography:** three stages hand off across the scrub — "Every Great Indian Coffee," → "made easy to find." → CTAs (Find my coffee / Surprise me) — driven by the same progress value via trapezoid fades, so text and film can never desync. Copy trimmed to the requested quiet register across every section.
- **Sections (per brief order):** cinematic hero → Popular picks (six openers, one per roaster) → How do you like your coffee (three gradient tiles) → Why Indian specialty coffee (bento: 4 wireframe points + parallax photo tile + 400-years stat tile) → Roasters (nine monogram tiles) → Newsletter CTA (front-end only success state; no backend on a static site).
- **Parallax:** gradient orbs and the bento photo drift on `[data-speed]`; disabled under `prefers-reduced-motion` (hero then shows the final stage statically).
- **Header:** transparent over the dark film, resolves to solid paper past the hero.
- **Bugs caught in browser verification, then fixed:** stage-1 headline was invisible at exactly scroll 0 (fade-in window began *at* 0); decorative orbs widened the page 140px (fixed with `overflow-x:clip` on `main` — `clip`, not `hidden`, so `position:sticky` survives); a CSS-relative image path 404'd.
- **Verified in Chromium before shipping:** frame index tracks scroll (0 → 36 → 71), canvas pixel signatures change between scroll positions, stage opacities hand off 1→2→3, header mode flips, all six pick cards reveal, parallax transforms move, roasters/cups render, newsletter succeeds, Surprise-me routes to a real coffee, mobile scrub reaches the final frame, zero horizontal overflow at 1440/1024/390px.
- The previous homepage is preserved as `index.v1.html.bak`.

## 14. Revision round 3 — full-sheet re-audit

Re-pulled the Google Sheet in full and diffed it against the dataset.

- **The 100-row wall, confirmed:** Google's no-JavaScript render of a Sheet serves exactly the first 100 rows — the fresh pull ends at row 100 (coffee #99) mid-alphabet, and the CSV-export and mobile endpoints are not fetchable from this environment. Rows beyond (roasters after Bloom) are unreachable via the link alone; they require the sheet's own CSV/XLSX export.
- **Corrections from the diff:** three Alchemist rows (Balanoor Washed, Kalledevarapura Honey, Kalledevarapura Espresso) have *blank* Source Classification in the sheet — previously filled as "Single Estate", now honestly blank and rendered as "—". Confirmed as sheet-faithful: the unnamed Bloom row (named editorially), empty flavour cells on four Alchemist rows (filled editorially), three identically-named Bili Hu "Balur Estate" rows (disambiguated), and the Link column being all placeholders ("LINK") — which validates the search-based buy button.
- **Pipeline upgraded for the full sheet:** `build_data.py` now ingests the sheet's own CSV export directly (`python3 build_data.py export.csv`) — header-driven parsing, skips divider rows, normalises prices, keeps real product URLs from the Link column and drops "LINK" placeholders. Verified with a synthetic export containing a beyond-row-100 brand and a live URL. The detail page's "Buy from website" now prefers a coffee's real `link` when the data has one.
- The sheet also contains a second tab ("Legend"), equally behind the render cap.

## 15. Revision round 4 — full dataset (613 coffees, 58 roasters, 9 states)

The complete sheet arrived as a CSV export and is now bundled at `data/directory.csv` — the default build source. Full-sheet realities handled:

- **Roast coverage:** 13 distinct values in the wild. Mapped: City→Medium, Omni Roast→Medium (designed for every brew), French/Italian→Dark. "Custom" (22) and blank (29) become honest **level 0 — "Not rated"**: empty-bean meter, open milk/black/cold suitability, generic brew suggestions, excluded from roast filtering rather than mislabelled Medium, and the quiz skips roast-fit rules for them (a to-order roast can be roasted to fit).
- **Price sanity:** "900 for 75 grams" (the Riverdale Geisha) is parsed and normalised to ₹1,200/100 g instead of the ₹90,075 a naive strip produced. Two coffees with no listed price show "price on request", always pass the price ceiling, and sort last — not dropped, not zero.
- **Geography:** state extraction rebuilt around a named-state search (the comma heuristic missed districts that *are* states). The map now spans 9 states including Nagaland, Meghalaya, Tripura and Himachal Pradesh; "Andra Pradesh" and "Tamilnadu" are normalised.
- **Duplicate names:** identical brand+name rows (4× "Ratnagiri Estate Estate", 2× "Unakki Dark Honey", etc.) are auto-disambiguated by process, then roast, then lot number — no more four indistinguishable cards.
- **Blank cells at scale:** 71 coffees ship no tasting notes → cards say "notes not published" instead of an empty line, detail pages show a single honest pill, art falls back to a neutral roast palette, and the quiz simply can't flavour-match them (correct).
- **Process facet:** now includes an Experimental bucket (funk / smoked / dark-room lots); pulp-sun-dried folds into Honey.
- **UI at 6× the data:** roaster facet (58 entries) scrolls within the sidebar; homepage roaster grid curates the top 11 by shelf size plus a "+47 more" tile; all hardcoded counts ("Ninety-nine beans", "All 99") are now computed from the data. Measured: 613-card directory renders in ~1.2 s with ~0.3 s filter response — acceptable without pagination.
- Re-verified end-to-end in Chromium on the full dataset: hero scrub to frame 71, directory filters/sort (nulls last, ₹1,200 max), Davrah's unrated coffees render correctly, quiz returns 3 diversified matches from 613, zero page errors.

## 16. Revision round 5 — imagery, Manrope, partner page

- **Hero becomes a three-act film:** the 72-frame scrub is regenerated as bean pile push-in → roastery pan (hero2) → estate reveal (Hero3), with 6-frame smoothstep crossfades between acts. The acts land on the three text stages, so the story reads bean → roast → origin as you scroll. The estate act is darkened ~26% in-frame (plus a headline text-shadow) so white type stays readable over bright green; verified luma behind the closing headline ≈ 86.
- **Cup cards get photography:** Milk/black/cold tiles now use the supplied photos as covers with a bottom scrim for legibility and a gentle hover zoom; icons and counts sit above the image.
- **Bento copy** replaced with the provided four points verbatim (one apostrophe fix: "Its Own Signature"); the grid moved to `minmax` row heights after the longer copy overflowed the fixed 172px rows — caught by an overflow assertion, not by eye. Layout is now a clean 5-tile composition; point 04 takes the gold tile.
- **Newsletter** backdrop swapped to the supplied roastery photograph.
- **Navigation:** "Stories" is replaced by **Become a partner** on every page; stories remain reachable from the footer. `partner.html` is a new registration page — perks strip plus a validated form (brand, contact, email, city required; inline error states; success panel greeting the brand by name). It's front-end only on a static site: submissions render a confirmation but aren't transmitted anywhere — wire the form to a backend/Formspree/Google Form endpoint when one exists.
- **Typography:** body text is now self-hosted **Manrope** (all seven supplied weights, `font-display:swap`, TTF `@font-face`); Fraunces stays for display, IBM Plex Mono for the ledger voice; Archivo removed from every page's font request.
- Verified in Chromium: act colors at scroll start/middle/end ([51,33,20] → [121,111,100] → [91,99,55] green-dominant), end frame 71, Manrope regular/bold/extralight all active, five bento tiles zero overflow (desktop + mobile), cup and newsletter images resolve, nav swap on all six pages, partner form flags 4 invalid fields then succeeds, zero horizontal overflow at 390px.

## 17. Revision round 6b — real header photography

The actual header images arrived (`directory.jpg` beans on wood, `knowcoffee.jpg` pour-over brewing, `becomepartner.jpg` coffee cherries on the branch — a fitting grower-facing image). Each gets the standard banner treatment: 3.4:1 crop, brightness pulled to ~0.6–0.68 under the green-tinted scrim so white headlines stay readable. The generated placeholders are deleted; references now point at the supplied photos (60–73 KB each after optimization, ~25× lighter than the PNG placeholders).

## 18. Revision round 7 — case simplified, marigold headers, perk icons

- **"The case" section** dropped the pinned scroll-scrub for a plain horizontal snap-scroller: same four cards, native sideways scroll with a styled thin scrollbar, no page hijacking. Two alignment bugs surfaced and were fixed under test: the scroller's edge padding was 24px short of the `.wrap` content edge (box-sizing math), and `scroll-snap` with no `scroll-padding` yanked the first card to x=0 on load — `scroll-padding-inline` now mirrors the edge padding, so the section heading, the other homepage headings, and the first card all sit at exactly the same x (verified 124px = 124px = 124px at 1440, aligned on mobile too). The scrub engine was removed from `home.js`.
- **Photo-header headlines** (Coffee directory / Know your coffee / Become a partner) switched to marigold for legibility over the photographs; first attempt lost the cascade to a later `.dir-hero h1{color:#fff}` rule at equal specificity — caught by the color assertion, fixed with a compound selector.
- **Partner perks** got three line icons in the house style (bean-magnifier, provenance ledger, free-listing tag), canopy strokes with marigold accents.

## 19. Revision round 8 — case cards: photos, no numbers, mouse scrubbing

- All four "case" cards are photo cards with the dark scrim treatment; the big numerals are gone and the unused colour-variant styles were cleaned out.
- **`card1.jpg`–`card4.jpg` were named in the request but did not arrive in the upload**, so the four files currently in `assets/img/` are semantically-matched crops from the existing photography (bean spread → flavour, estate → regions, pour-over → heritage, roastery → new wave). They carry the exact requested filenames: dropping the real images over them updates the site with zero code changes.
- **Mouse scrubbing:** the row is now grabbable — pointer-drag scrubs it (cursor grab/grabbing, snap suspended mid-drag and eased back on release) and a vertical mouse-wheel over the row scrubs it horizontally, with a no-trap rule: at either end the wheel hands control back to the page. Touch keeps native scrolling.
- **Padding:** symmetric breathing room on both ends of the scroller (edge + 8px), verified equal left/right (132px at 1360w).
- Verified: all four images 200, drag moves scrollLeft 0→490, wheel likewise, page regains scroll at the track end, hero scrub and overflow regressions green.

## 20. Revision round 8b — real card imagery

The four card images arrived: café cups (flavour), plantation rows (regions), a Turkish-coffee tray with a vintage photograph (heritage — a lovely match), and a modern pour-over bench (new wave). Each processed to the 1.25:1 card crop at ~50–150 KB; the brighter two eased down ~10% so the scrim keeps text contrast. Placeholders overwritten in place, zero code changes — exactly the drop-in path designed for.

## 21. Revision round 9 — brand pages

- New `brand.html?brand=…` template (the wireframe's Desktop-4 layout): sticky logo card, name, location chip, bio, live stats (coffees / estates / from-price / sample packs) and an "Explore their products" grid. One template serves **all 58 roasters**: `js/brands.js` holds per-brand editorial overrides (logo + custom bio), everything else is derived from the dataset with a sensible auto-bio fallback and a monogram in place of a logo.
- **Araku** is the first curated entry: the uploaded wordmark (which was an SVG wearing a .png name — kept as vector, script-sanitised) plus the full supplied bio. **Location note:** the brief said "Karnataka", but Araku's own bio and all six of its dataset coffees say Andhra Pradesh — the Karnataka chip in the original wireframe was mock data. The page derives location from the dataset (Andhra Pradesh) rather than contradict itself; a hardcoded override is one line in `brands.js` if ever wanted.
- Homepage roaster tiles and each coffee page's "More from {brand}" button now route to brand pages instead of a directory search.
- One testing lesson logged: the location chip is CSS-uppercased, so text assertions must be case-insensitive — the page was right, the probe was wrong.
