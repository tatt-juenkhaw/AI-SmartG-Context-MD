# AI SmartG — Enhanced AI Training Workflow: Consolidated Design Reference

**Objective:** From RGB images and 3D heatmap alone, automatically assign accurate inspection criteria to every part on a board, with minimal user input, and give the user a guided path to review and fine-tune the result until production-ready. Two phases: **Auto Train** (generates the parameter database from scratch) and **Device Review** (validates and fine-tunes it).

Reviewed in a design session on 8/9. Feedback from that session is marked **[Review 9/9]**. Anything still unresolved is marked **[Open]**, and repeated in the Open Items list at the end.

---

## Design Consideration

- Seamless integration of modules in the programming experience — a smooth pipeline between segregated modules (AI Param Profile, Auto Train, Recipe QC, Negative Test, Advanced Parameter…).
- Insightful clues to lead the user through completing review and fine-tuning.
- Data-driven presentation to build user confidence in AI Setup accuracy.
- Keep each module purpose-guided in the presentation layer; keep the pipeline generic in the backend layer.
- Tailor-made algorithms (Coplan, etc.) stay available on the Editor — an extra parsing layer makes them compatible with the generic algorithm (3D Box) pipeline underneath, so nothing validated is thrown away.

## New Device Concept

**1. Single generic algorithm container (U-Type)** retires the customized pipeline per CBRS category (Capacitor / Bright Box / Resistor / SOIC — each letter a tailored category). U-Type is modular rather than category-tailored: broader coverage, higher learning curve, since nothing is pre-shaped to a specific use case.

Worked example — Coplanarity, before/after:
| | CBRS (today) | U-Type (proposed) |
|---|---|---|
| User-facing screen | "Coplanarity Setup" | Unchanged |
| Backend algorithm | Custom Coplan pipeline per category | Generic 3D Box-to-Box comparison |
| What the user relearns | — | Nothing |
| What engineering maintains | A separate pipeline per category, per criterion | One pipeline, reused across every package |

**2. "Package" as the primary classification tier**, replacing device-type-first classification — improves syncing of algorithm coverage, threshold standard, and data analysis across a package, and scales better than recognizing a device type's first letter (the CBRS approach).

## Threshold Parameter Concept

Prioritize **percent over micron**: shared unit across modules/algorithms eases syncing; quantifiable in data analysis for reference values; resilient to geometry/height changes (saves re-tuning time). Micron stays tunable — every threshold pairs with both units, synced.

Worked example — Lifted Lead, two parts in the same package:
- Micron rule ("Lift > 50µm = defect", tuned for Part A at 0.5mm lead height) stays 50µm for Part B (1.2mm lead height) — now only ~4% deviation, silently under/over-flagging. Someone has to notice and manually re-derive.
- Percent rule ("Lift > 10% of lead height") gives 50µm for Part A and 120µm for Part B automatically — same rule, same severity, no re-tuning.

**[Review 9/9]** Percent-over-micron is only justified for parameters that reference part size/height. **Locator Offset and Lifted Lead qualify — Coplanarity does not.**

**Resolved:** the three illustrations that originally used Coplanarity as the percent-primary worked example (AI Setup median example, Fine-Tune UI, Drift & Apply) now use **Locator Offset** instead, referenced against **body size** (not body height, which was Coplan's reference dimension and doesn't apply here). Coplanarity remains a real criterion in the coverage matrix — it's just no longer the worked example for this pattern.

## End-to-End Use Case Overview

A new program starts with an empty database.

**Auto Train:** user triggers AI Setup → defines algorithm coverage and preset parameters by package → AI locates Body/Lead/Joint and Height → AI Param Config presets threshold parameters → performs Negative Test with AI Synthetic Defect Images *(deprioritized, see below)* → performs Abnormalities Analysis on AI inference data → flags Auto Train status for review.

**Device Review:** user reviews AI Setup → identifies package type / locates unrecognized parts manually as a template for AI → triggers AI Param Config to preset threshold parameters → fine-tunes flagged threshold parameters with reference values from analysis → proliferates the fine-tuned standard to package level for reuse.

**Artifacts exchanged:** Auto Train hands off user-defined inspection criteria/standard by package, AI segmentation and inference data by part/package, and a trained database with AI flags. Device Review hands back a fine-tuned inspection standard by package, an AI template for unrecognized parts, and a fine-tuned database — looping back into Auto Train for reuse, and forward into a production-ready program.

**[Review 9/9]** Negative Test with Synthetic Images is not being pursued now — parallelism with Auto Train isn't achievable, and it's a programming-time concern. Treat as optional/future wherever it appears below.

---

# AUTO TRAIN

## Step 1 — Algo Coverage + AI Param Config

One-stop UI for algorithm coverage checking and threshold presets by package. Before Auto Train runs, the user does a quick coverage confirmation and, if needed, checks preset parameters. Multiple preset profiles are possible.

**Illustration — coverage matrix:** rows are package types (SOIC, Capacitor, Resistor, MELF, BGA), columns are six criteria (Body Locator, Lead Locator, Coplanarity, Damaged, Solder Coverage, Lifted Lead). Each cell is one of three states: covered (clickable, opens detail), not applicable to that package, or roadmap/pending. Coverage differs meaningfully by package — Capacitor/Resistor (leadless chip parts) lack Lead Locator, Coplanarity, and Lifted Lead; MELF gets Coplanarity (end-cap flatness) but not discrete lead checks; BGA's Solder Coverage is pending since ball-based inspection needs a different metric than area coverage. SOIC's Body Locator cell is the highlighted entry point into the detail screen below.

**Illustration — Locator Offset preset detail (SOIC):** reached from the highlighted cell. Fields: **X Offset** toggle + **X Offset Tolerance (% of body size)** slider; **Y Offset** toggle + **Y Offset Tolerance (% of body size)** slider; **Skew** toggle + **Skew Tolerance (°)** slider (rotational, so degrees rather than percent — it doesn't scale with body size the way a positional offset does); an **IPC Class / Custom** selector (IPC-A-610 Class 1/2/3, or Custom); an Apply-to-package button. A reference panel alongside shows, for each of the three tolerances, a shaded band (peer min–max range) with a median marker, plus the numeric range/median/sample count — e.g. X Offset: peer median 7.5%, range 5–11%, across 12 refdes over 3 validated SOIC PNs.

## Step 2 — AI Setup & Median Aggregation

AI segmentation runs on every refdes (sample) of a part independently; the final ROI and every downstream measurement is the **median** across all of them — one mis-seated instance can't skew the whole part's profile.

**[Review 9/9]** All AI inference, including the Classifier itself, adopts the median value — not just geometric measurements. Speed is a primary concern, since this means running AI models on every component, not a sample.

**Illustration — locating across samples:** three schematic SOIC instances (refdes U1, U4, U7), each with color-coded ROI overlays: Body (teal), Lead (orange), Solder (green). Below them, range bars for Body Size, Lead Height, and Solder Coverage — each a shaded min–max band with a median marker, e.g. Body Size 5180–5220µm (median 5200µm), Lead Height 440–460µm (median 450µm), Solder Coverage 89–93% (median 91%).

**Illustration — preset % + median → effective threshold:** three small cards. **Locator Offset** (preset 8%, median body size 5200µm, badge "Auto-scaled") and **Lifted Lead** (preset 10%, median lead height 450µm, "Auto-scaled") are *scaling* thresholds — the percent is what's stored; the µm figure is shown only for confidence, not cross-converted back into a stored value. **Solder Coverage** (preset minimum 75%, median measured 91%, badge "Pass") works differently — the percent *is* the pass bar, and the AI's job is to confirm the part's actual measured performance clears it, not to rescale anything.

## Step 3 — Setup Abnormalities Analysis

Detects abnormally high variance in AI inference results within the same part, filtering outlier data from genuinely defective units — giving the user a concrete, data-driven clue about which device and which specific parameter needs attention.

**[Review 9/9]** Aside from AI/3D abnormal performance (high *variance*), a high *median* (not variance) on Locator Offset can indicate a **CAD offset** — a systematic shift, not sample noise. **[Open]** Not yet represented as a category below — the current design only covers variance-based flags; a "systematic offset" category would need adding alongside them.

**Illustration — five flagged parameters, grouped by root cause, not treated as one undifferentiated pile:**
- **AI detection ambiguity** (teal) — *Count of Lead* (e.g. 8 leads detected in 10 of 12 refdes, 7 in the other 2 — one specific pin is ambiguous) and *Body Size* (range 4850–5580µm). User verifies what the AI actually saw.
- **Spec mismatch** (orange) — *Lead Height* (range 380–520µm) and *Body Height* (range 960–1310µm). User checks the datasheet — the part's true dimension may differ from assumption.
- **Process judgment** (violet) — *Solder Size* (range 64–96%, median 85%). Nothing is technically wrong; the user decides whether the generic preset actually fits this production line's real tolerance.

Every row also gets a uniform "Flagged" badge regardless of category — category color signals *why*, the badge signals *that* it needs attention. The routing card (second half of this illustration) maps each category to its specific required action in one line, so the flag isn't just a warning but an instruction.

## Step 4 — Classify Auto Train Outcome + Outcome Report

Three outcomes from a four-stage pipeline: **AI Classify → AI Locate ROI → Param Preset → Abnormality Analysis.**
- Clean pass through all four → **Complete**.
- Passes Classify/Locate/Preset but gets flagged at Abnormality Analysis → **Complete with Risk**.
- Fails to classify ("Unknown class") or fails to locate ("can't find body/lead") → **Failed — Manual Setup Required**. Both failure points converge on the same outcome regardless of which stage caused it. Param Preset has no independent failure branch in this model — it only runs once Classify and Locate ROI have already succeeded — worth confirming that's actually true of the real pipeline.

- Status is backed by data analysis, giving the user confidence in what they're looking at.
- Abnormalities are pinpointed with a concrete action item, not a bare flag.
- The report doubles as a checklist toward production-ready.

**Illustration — results table by part:** columns are PN, refdes count, detected package type (what the classifier actually found, not the PN's expected package — hence "Unknown" for a failed-classify row), status, and issue. Sample rows span all three outcomes and reuse the exact category colors from Step 3 for "Complete with Risk" issues (e.g. a Body Height issue shows an orange dot, matching "spec mismatch"), plus a distinct red dot for "Failed" rows — so a reviewer pattern-matches severity/type by color across both screens without re-reading text.

---

# DEVICE REVIEW

## Step 1 — Starting with Unrecognized Parts

User is guided to the serious setup failures first, proceeding directly to the Editor page with the Auto Train checklist as the primary guide.

Interaction sequence: guide user to the serious fault first → user manually identifies package type / locates the missing ROI with minimal effort (one rough box, one dropdown — not per-lead detail) → trigger Auto Setup again for that part, re-run abnormality analysis, update the outcome status → few-shot learning lets the AI learn from the user-defined template so the next occurrence of an unrecognized part like this one needs no manual input at all.

**[Review 9/9]** Few-shot learning applies program-wide, not just to future occurrences in one session — and it improves the Classifier itself, not only ROI location. Centralizing user-defined templates (potentially across programs) is a future idea, not yet designed.

**Illustration — four-panel User/UI/AI Module interaction, using a SOIC example:** each panel tags which of the three actors (User, UI, AI Module) is driving it, with inactive actors dimmed.
1. *UI-driven.* A triage worklist surfaces the Failed part at top, ahead of Risk/Complete parts shown dimmed below.
2. *User-driven.* Visual: a vague, dashed "unknown" blob with a "?" — deliberately not a mis-shaped SOIC, since the AI genuinely has nothing to seed from — transitions to the same shape with one dashed box drawn around the approximate body, plus a package-type dropdown pick.
3. *AI-driven.* The seeded ROI re-enters the exact same pipeline as any other part (AI Locate ROI → Param Preset → Abnormality Analysis), resolving to the fully segmented SOIC (same Body/Lead/Solder ROI colors used elsewhere) and a Complete badge.
4. *AI-driven.* Same blob silhouette, but now transitioning straight to full segmentation with no user icon active at all — the payoff of few-shot learning: the next occurrence costs the user nothing.

## Step 2 — Fine Tuning Threshold

User stays in charge of the final value, but with data support from two sources: **Visit All** (range/median from inspecting all samples of this specific part) and **Same Package** (range/median across other parts sharing the package, with the preset/golden-standard value marked distinctly).

**[Review 9/9]** These two sources are *prioritized*, not equally weighted: **Visit All** takes precedence once there's an abundant sample of the same part (i.e. programming on a good board). **Same Package Standard** is the reference for a recipe that's freshly generated and hasn't been through Visit All or fine-tuning yet. Centralized production data across sites would make either reference more accurate. **[Open]** The current illustration presents both as equal-weight parallel rows — the prioritization logic above isn't yet visually encoded (e.g. dimming or reordering whichever source is less authoritative at a given point in the part's lifecycle).

**Illustration — Locator Offset fine-tune, worked on part SOIC8-3390:** a large percent readout drives a slider (the primary control); a small synced micron textbox sits beside it, computed from this part's measured body size (5200µm) — typing into either updates the other live. Below, two reference rulers share one 0–16% scale so everything lines up vertically: **Visit All Result** shows this part's own measured range (6–13%, median 9%) as a band with a median tick; **Same Package** shows peer parts' configured values (5–11%, median 7.5%) plus a distinct triangle marker for the golden standard (8%) — deliberately a different marker style from the median tick, since a policy default and an empirical peer statistic are conceptually different numbers even when close. A synthesized "suggested range" band (8–9.5%) sits behind all rows, and a thin current-value marker moves live across both reference rows as the slider is dragged, so the user can see where their setting falls against both distributions while adjusting it.

## Step 3 — Adapt as New Package Standard

With abundant fine-tuning data, the UI can flag that the golden standard is drifting away from the median of actual values in use. The user decides whether to adopt the current median as the new golden standard for the whole package — a closed loop where the user's own fine-tuning improves Auto Param Setup accuracy over time.

**Illustration — drift chart:** five SOIC parts, fine-tuned in chronological order, plotted against a flat 8% golden-standard line: 9.0% → 9.3% → 9.8% → 10.4% → 11.0% (current). Each value looks reasonable in isolation; the trend across all five is what's actually informative — the golden standard hasn't moved while the fine-tuned values have climbed steadily away from it.

**Illustration — extended fine-tune UI with Apply Changes:** same slider/ruler layout as Step 2, now showing the drifted value (11.0%, 572µm at 5200µm body size) with a banner noting this is the 5th SOIC part in a row above standard. Two independent checkboxes, deliberately not one combined action:
- **"Apply 11.0% to all parts in SOIC package"** (orange, scoped to *existing parts*) — retroactive; impact preview: 9 validated parts, 108 refdes, 2 currently-Risk parts would move to Complete.
- **"Set 11.0% as new golden standard for SOIC"** (teal, scoped to *future parts*) — only changes what new SOIC parts start from at Auto Train time; existing parts untouched unless the option above is also checked.

Both default to checked in the current mockup — worth reconsidering, since defaulting to *unchecked* would better match the "confirm impact before applying" principle used elsewhere in this design; as built, the user opts out of a primed action rather than opting into it.

---

## Open Items / Follow-ups

- **CAD-offset detection**: high *median* (not variance) on Locator Offset signals a systematic CAD offset — a fourth abnormality category, distinct from the three variance-based ones currently illustrated. Not yet designed.
- **Fine-Tune reference prioritization**: Visit All should visually outrank Package Standard once available; currently shown as equal-weight.
- **"Direct to Editor page" navigation**: confirm this matches the real app's actual page structure — illustrated here as a generic concept only.
- **Apply Changes default state**: both checkboxes default to checked; consider defaulting to unchecked.
- **Param Preset's failure independence**: the outcome pipeline assumes Param Preset can't fail on its own (only Classify/Locate ROI can) — confirm against the real pipeline.
- **Negative Test with Synthetic Images**: deprioritized (parallelism/programming-time concern) — treat as future scope, not near-term, wherever it's referenced above.
- **Few-shot learning / centralized templates**: scope is program-wide and includes the Classifier; a cross-program centralized template store is a future idea, not yet designed.
