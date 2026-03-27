# Localization Audit Skills — Design Spec

## Problem

Preparing a client app for localization requires significant upfront analysis before string extraction begins. Without it, teams extract strings that are structurally broken (concatenated fragments, naive pluralization), tonally inconsistent, or terminologically messy — leading to rework, poor translations, and brand inconsistency across locales.

Today, this audit process is manual and ad hoc. There is no structured, repeatable skill that guides a Claude Code agent through a comprehensive localization readiness assessment.

## Solution

Four Claude Code skills that guide an agent through a structured localization audit of any codebase. Three analysis skills each address a distinct concern, and one orchestrator skill coordinates them and consolidates findings. All are independently invocable.

## Architecture

### Skill Set

| Skill | Role | Purpose |
|-------|------|---------|
| `auditing-i18n-readiness` | Orchestrator | Invokes missing analysis skills, consolidates recommendations |
| `auditing-i18n-string-patterns` | Analysis | Discover all strings, analyze construction patterns, check formatting, assess scope |
| `auditing-i18n-tone` | Analysis | Assess brand/tone consistency and cultural risks |
| `auditing-i18n-terminology` | Analysis | Ensure vocabulary consistency, build proto-glossary |

### Dependency Model

```
auditing-i18n-readiness (orchestrator — no analysis of its own)
       │
       ├── 1. auditing-i18n-string-patterns (if not already run)
       │      (produces tech stack + string inventory + scope metrics
       │       that tone and terminology consume)
       │
       ├── 2. In parallel (after string-patterns):
       │       ├── auditing-i18n-tone
       │       └── auditing-i18n-terminology
       │
       └── 3. Consolidate Recommended Next Steps
```

The readiness skill is the primary entry point for general i18n audit requests. It orchestrates the analysis skills, invoking any that haven't already run. Each analysis skill remains independently invocable for targeted use.

The string-patterns skill runs first because it produces the tech stack and string inventory that tone and terminology consume. If tone or terminology runs independently without this data, it performs lightweight string discovery on its own rather than failing.

### Recommended Next Steps Consolidation

The readiness skill, as the orchestrator, consolidates the "Recommended Next Steps" in its final phase — deduplicating, ordering by priority, and ensuring pre-extraction action items are clearly separated from extraction guidance. If an analysis skill runs independently (not orchestrated), it appends its own recommendations.

### Output Files

Skills write to two files in the target repo root:

**`i18n-pre-extraction-fixes.md`** — pre-extraction findings and recommendations:

```markdown
# i18n Pre-Extraction Fixes

## Tech Stack & Configuration          <!-- auditing-i18n-string-patterns -->
## Scope Assessment                    <!-- auditing-i18n-string-patterns -->
## Pre-Extraction Fixes                <!-- auditing-i18n-readiness -->
## Tone & Brand Analysis               <!-- auditing-i18n-tone -->
## Terminology Consistency             <!-- auditing-i18n-terminology -->
## Recommended Next Steps              <!-- each skill contributes -->
```

**`i18n-extraction-pattern-catalog.md`** — context for the extraction step:

```markdown
# i18n Extraction Pattern Catalog

## Summary                             <!-- auditing-i18n-readiness -->
## Pattern Entries                     <!-- one section per technique type -->
## Cross-Cutting Gotchas               <!-- auditing-i18n-readiness -->
## Translator Context Notes            <!-- auditing-i18n-readiness -->
## Recommended Next Steps              <!-- extraction guidance -->
```

### Framework Agnosticism

Skills adapt to the detected tech stack. Supported ecosystems:
- **Web:** React, Vue, Angular, Svelte, plain HTML/JS
- **iOS:** Swift, Objective-C (UIKit, SwiftUI, Storyboards)
- **Android:** Kotlin, Java (XML layouts, Compose)
- **Other:** Any codebase with identifiable UI-rendering code

Detection heuristics are stack-specific but the analysis process is universal.

### Skill Location

Personal skills in `~/.claude/skills/`:

```
~/.claude/skills/
  auditing-i18n-readiness/SKILL.md
  auditing-i18n-string-patterns/SKILL.md
  auditing-i18n-tone/SKILL.md
  auditing-i18n-terminology/SKILL.md
```

---

## Skill 1: `auditing-i18n-readiness`

### Frontmatter

```yaml
---
name: auditing-i18n-readiness
description: Use when assessing localization readiness, auditing a codebase for i18n, or preparing for string extraction — orchestrates all audit skills and consolidates findings
---
```

### Orchestrator Role

This skill performs no analysis of its own. It checks which analysis skills have already run, invokes any that are missing, and consolidates the Recommended Next Steps across all skills.

### Process

**Phase 1: Check existing state**
- Examine `i18n-pre-extraction-fixes.md` and `i18n-extraction-pattern-catalog.md` for section markers indicating which skills have run

**Phase 2: Invoke missing skills**
- `auditing-i18n-string-patterns` first (if missing) — other skills consume its output
- `auditing-i18n-string-patterns`, `auditing-i18n-tone`, `auditing-i18n-terminology` in parallel (after scope)

**Phase 3: Consolidate**
- Deduplicate Recommended Next Steps across all skills
- Order by priority, separate pre-extraction actions from extraction guidance
- Offer to create implementation plans for all actionable items
- Write consolidated section to `i18n-pre-extraction-fixes.md`

---

## Skill 2: `auditing-i18n-string-patterns`

### Frontmatter

```yaml
---
name: auditing-i18n-string-patterns
description: Use when analyzing string construction patterns, pluralization, formatting, and non-code content for i18n — catalogs what the extraction step must handle and identifies formatting fixes to do beforehand
---
```

### Dual-Output Model

This skill categorizes findings using a framework that separates pre-extraction fixes from extraction patterns:

- **Pre-Extraction Fixes** → `i18n-pre-extraction-fixes.md`: Formatting issues (dates, numbers, currency) that are library-agnostic and should be centralized before extraction.
- **Extraction Pattern Catalog** → `i18n-extraction-pattern-catalog.md`: String construction techniques (concatenation, interpolation, ternary switching, pluralization) where the fix IS the extraction. Documented with locations, conversion recipes, and gotchas.

### Process

**Phase 1: Load context**
- Read scope output from `i18n-pre-extraction-fixes.md` (tech stack, string inventory)
- If scope hasn't run, perform lightweight string discovery

**Phase 2: Analyze string construction** → *Extraction Pattern Catalog*
- Concatenation, template literals, ternary text switching, fragment assembly
- Each pattern recorded with technique, location, conversion recipe, and gotchas
- Exception: genuinely tangled logic (multi-line assembly, cross-function fragments) → Pre-Extraction Fix

**Phase 3: Detect pluralization patterns** → *Extraction Pattern Catalog*
- Ternary plurals, custom utilities (`pluralize()`), verb conjugation
- All cataloged with ICU conversion recipes (no value in standardizing English-only pluralization pre-extraction)

**Phase 4: Check formatting patterns** → *Pre-Extraction Fixes*
- Hardcoded date formats, locale args, currency symbols, number formatting
- These are library-agnostic code quality fixes — centralize behind `Intl.*` APIs before extraction

**Phase 5: Scan non-code content** → *Extraction Pattern Catalog*
- Images/SVGs with text, CSS `content:`, a11y attributes, HTML attributes, confirmation dialogs
- Extraction targets the agent needs to locate and handle

**Phase 6: Assess translator context** → *Both*
- Ambiguous terms and product names → pre-extraction glossary work (feed into terminology skill)
- Translator descriptions for extracted messages → extraction guidance

**Phase 7: Write reports**
- Write "Pre-Extraction Fixes" section to `i18n-pre-extraction-fixes.md`
- Write full pattern catalog to `i18n-extraction-pattern-catalog.md`
- Contribute to Recommended Next Steps in both files

---

## Skill 3: `auditing-i18n-tone`


### Frontmatter

```yaml
---
name: auditing-i18n-tone
description: Use when auditing copy for brand and tone consistency before localization — identifies voice deviations, cultural risks, and style guide mismatches across user-facing strings
---
```

### Process

**Phase 1: Load context**
- Read scope output for string inventory
- Search for brand/style guide documents in repo: `STYLE_GUIDE.md`, `brand-guidelines.*`, `content-guide.*`, docs directories, design system docs

**Phase 2: Establish baseline tone profile**
- Sample representative cross-section of strings (across features, string types)
- Characterize dominant tone along dimensions:
  - **Formality:** casual ↔ formal
  - **Warmth:** friendly/personal ↔ neutral/institutional
  - **Directness:** concise/imperative ↔ verbose/explanatory
  - **Technical level:** plain language ↔ jargon-heavy
- Present as the "de facto voice" of the app

**Phase 3: Detect tone deviations**
- Error messages that are suddenly harsh or overly technical
- Marketing-style language in functional UI
- Inconsistent formality levels across screens (casual onboarding vs. formal settings)
- Passive voice in a codebase that's otherwise direct
- Overly casual copy in serious contexts (financial, medical, legal)

**Phase 4: Brand guideline comparison (conditional)**
- If brand/style guide documents are found: compare de facto voice against documented guidelines
- Flag systematic mismatches (e.g., guidelines say "friendly and casual" but error messages are "formal and terse")
- If no guidelines found: note this as a recommendation (create guidelines before localizing)

**Phase 5: Identify localization risks from tone**
- Humor, idioms, colloquialisms (often don't translate well or may be offensive)
- Culture-specific references (sports metaphors, holidays, social norms)
- Emotional language that may land differently across cultures
- Slang or informal contractions

**Phase 6: Write to report**
- Append "Tone & Brand Analysis" section with:
  - Baseline tone profile summary
  - Deviations table: string, location, expected tone, actual tone, severity
  - Localization risk items with specific strings flagged
  - Recommendations for standardization before extraction
- Contribute items to "Recommended Next Steps"

---

## Skill 4: `auditing-i18n-terminology`

### Frontmatter

```yaml
---
name: auditing-i18n-terminology
description: Use when auditing vocabulary consistency before localization — finds synonyms used for the same concept, ambiguous terms, and builds a proto-glossary for translators
---
```

### Process

**Phase 1: Load context**
- Read scope output for string inventory and tech stack context

**Phase 2: Extract key terms**
- **Action labels:** button text, menu items, link text ("Save", "Submit", "Delete", "Cancel")
- **Navigation labels:** tab names, section headers, breadcrumbs
- **Status/state words:** "Loading", "Error", "Success", "Pending", "Active", "Disabled"
- **Domain-specific nouns:** what the app calls its core entities (e.g., "project" vs. "workspace" vs. "space")

**Phase 3: Group by semantic intent**
- Cluster terms that convey the same meaning
- Example: "Delete" / "Remove" / "Trash" / "Discard" for the same destructive action
- Example: "Save" / "Submit" / "Apply" / "Confirm" for completion actions
- Present each cluster with the contexts where each variant appears

**Phase 4: Detect inconsistencies**
- **Same concept, different words:** "Settings" in one place, "Preferences" in another
- **Same word, different meanings:** "Post" as a noun (blog post) and verb (submit) — critical for translator context
- **Unnecessary variation:** "Sign in" vs "Log in" vs "Login" across the app
- **Contextual mismatches:** using "Delete" for soft-delete and hard-delete without distinction

**Phase 5: Build proto-glossary**
- For each semantic cluster, recommend a canonical term:
  - Most frequently used term (momentum/consistency), OR
  - Most precise/clear term (if the frequent one is ambiguous)
- Include translator context notes: what does this term mean in this app?
- Flag terms that will need special handling in translation (polysemous words, technical terms)

**Phase 6: Write to report**
- Append "Terminology Consistency" section with:
  - Inconsistency table: concept, variants found, locations, recommended canonical term
  - Proto-glossary for localization team
  - Ambiguous terms requiring translator context notes
- Contribute items to "Recommended Next Steps"

---

## Success Criteria

1. **Skill loading:** Each skill appears in Claude Code's skill list and loads when relevant triggers match the description
2. **Independent execution:** Each skill runs standalone; downstream skills perform lightweight discovery if scope hasn't run
3. **Sequential execution:** Running scope → readiness → tone → terminology produces a coherent, non-duplicative report
4. **Framework adaptation:** Skills adapt detection heuristics to the detected tech stack
5. **Report quality:** Output is structured, actionable, includes severity ratings and next-step recommendations
6. **Actionable findings:** Every issue in the report includes enough context (location, example, severity) for a developer to act on it

## Non-Goals (v1)

- Automated string extraction (separate future skill)
- RTL layout analysis (requires runtime/visual analysis)
- Text expansion simulation (requires design tooling)
- Gender/grammatical agreement analysis (requires linguistic knowledge beyond static analysis)
- Integration with translation management systems
- Automated fixes or refactoring
