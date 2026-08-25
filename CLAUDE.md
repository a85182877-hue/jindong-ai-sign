# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this project is

**筋動人生 AI 簡易身體評估** — a marketing/lead-generation site for 筋動人生 (Jin Dong Ren Sheng), a
Taiwanese 整復推拿 (traditional manual-therapy / bodywork) studio in 台中南屯. Visitors answer a short
symptom questionnaire, the answers are sent to an LLM, and the reply is a plain-language
**人體力學 (body-mechanics) analysis** that ends with a LINE booking call-to-action.

There is **no build system, no package manager, no test suite, and no CI**. The repo is four files:
two self-contained HTML pages plus two images. Everything — markup, styles, prompt, and client
logic — lives inline in each HTML file.

## Repository layout

```
index.html                      # A4 poster / desktop page — signage + AI modal (Ver 6.0.0)
mobile.html                     # Mobile-first standalone AI form (Ver 7.0.0 Mobile)
筋動人生 (廣告看板 (直式)).png     # Studio logo (referenced by both pages)
營業時間.jpg                     # Business-hours table image (index.html only)
```

Filenames contain **Chinese characters, spaces, and parentheses**. Quote them in shell commands
(`git add "筋動人生 (廣告看板 (直式)).png"`) and leave the `src` attributes exactly as they are — the
pages are served from a path where these names resolve. Both `<img>` tags carry an `onerror`
fallback (a placeholder for `index.html`, `display:none` for `mobile.html`), so a broken image path
fails quietly rather than visibly — do not treat "the page still looks fine" as proof an image loads.

## The two pages are independent, not shared

`index.html` and `mobile.html` duplicate almost everything on purpose. They differ in ways that
matter, and **a change to one is not automatically correct for the other**:

| | `index.html` | `mobile.html` |
|---|---|---|
| Layout | Fixed A4 canvas (`.a4-canvas`, `aspect-ratio: 1/1.414`), form inside a slide-up modal | Vertical scroll, form always visible |
| Model sent to backend | `selectedModel`, hard-coded to `"gemini"` | `"claude"`, with `max_tokens: 2000` |
| System prompt | Short 力學 essay, 150–200 chars, one continuous paragraph | Structured "Prompt v5.1" — per-body-part blocks + a mandatory 感覺分析 section |
| Q3 感覺 options | 痠 / 痛 / 麻 / 緊繃 / 無力 | 痠痛+緊繃 / 沉重感 (deliberately narrowed) |
| Q2 上背 label | `上背/膏肓` | `上背/肩胛骨內側` |
| Extra questions | Q5 醫生說法, Q6 補充說明 (both optional) | none |
| Result rendering | Single block, `**bold**` → `<strong>` | Paginated (`splitIntoPages`), 2 body parts per page |
| Retries | none | 3 attempts with backoff on HTTP 503 |
| Fixed closing copy | Built into the response string in JS | `#footer-text`, rendered only on the last page |

When asked to "update the form" or "change the wording", clarify **which page** — or apply the change
to both and say so explicitly.

## Backend: the Cloudflare Worker

Both pages POST to a Cloudflare Worker that holds the API keys:

```js
const WORKER_URL = "https://twilight-sky-7d28.a85182877.workers.dev";
// POST body: { model, clientInfo, systemPrompt[, max_tokens] }
// Expected reply: { result: "..." }  |  { error: "..." }
```

**The Worker source is not in this repository.** Do not invent it, and do not assume you can change
the response shape — the pages read `data.result` and `data.error` and nothing else. If a task needs
Worker-side changes, say so rather than working around it in the browser.

Never put an Anthropic, Google, or any other API key into these HTML files. The Worker indirection
exists precisely so the keys stay off the client; `index.html` labels itself "Cloudflare 安全版" for
this reason.

## Content rules — these are the important part

The system prompts encode **legal and regulatory constraints for Taiwan**, where 整復推拿 is
explicitly *not* medical practice. Treat the banned-word lists as hard requirements, not styling
preferences. Never soften, shorten, or "clean up" these blocks without being asked.

Forbidden across the generated output:

- **神經 / 壓迫神經** → say `擠壓周圍空間，影響循環產生痠麻感` instead
- Diagnosis/treatment verbs: 矯正、復位、治癒、治療、發炎、診斷
- `mobile.html` additionally bans: 骨頭、脊椎、椎間盤、無力、建議您

Approved replacements: 調理、放鬆、釋放壓力、平衡、身體順開、喚醒肌肉、找回空間.

`mobile.html` also carries a visible disclaimer ("非屬醫療器材，亦不具備疾病診斷或治療目的") — keep it.

### Safety gates run before the network call

Both pages short-circuit the LLM entirely for two selections and render a fixed refusal:

- **懷孕中** — declines service, refers to 婦產科醫師 / prenatal massage
- **近期骨折/急性發炎** — declines service, refers to 醫療院所

Three other conditions (**植入金屬物**, **骨質疏鬆**, **心血管疾病/抗凝血劑**) do *not* block the
analysis; they append a ⚠️ reminder to tell the therapist in person. Preserve this split — moving a
condition between the two groups is a safety decision, not a UI tweak.

## Conventions

- **Language**: all user-facing copy is Traditional Chinese (zh-TW). Comments in the source are
  Chinese too — match that. Do not translate copy to Simplified Chinese or English.
- **Styling**: Tailwind via the CDN `<script src="https://cdn.tailwindcss.com">`, plus a `<style>`
  block for what Tailwind utilities can't express. Font is Noto Sans TC from Google Fonts. There is
  no Tailwind config and no build step, so **arbitrary-value classes** (`text-[26px]`, `bg-[#FFF8F3]`)
  are the normal way to reach off-palette values — that is existing style, not a mistake.
- **Palette**: charcoal `#333333`, warm orange `#E06A3B`, canvas `#F8F7F3` / `#F2F0EB`, LINE green
  `#06C755`. `index.html` exposes these as `.text-charcoal` / `.bg-warm-orange` helper classes.
- **JS**: plain ES2017+, no modules, no framework. Handlers are inline `onclick` attributes; state is
  a couple of module-level `let`s (`pages`, `currentPage`, `currentWarns`, `selectedModel`).
  Elements are looked up by `id` with `document.getElementById`. Follow the same pattern rather than
  introducing a framework or a bundler.
- **Booking link**: `https://lin.ee/vkwcVbs` (official LINE). Appears in both pages' closing copy.
- **Version badge**: each page shows its own version string in the UI (`Ver 6.0.0 - Cloudflare 安全版`,
  `Ver 7.0.0 Mobile`). Bump the badge when making a user-visible change to that page.

## Known dead code

`index.html` keeps `switchModel()` and the `.model-tab` / `.model-badge-*` CSS from an earlier build
that let visitors pick Claude or Gemini. **The tab elements were removed from the DOM**, so
`switchModel()` would throw on `tabClaude.className` if anything called it — nothing does.
`selectedModel` is initialised to `"gemini"` and never changes, so `index.html` always requests
Gemini despite the comment saying the default is Claude. Leave it alone unless asked; if you are
asked to change the model for `index.html`, edit the `selectedModel` initialiser (line ~278) rather
than reviving the tabs.

## Development workflow

There is nothing to install or compile. To preview:

```bash
python3 -m http.server 8000    # from the repo root, so the Chinese image paths resolve
# then open http://localhost:8000/index.html or /mobile.html
```

Opening the files over `file://` also works, but the fetch to the Worker may be blocked by CORS —
use the local server when testing the AI path.

Verify by hand after changes:

1. Submit with no 職業 selected → alert, no request sent.
2. Submit with no 部位 → alert. (`mobile.html` also requires at least one 感覺.)
3. Tick 懷孕中 → fixed refusal copy, **no** network request.
4. Tick 骨質疏鬆 → analysis runs, ⚠️ reminder appended.
5. On `mobile.html`, pick 3+ 部位 → pagination appears and the 📌 感覺分析 block lands on the last page.
6. On `mobile.html`, tap chips on a real iOS Safari device if you touched the chip code — there is a
   `DOMContentLoaded` handler that manually toggles the checkbox and inline styles to work around
   iOS label-tap behaviour. It is easy to break from a desktop browser without noticing.

## Git

- The default branch is `main`. Deployment appears to be GitHub Pages served from the repo root, so
  **anything merged to `main` is live** — treat pushes to `main` as publishing.
- History is mostly GitHub web-UI commits ("Add files via upload", "Update mobile.html"). Prefer
  clearer messages than that for new work.
- Commit the images alongside markup when their filenames change; the `src` strings must stay in sync.
