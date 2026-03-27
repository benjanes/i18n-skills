---
name: localize
description: "Extract hardcoded user-facing strings from source files and replace them with localization keys pointing to a JSON catalog. Use this skill whenever the user wants to internationalize, localize, or i18n their code, make a repo translation-ready, extract strings for localization, replace hardcoded text with translation keys, or set up i18next/react-intl/NSLocalizedString/gettext. Trigger on phrases like 'localize this', 'make this translatable', 'extract strings', 'i18n', 'internationalize', 'add translations support', or 'localization ready'. Also trigger when users mention specific files or components they want to prepare for multiple languages."
---

# Localize

Replace hardcoded user-facing strings in source files with keys that reference a JSON catalog using ICU MessageFormat syntax.

## How it works

1. **Scan** the target file(s) for user-facing strings (UI text, labels, messages, tooltips, error messages shown to users, placeholder text, aria-labels, alt text)
2. **Generate hierarchical keys** based on the file path and component/function context
3. **Replace** each string in the source code with the appropriate localization function call
4. **Write** the extracted strings into a JSON catalog file

## What counts as a user-facing string

Extract these:
- Text rendered in UI (labels, headings, button text, descriptions, placeholder text)
- User-visible error and success messages
- Tooltip and aria-label content
- Alt text for images

Leave these alone:
- Log messages, debug strings, comments
- CSS class names, HTML attributes that aren't user-visible
- URLs, file paths, regex patterns
- Environment variable names, config keys
- Identifiers, enum values, constants used for logic
- Strings used purely for internal comparisons or routing

When in doubt about whether a string is user-facing, err on the side of extracting it — it's easier to remove an unnecessary key than to find a missed string later.

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
- If the repo already has localization keys, match the existing naming convention instead

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

**Auto-detection rules:** When scanning source code, automatically convert these patterns to ICU:
- Ternary or if/else based on a count (e.g., `count === 1 ? "1 item" : \`${count} items\``) → `{count, plural, one {# item} other {# items}}`
- Template literals / f-strings with variables (e.g., `\`Hello, ${name}\``) → `Hello, {name}!`
- Conditional rendering based on a category (e.g., gender, role, status switches) → `{variable, select, ...}`
- A string with a `{count}` / `{number}` / numeric placeholder adjacent to a noun (e.g., `"Archive {count} chats"`) → `{count, plural, one {Archive # chat} other {Archive # chats}}` — even if the source code doesn't branch on the value. Any noun next to a count variable is inherently plural-sensitive.

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

Before starting, identify the project's framework from `package.json`, `Podfile`, `build.gradle`, `requirements.txt`, or similar config files. Then read only the relevant reference file.

## Workflow

1. **Detect repo structure** — check if this is a monorepo (look for workspace config in root `package.json`, `pnpm-workspace.yaml`, `lerna.json`, or multiple app directories like `apps/`, `packages/`, `services/`). If it's a monorepo, identify the apps and their frameworks
2. **Determine scope** — if the user pointed at specific files or directories, localize just those. If they said something broad like "localize the repo", scan all apps
3. **Scan and assess scope completeness** — before making any code changes, do a full inventory of the target scope. Identify every location where user-facing strings exist, including:
   - Strings in the rendering/template layer (JSX, HTML templates, view functions)
   - Strings in module-scope data definitions (config objects, constant arrays, preference definitions, menu items, route labels)
   - Strings in individual custom components that aren't part of a shared rendering pipeline
   - Strings that require threading intl/translation context through components or functions that don't currently have it
   - Existing catalog keys that are defined but not yet wired up in the code

   Categorize what you find into a clear breakdown. If there are strings that will require non-trivial structural changes (e.g., threading intl context through a component tree, refactoring data definitions to support dynamic values), identify those explicitly.

4. **Confirm scope with the user** — use the `AskUserQuestion` tool to present the scope assessment and get confirmation before proceeding. The question should:
   - State the total number of strings found and where they live (e.g., "Found 47 user-facing strings across 12 files in the settings page")
   - Break down what will be covered into concrete categories (e.g., "23 strings in JSX render methods, 18 strings in preference data definitions, 6 strings in custom modal components")
   - If some areas require more involved changes (like threading intl through a component that doesn't have it), call that out explicitly so the user knows
   - Offer options like:
     1. "Localize everything listed above" (default — full coverage of the scope)
     2. "Let me choose which categories to include"

   **Do NOT silently skip parts of the scope.** The goal is comprehensive i18n of whatever the user asked for. If you'd have to leave some strings untouched, the user must know that upfront and agree to it — never surprise them after the fact with "here's what we didn't do."

5. **Identify the framework(s)** — check project config files to determine the tech stack. In a monorepo, each app may use a different framework — read the appropriate reference file for each
6. **Read the appropriate reference file(s)** from the table above
7. **Check for existing localization setup** — look for existing catalog files, i18n config, or translation imports already in the project. If the project already uses a localization library, use its patterns
8. **Scan target files** for user-facing strings
9. **Generate keys** using the hierarchical naming convention (with app prefixes in a monorepo — see the monorepo key naming section)
10. **Check for duplicates** — before creating a new key, check if the same English string already exists in the catalog. If so, reuse that key. Pay special attention to common UI strings that belong under `common.*`
11. **Replace strings** in source files with the framework-appropriate function call. This includes all string locations confirmed in step 4 — rendering layer, data definitions, custom components, and any intl threading needed
12. **Update the catalog** — append new key-value pairs to the single JSON catalog (create it if it doesn't exist)
13. **Add necessary imports** — if the replacement pattern requires an import (e.g., `useTranslation` hook), add it to the file
14. **Report** what was done: how many strings extracted, how many were deduplicated, which files modified, where the catalog was written. Confirm that all categories from the scope assessment in step 4 were covered. In a monorepo, break down the report by app

## Important considerations

- **String interpolation**: When a string contains template literals or variable interpolation (e.g., `` `Hello, ${name}` ``), use ICU `{variableName}` syntax in the catalog value and pass the variable as a parameter to the translation function.
- **Plurals**: If a string varies by count (e.g., "1 item" vs "5 items"), use ICU plural syntax: `{count, plural, one {# item} other {# items}}`. Do NOT use separate keys with suffixes — ICU handles it in a single key.
- **Select/Gender**: If a string varies by a category (gender, role, status), use ICU select syntax: `{variable, select, value1 {text} value2 {text} other {text}}`.
- **Context**: If the same English word means different things in different places (e.g., "Post" as a noun vs verb), use distinct keys to allow translators to provide different translations.
- **JSX**: In JSX, strings inside `{}` expressions AND bare text children both need extraction. The `<Trans>` component handles JSX with embedded elements — read the React reference for details.
- **Don't break logic**: If a string is used in a comparison (e.g., `if (status === "active")`), that's a logic string, not a user-facing one. Leave it alone even if it looks like English text.
- **Escaping in ICU**: Literal curly braces in text must be escaped with single quotes: `"Use '{' and '}' for objects"`. A literal single quote is `''`.
- **Wire interpolation values through**: Every `{placeholder}` in the catalog must have a corresponding values argument at the call site. Without it, the placeholder renders as literal text (e.g., `{name}` instead of "Alice"). This is especially easy to miss when i18n keys are stored as data (config objects, arrays, preference definitions) rather than called inline — the data structure needs a field for interpolation values, and the renderer must forward them. Before finishing, search the catalog for all `{...}` placeholders and verify each one has values wired through the rendering path. See the framework-specific reference files for detailed examples.
