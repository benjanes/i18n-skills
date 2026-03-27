---
name: localize
description: "Extract hardcoded user-facing strings from source files and replace them with localization keys pointing to a JSON catalog. Use this skill whenever the user wants to internationalize, localize, or i18n their code, make a repo translation-ready, extract strings for localization, replace hardcoded text with translation keys, or set up i18next/react-intl/NSLocalizedString/gettext. Trigger on phrases like 'localize this', 'make this translatable', 'extract strings', 'i18n', 'internationalize', 'add translations support', or 'localization ready'. Also trigger when users mention specific files or components they want to prepare for multiple languages."
---

# Localize

Replace hardcoded user-facing strings in source files with keys that reference a JSON catalog using ICU MessageFormat syntax.

## How it works

1. **Analyze** — ensure string patterns have been cataloged (run auditing-i18n-string-patterns if needed)
2. **Generate hierarchical keys** based on the file path and component/function context
3. **Replace** each string in the source code with the appropriate translation function call
4. **Write** the extracted strings into a JSON catalog file

## Prerequisite: String pattern analysis

Before extraction, the codebase's strings must be inventoried and their construction patterns cataloged. Check for `i18n-extraction-pattern-catalog.md` in the project root. If it doesn't exist, you must run auditing-i18n-string-patterns on the specified path(s), specified domain/view or the entire project if not specificed. Do not analyze on your own, you must use the catalogs.

The pattern catalog (`i18n-extraction-pattern-catalog.md`) provides:
- What string construction patterns exist (template literals, ternaries, plurals, select switches, fragment assembly)
- How to convert each pattern to ICU MessageFormat
- Gotchas and edge cases per pattern type
- Cross-cutting concerns (ICU escaping, wiring interpolation values through)

The pre-extraction report (`i18n-pre-extraction-fixes.md`) provides:
- Tech stack and frameworks in use
- Scope metrics (how many strings, where, what's already localized)
- String density heatmap (which files have the most strings)

## Key naming convention

Use **hierarchical dot-notation** keys derived from the file path and semantic context:

```
<domain>.<component>.<element>
```

**Examples:**
- `settings.profile.title` — title in the profile section of settings
- `auth.login.submitButton` — submit button on the login form
- `errors.network.timeout` — network timeout error message
- `dashboard.widgets.salesChart.title` — title of the sales chart widget

**Rules:**
- Derive the domain from the file's directory path (e.g., `src/components/settings/` → `settings`)
- Use the component or function name as the middle segment
- Use a descriptive suffix for the specific element (`title`, `description`, `placeholder`, `errorMessage`, `label`, `tooltip`, `altText`)
- Use camelCase for each segment
- Keep keys concise but unambiguous — if two components could collide, add more specificity
- If the repo already has translation keys, match the existing naming convention instead

### Monorepo key naming

In a monorepo with multiple apps or packages, use a two-tier namespace strategy:

1. **Common strings** — generic UI strings that appear across multiple apps (e.g., "Save", "Cancel", "OK", "Delete", "Loading...") go under `common.*`:
   ```
   common.save        → "Save"
   common.cancel      → "Cancel"
   common.delete      → "Delete"
   common.loading     → "Loading..."
   common.errors.notFound → "Not found"
   ```

2. **App-specific strings** — strings unique to a particular app get prefixed with the app name:
   ```
   web.auth.login.title       → "Sign In"
   mobile.profile.editButton  → "Edit Profile"
   api.errors.rateLimited     → "Too many requests"
   ```

**Deduplication rule:** If the exact same English string appears in multiple apps, use one shared key. For common UI verbs and phrases, always check `common.*` first before creating an app-specific key. This keeps the catalog lean and ensures consistent translations across platforms.

**Detecting app names:** To determine what to use as the app prefix:
1. Read the app's config file (`package.json` name field, `build.gradle` app name, Podfile target name, etc.)
2. If no config file gives a clear name, fall back to the directory name (e.g., `packages/web` → `web`, `apps/mobile` → `mobile`)

## Catalog file

Use a **flat JSON** file with dot-notation keys and **ICU MessageFormat** values:

```json
{
  "settings.profile.title": "Profile Settings",
  "settings.profile.description": "Manage your account preferences",
  "auth.login.submitButton": "Sign In",
  "auth.login.placeholder.email": "Enter your email",
  "dashboard.welcomeMessage": "Welcome back, {name}!",
  "cart.itemCount": "{count, plural, one {# item} other {# items}}",
  "profile.greeting": "{gender, select, male {He updated his profile} female {She updated her profile} other {They updated their profile}}"
}
```

### ICU MessageFormat syntax

ICU MessageFormat is the industry standard for translatable strings. It handles interpolation, plurals, and gender/select in a single string — no need for separate key suffixes or special conventions.

**Simple interpolation:** Use `{variableName}` for variable placeholders:
```
"Welcome back, {name}!"
```

**Plurals:** Use `{variable, plural, ...}` when a string varies by count. The `#` symbol is replaced by the count value:
```
"{count, plural, one {# item in your cart} other {# items in your cart}}"
```

**Select (gender, status, etc.):** Use `{variable, select, ...}` when a string varies by a category:
```
"{role, select, admin {Administrator} editor {Editor} other {Viewer}}"
```

**Nested:** ICU supports nesting these constructs:
```
"{gender, select, male {He has {count, plural, one {# notification} other {# notifications}}} female {She has {count, plural, one {# notification} other {# notifications}}} other {They have {count, plural, one {# notification} other {# notifications}}}}"
```

**Catalog location:**
1. If a catalog file already exists in the repo (look for files like `en.json`, `locale/en.json`, `src/locales/en.json`, `public/locales/en/translation.json`, `i18n/en.json`), use it and append new keys
2. If no catalog exists, create `locales/en.json` at the project root
3. When appending to an existing catalog, preserve all existing keys — never remove or rename keys that are already there
4. Sort keys alphabetically in the output for readability

**Monorepo catalog:** In a monorepo, there is **one shared catalog** for the entire repo — not one per app. Look for an existing catalog in these locations (in order):
1. A dedicated i18n package (e.g., `packages/i18n/locales/en.json`, `packages/shared/i18n/en.json`)
2. The repo root (`locales/en.json`, `i18n/en.json`)
3. If no catalog exists, ask the user where they'd like it. If they don't have a preference, create `locales/en.json` at the repo root

## Framework-specific replacement patterns

The way you replace strings depends on the language and framework. Read the appropriate reference file before making changes:

| Environment | Reference file | When to read it |
|---|---|---|
| React / JS / TS | `references/react-js-ts.md` | `.jsx`, `.tsx`, `.js`, `.ts` files in a React/Next.js/Remix project |
| iOS (Swift) | `references/swift.md` | `.swift` files |
| Android (Kotlin) | `references/kotlin-android.md` | `.kt` files in an Android project |
| Python | `references/python.md` | `.py` files |
| General / Other | `references/general.md` | Anything not covered above |

Use the tech stack from the pre-extraction report to select the right reference file. Then read only the relevant one.

## Workflow

1. **Check for pattern catalog** — if `i18n-extraction-pattern-catalog.md` doesn't exist, run auditing-i18n-string-patterns on the target path(s) (or the entire repo if no paths were specified)
2. **Read the reports** — read both `i18n-extraction-pattern-catalog.md` and `i18n-pre-extraction-fixes.md` for tech stack, scope metrics, string inventory, and pattern conversion recipes
3. **Check for blockers** — scan the pre-extraction report for any findings marked **Blocker** severity. If blockers exist, list them for the user and strongly recommend fixing them before proceeding with extraction — these are issues (hardcoded locales, hardcoded currency symbols, etc.) that will produce incorrect results for non-English users even after strings are extracted. Do not refuse to proceed, but make the risk clear and get acknowledgment before continuing.
4. **Determine scope** — if the user pointed at specific files or directories, localize just those. If they said something broad like "localize the repo", use the full string inventory from the reports
5. **Confirm scope with the user** — use the `AskUserQuestion` tool to present a scope assessment from the reports and get confirmation before making any code changes. The question should:
   - State the total number of strings found and where they live (e.g., "Found 47 user-facing strings across 12 files in the settings page")
   - Break down what will be covered into concrete categories (e.g., "23 strings in JSX render methods, 18 strings in preference data definitions, 6 strings in custom modal components")
   - If some areas require more involved changes (like threading intl through a component that doesn't have it, or refactoring data definitions to support dynamic values), call that out explicitly so the user knows
   - Offer options like:
     1. "Localize everything listed above" (default — full coverage of the scope)
     2. "Let me choose which categories to include"

   **Do NOT silently skip parts of the scope.** The goal is comprehensive i18n of whatever the user asked for. If you'd have to leave some strings untouched, the user must know that upfront and agree to it — never surprise them after the fact with "here's what we didn't do."

6. **Read the appropriate reference file(s)** from the table above — use the tech stack from the pre-extraction report to select the right one
7. **Check for existing localization setup** — look for existing catalog files, i18n config, or translation imports. If the project already uses a localization library, use its patterns
8. **Generate keys** using the hierarchical naming convention (with app prefixes in a monorepo — see the monorepo key naming section)
9. **Check for duplicates** — before creating a new key, check if the same English string already exists in the catalog. If so, reuse that key. Pay special attention to common UI strings that belong under `common.*`
10. **Replace strings** in source files with the framework-appropriate function call — use the pattern catalog's conversion recipes for template literals, ternaries, plurals, select patterns, and other non-trivial constructions. This includes all string locations confirmed in step 5 — rendering layer, data definitions, custom components, and any intl threading needed
11. **Update the catalog** — append new key-value pairs to the single JSON catalog (create it if it doesn't exist)
12. **Add necessary imports** — if the replacement pattern requires an import (e.g., `useTranslation` hook), add it to the file
13. **Report** what was done: how many strings extracted, how many were deduplicated, which files modified, where the catalog was written. Confirm that all categories from the scope assessment in step 5 were covered. In a monorepo, break down the report by app

## Important considerations

- **Context**: If the same English word means different things in different places (e.g., "Post" as a noun vs verb), use distinct keys to allow translators to provide different translations.
- **JSX**: In JSX, strings inside `{}` expressions AND bare text children both need extraction. The `<Trans>` component handles JSX with embedded elements — read the React reference for details.
- **Don't break logic**: If a string is used in a comparison (e.g., `if (status === "active")`), that's a logic string, not a user-facing one. Leave it alone even if it looks like English text.
- **Escaping in ICU**: Literal curly braces in text must be escaped with single quotes: `"Use '{' and '}' for objects"`. A literal single quote is `''`.
- **Wire interpolation values through**: Every `{placeholder}` in the catalog must have a corresponding values argument at the call site. Without it, the placeholder renders as literal text (e.g., `{name}` instead of "Alice"). This is especially easy to miss when i18n keys are stored as data (config objects, arrays, preference definitions) rather than called inline — the data structure needs a field for interpolation values, and the renderer must forward them. Before finishing, search the catalog for all `{...}` placeholders and verify each one has values wired through the rendering path. See the framework-specific reference files for detailed examples.
