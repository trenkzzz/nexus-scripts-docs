---
icon: user-question
---

# Nexus Helpcenter

> An AI-powered, standalone help center for FiveM — rules, jobs, server info, and a Google Gemini chat assistant, all served from one config file.

**Version:** 1.0.5 · **Author/Type:** Nexus Scripts · Standalone (framework-agnostic)

***

## 📝 Description

Nexus Help Center replaces the usual "read the Discord rules channel" workflow with an in-game information hub. Pressing a single key (`F1` by default) opens a sidebar-driven, fully animated NUI with six screens — **Home**, **Rules**, **Jobs**, **Server Information**, **AI Assistant** and **FAQ** — plus a step-by-step **How to report** modal. Every piece of content shown (rule categories, job listings, server features, external links, FAQ entries) is read from `config/config.lua`, the only file left outside the escrow. The script never touches a database and never calls into a framework, so it runs identically on ESX, QBCore, vRP, or a completely standalone server.

The AI assistant is the part that makes the script unusual. When a player sends a message, the NUI forwards it over a NUI callback to the client, the client relays it to the server, and **the server** calls the Google Gemini `generateContent` REST endpoint with `PerformHttpRequest`. The API key therefore never reaches a game client. Before any HTTP call is made, the server runs four gates in order: an empty/type check, a 500-character length cap, a per-player rate limiter (10 requests per rolling 60 seconds), and a substring-based anti-jailbreak filter that blocks roughly 32 known prompt-injection phrases. Only if all four gates pass is the request built and dispatched. The reply travels back down the same path and is rendered in the chat bubble through a small Markdown parser (bold, italics, inline code, line breaks).

Token cost is actively managed in two places. The NUI keeps the full conversation client-side but only ships the last 8 messages (`MAX_CLIENT_HISTORY`); the server then truncates that again to the last 4 messages (`MAX_HISTORY_MESSAGES`) and prepends `Config.AISystemPrompt` as a synthetic first user turn plus a canned model acknowledgement. The server also normalizes the turn order so Gemini always receives an alternating `user` / `model` sequence ending on a `user` turn — a malformed history from a tampered client cannot break the request. `Config.MaxTokens` caps the response size, and the sampling temperature is fixed at `0.7`.

The problem it solves is support load. New players ask the same twenty questions, and staff end up answering them one ticket at a time. Here the answers live in `config.lua` — authoritative, searchable, categorized — and anything not covered gets routed to an assistant that has been told, via its system prompt, to stay on topic and to point the player to Discord when it doesn't know.

***

## ✨ Features

* **Six-screen custom NUI** — a fixed sidebar with Home, Rules, Jobs, Information and AI Assistant navigation, plus a "Quick Help" block holding the Report modal and the FAQ screen.
* **Home dashboard** — hero banner with your logo (`web/logo.png`), a live `online/max` player badge, a version badge, four gradient feature cards, and a three-step quick-start onboarding list.
* **Rules screen** — renders `Config.Rules` as color-coded category cards (one card per category, with a bulleted rule list inside), each cycling through four preset gradient/icon themes, followed by a fixed "ignorance of the rules is no excuse" notice box.
* **Jobs carousel** — renders `Config.Jobs` as a one-card-at-a-time animated carousel (0.42s cubic-bezier slide) with prev/next arrows, clickable dots, an `n / total` counter, and keyboard arrow-key navigation. Each card shows name, Legal/Illegal badge, salary, difficulty (color-mapped), description, three derived "characteristics" pills, and a vertical requirements checklist.
* **Jobs statistics row** — automatically counts total jobs, legal jobs and illegal jobs from `Config.Jobs`.
* **Server information screen** — a six-field status grid (server name, region, online/max players, founding date, measured ping, version), a features grid from `Config.Features`, a four-button links grid (Discord / Store / Forum / Full Rules), and an "About us" block.
* **Measured ping, not faked** — on open, the NUI timestamps a round-trip to the resource's own `ping` NUI callback and displays the measured latency in the Information grid, falling back to `< 30ms` if the measurement fails.
* **Gemini AI assistant** — a chat screen with avatars, timestamps, an animated "thinking" spinner, four one-click suggested questions, a live `0/500` character counter, and Enter-to-send.
* **Markdown rendering in AI replies** — `**bold**`, `__bold__`, `*italics*`, `` `code` `` and newlines are converted to HTML; the raw text is HTML-escaped first, so a model reply cannot inject markup.
* **Server-side API key handling** — the Gemini key lives in `config/config.lua` and is used exclusively from `server/server.lua`. No client ever receives it.
* **Per-player rate limiting** — 10 AI requests per rolling 60 seconds per server ID, with the bucket cleared automatically on `playerDropped`.
* **Anti-jailbreak filter** — a 32-entry plain-substring blocklist (`ignora`, `olvida tus`, `modo desarrollador`, `jailbreak`, `dan mode`, `system prompt`, `bypass`, `override`, …). A hit is logged to the server console with the player's name and ID, and the player gets a refusal instead of an API call.
* **Searchable, categorized FAQ** — a card grid built from `Config.FAQs` with a live text search over both question and answer, auto-generated category filter chips, a 90-character preview with ellipsis, an empty state, and a detail modal that renders the full Markdown answer.
* **Report guide modal** — a fixed four-step walkthrough (join Discord → find the ticket channel → describe the problem → wait for staff) wired to `Config.DiscordLink`, plus a false-report warning.
* **Rebindable key** — default `F1`, registered through `RegisterKeyMapping` so players can change it in their own FiveM settings. Also available as `/helpcenter`.
* **Resource name validation** — `shared/_resource.lua` hard-errors on startup if the folder is not named exactly `nexus_helpcenter`.
* **Framework-free** — no ESX, no QBCore, no oxmysql, no SQL file. Drops into any server.

***

## 📋 Dependencies

| Dependency                                            | Required?                   | Notes                                                                                                                                                                                                             |
| ----------------------------------------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FiveM server, `fx_version 'cerulean'`, `game 'gta5'`  | **Required**                | Declared in `fxmanifest.lua`.                                                                                                                                                                                     |
| A Google AI Studio / Gemini API key                   | **Required for the AI tab** | `Config.GeminiAPIKey`. Everything else in the menu (Rules, Jobs, Info, FAQ, Report) works fine without it; only the assistant degrades to "AI assistant is not configured."                                       |
| Outbound HTTPS to `generativelanguage.googleapis.com` | **Required for the AI tab** | If your server enforces the cURL whitelist, add `set curl_enforceWhitelist 0` to `server.cfg`, otherwise `PerformHttpRequest` returns status `0`. The script prints this exact hint to console when that happens. |
| Internet access from the **client** for Google Fonts  | Optional                    | `web/index.html` loads `Bebas Neue` + `Inter` from `fonts.googleapis.com`. Without it the UI falls back to a generic sans-serif; nothing breaks.                                                                  |
| Framework (ESX / QBCore / other)                      | **Not required**            | The script is fully standalone.                                                                                                                                                                                   |
| Database / MySQL / oxmysql                            | **Not required**            | No SQL file, no queries, no persistence.                                                                                                                                                                          |
| Notification resource                                 | **Not required**            | The script shows no in-world notifications; all feedback is rendered inside the NUI.                                                                                                                              |

***

## ⚙️ Installation

1. **Unzip the resource.** Extract the folder from the cfx.re portal download into your `resources` directory. The folder **must** be named exactly `nexus_helpcenter` — `shared/_resource.lua` throws a startup error for any other name.
2. **Add your server logo.** Replace `web/logo.png` with your own PNG. It is displayed in the Home hero panel; a transparent background and a roughly square aspect ratio look best.
3.  **Allow outbound HTTP (if needed).** Add this to `server.cfg` if your server blocks un-whitelisted cURL targets:

    ```cfg
    set curl_enforceWhitelist 0
    ```
4.  **Add the resource to `server.cfg`.** There's no load-order requirement — no framework, no database — so anywhere in your list works:

    ```cfg
    ensure nexus_helpcenter
    ```
5.  **Get a Gemini API key.** Create one at [Google AI Studio](https://aistudio.google.com/app/apikey) and paste it into `Config.GeminiAPIKey`:

    ```lua
    Config.GeminiAPIKey = "AIza..."
    ```

    **Delete the demo key that ships in `config.lua` and replace it with your own** (see the FAQ).
6. **Fill in `config/config.lua`.** At minimum, change `Config.ServerName`, `Config.ServerRegion`, `Config.MaxPlayers`, `Config.FoundedDate`, `Config.Tagline`, the three link fields, and `Config.AISystemPrompt`. Then replace the demo `Config.Rules`, `Config.Jobs`, `Config.Features` and `Config.FAQs` with your own content — the defaults describe a fictional server.
7. **Restart.** A full server restart is recommended; `restart nexus_helpcenter` is enough for config-only changes.
8. **Open it.** Press `F1` (or whatever you set `Config.OpenKey` to), or type `/helpcenter` in chat. `Esc` closes the menu. Players can rebind the key freely under **FiveM → Settings → Key Bindings → FiveM**, because the script registers it through `RegisterKeyMapping`.

> **Tip:** If AI requests fail with a connection error and the server console shows `Status 0`, your server is blocking outbound HTTP. Add `set curl_enforceWhitelist 0` to `server.cfg`, or whitelist `generativelanguage.googleapis.com` with your host.

***

## 🔧 Configuration

Everything configurable lives in `config/config.lua`, the only file in `escrow_ignore`.

### General info

| Config key             | Type     | Default            | Description                                                                                                                                                                                                                                                                                             |
| ---------------------- | -------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.ServerName`    | `string` | `"Nexus Roleplay"` | Shown in the Information grid, interpolated into the Home welcome card, the Information subtitle and the "About us" paragraph.                                                                                                                                                                          |
| `Config.ServerVersion` | `string` | `"v1.0.5"`         | Shown in the Home hero badge and the Information grid. This is **your server's** version string, not the resource version — purely cosmetic.                                                                                                                                                            |
| `Config.ServerRegion`  | `string` | `"Europe - Spain"` | Displayed as "Location" in the Information grid.                                                                                                                                                                                                                                                        |
| `Config.MaxPlayers`    | `number` | `512`              | The denominator of every `online/max` counter in the UI. Set it to your actual slot count.                                                                                                                                                                                                              |
| `Config.FoundedDate`   | `string` | `"2023"`           | Displayed as "Opening Date". Free-form string, so `"March 2023"` works too.                                                                                                                                                                                                                             |
| `Config.Tagline`       | `string` | _server tagline_   | One-line slogan under the sidebar logo.                                                                                                                                                                                                                                                                 |
| `Config.OpenKey`       | `string` | `"F1"`             | Default key passed to `RegisterKeyMapping('helpcenter', …, 'keyboard', Config.OpenKey)`. Use a FiveM key name (`F1`–`F12`, `M`, `HOME`, `DELETE`, …). **This is only the default** — once a player has bound the key, FiveM stores their own choice, and changing this value will not move it for them. |

### Links

| Config key           | Type           | Default                           | Description                                                                                                                   |
| -------------------- | -------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `Config.DiscordLink` | `string` (URL) | `"https://discord.gg/yourserver"` | Used twice: the "Discord" button in the Information links grid, and the "Go to Discord" button in step 1 of the Report modal. |
| `Config.StoreLink`   | `string` (URL) | `"https://store.yourserver.com"`  | The "Store" button in the Information links grid.                                                                             |
| `Config.ForumLink`   | `string` (URL) | `"https://forum.yourserver.com"`  | The "Forum" button in the Information links grid.                                                                             |

> The fourth button in that grid, **"Full Rules"**, is hardcoded to `#` in `web/script.js` and is **not** configurable — there is no `Config.RulesLink` in this build. This is recorded as a known limitation; see the FAQ.

### AI

| Config key              | Type                         | Default           | Description                                                                                                                                                                                                                                                                                          |
| ----------------------- | ---------------------------- | ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Config.GeminiAPIKey`   | `string`                     | _a live demo key_ | Your Google Gemini API key. Appended as the `?key=` query parameter on the `generateContent` call. **Replace it before going live.**                                                                                                                                                                 |
| `Config.GeminiModel`    | `string`                     | `"gemma-3-4b-it"` | The model segment of the endpoint URL: `https://generativelanguage.googleapis.com/v1beta/models/<MODEL>:generateContent`. Any model your key can reach on the `v1beta` REST surface works (`gemini-2.0-flash`, `gemma-3-4b-it`, …). A bigger model means better answers but higher cost and latency. |
| `Config.MaxTokens`      | `number`                     | `400`             | Sent as `generationConfig.maxOutputTokens`. Hard cap on reply length. Too low and answers get cut off mid-sentence; `400` is roughly 300 words.                                                                                                                                                      |
| `Config.AISystemPrompt` | `string` (multiline `[[ ]]`) | a persona block   | The assistant's entire personality and scope. Injected as the **first** `user` turn of every request, followed by a canned `model` acknowledgement. This is where you set the language, the tone, the topics it may discuss, and what it should do when it doesn't know.                             |

The shipped prompt tells the model: its name is "Nexus AI", answer always in Spanish, be brief and friendly, only help with server topics (rules, jobs, economy, factions, how to join, how to play), refuse off-topic questions, defer to Discord staff when unsure, and use Markdown where it aids clarity. **If you want the assistant to answer in English (or any other language), rewrite this block accordingly** — the response language is decided entirely by this prompt, not by any locale setting, and there is no automatic "reply in the player's own language" behavior in this build.

A practical tip: the assistant has no access to `Config.Rules`, `Config.Jobs` or `Config.FAQs`. If you want it to quote your actual rules and salaries, paste the key facts into `Config.AISystemPrompt`. That costs input tokens on every message, so keep it to the facts people actually ask about.

### `Config.Rules`

An array of category objects, rendered in order. Each produces one card on the Rules screen.

```lua
Config.Rules = {
    {
        title = "General Rules",     -- card heading
        color = "cyan",               -- see note below
        rules = {                     -- one bullet per entry, unlimited
            "Respect all players and staff on the server.",
            "Cheats, hacks and unauthorized modifications are forbidden.",
        },
    },
}
```

| Field   | Type       | Description                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `title` | `string`   | Card heading.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `color` | `string`   | **Currently ignored.** `web/script.js` assigns each card a gradient/icon from a fixed four-entry `styles` array by index (`i % 4`), so card 1 is blue/cyan, card 2 violet, card 3 red/orange, card 4 amber, card 5 blue again, and so on. Keep the key for forward compatibility, but don't expect it to change anything. Accepted values elsewhere in the catalogue (`cyan`, `blue`, `violet`, `red`, `amber`, `emerald`) have no effect here. |
| `rules` | `string[]` | The bullet list inside the card. No length limit.                                                                                                                                                                                                                                                                                                                                                                                               |

### `Config.Jobs`

An array of job objects, rendered as carousel slides in order.

```lua
Config.Jobs = {
    {
        name         = "Police Officer",
        category     = "Legal",                     -- "Legal" or anything else
        salary       = "$2,500 - $5,000",            -- free-form string
        difficulty   = "Alta",                       -- Baja | Media | Alta | Muy Alta
        description  = "Keep order and safety in the city.",
        requirements = { "Level 5+", "No criminal record", "Interview required" },
    },
}
```

| Field          | Type       | Description                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`         | `string`   | Job title, shown as the card heading and as the tooltip on its carousel dot.                                                                                                                                                                                                                                                                                                                                                                 |
| `category`     | `string`   | Drives the badge color and the legal/illegal counters. The string **`"Legal"` exactly** renders a green badge and counts as legal; **any other value** renders a red badge, counts as illegal, and switches the card's derived pills to "cash payment" / "high-risk environment".                                                                                                                                                            |
| `salary`       | `string`   | Free-form. Displayed verbatim in the top-right "Salary" stat.                                                                                                                                                                                                                                                                                                                                                                                |
| `difficulty`   | `string`   | Recognized values are the Spanish strings `"Baja"` (green), `"Media"` (amber), `"Alta"` (orange) and `"Muy Alta"` (red). Any other value renders white. It also picks the "demand" pill: Baja → _high demand_, Media → _moderate demand_, Alta → _highly sought_, Muy Alta → _exclusive_, anything else → _available_. There is no automatic translation of these values — write them exactly as shown regardless of your server's language. |
| `description`  | `string`   | Paragraph in the card's left column. Keep it to \~2 lines; the column does not scroll.                                                                                                                                                                                                                                                                                                                                                       |
| `requirements` | `string[]` | Checklist pills in the card's right column. Three or four entries fit comfortably.                                                                                                                                                                                                                                                                                                                                                           |

The card gradient is also index-based (`i % 8`), cycling through eight preset color schemes.

### `Config.Features`

```lua
Config.Features = {
    "Advanced economy system with investments and properties",
    "20+ legal and illegal jobs with progression",
}
```

A flat `string[]`. Each entry becomes one pill in the "Featured Highlights" grid on the Information screen, colored by index from a six-entry palette.

### `Config.FAQs`

```lua
Config.FAQs = {
    {
        question = "How do I join the server?",
        answer   = "Open FiveM, search for your server name and connect. **Markdown works here.**",
        category = "General",
    },
}
```

| Field      | Type     | Description                                                                                                                                                                                                                   |
| ---------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `question` | `string` | Card title and one of the two fields searched by the live search box.                                                                                                                                                         |
| `answer`   | `string` | Full answer, shown in the detail modal. Supports the same Markdown subset as AI replies (`**bold**`, `*italic*`, `` `code` ``, newlines). The card preview strips `**` markers and truncates to 90 characters. Also searched. |
| `category` | `string` | Free-form. Every distinct value automatically becomes a filter chip next to the permanent "All" chip. Entries with no `category` fall back to displaying "General" on the card, but will **not** get a chip.                  |

### Hardcoded limits (not in `config.lua`)

These live inside the escrow and cannot be changed without a new build. There is **no** `Config.MaxRequestsPerMinute`, `Config.MaxHistoryMessages`, `Config.MaxMessageLength`, `Config.Temperature` or `Config.JailbreakPatterns` in this version — they are listed here so you know these limits exist and where:

| Value                          | Where                                               | Effect                                                              |
| ------------------------------ | --------------------------------------------------- | ------------------------------------------------------------------- |
| `MAX_REQUESTS_PER_MINUTE = 10` | `server/server.lua`                                 | AI requests allowed per player per rolling 60s.                     |
| `MAX_HISTORY_MESSAGES = 4`     | `server/server.lua`                                 | Conversation turns forwarded to Gemini.                             |
| `MAX_CLIENT_HISTORY = 8`       | `web/script.js`                                     | Turns the NUI ships to the server.                                  |
| `500`                          | `server/server.lua` + `maxlength` on the chat input | Maximum characters per player message.                              |
| `temperature = 0.7`            | `server/server.lua`                                 | Gemini sampling temperature.                                        |
| \~32 blocked phrases           | `server/server.lua`                                 | Plain-substring anti-jailbreak blocklist (see Compatibility / FAQ). |

***

## 🌐 Locales & Editable Strings

**This resource has no locale system.** There is no `locales/` folder, no `locales.lua`, and no `Config.Locale` table in this build.

What you **can** edit, because it lives in `config/config.lua` (outside the escrow):

* Every rule, rule category title, job field, server feature, FAQ question/answer/category.
* `Config.ServerName`, `Config.ServerVersion`, `Config.ServerRegion`, `Config.FoundedDate`, `Config.Tagline`, the three links.
* `Config.AISystemPrompt` — which in practice controls the **assistant's** output language.

What you **cannot** edit, because it is inside the escrow:

* All NUI chrome in `web/index.html` and `web/script.js`: sidebar labels (Navigation, Home, Rules, Jobs, Information, AI Assistant, Quick Help, Report, FAQ), page headings, the four Home feature cards, the three quick-start steps, all column labels on job cards (Description, Characteristics, Requirements, Salary, Difficulty), the derived job pills, the Information grid labels, the entire "About Us" and Report-modal copy, the four suggested questions, the AI greeting message, placeholders and the `0/500` counter.
* All server-side player-facing strings in `server/server.lua`: the invalid-message, message-too-long, rate-limit, jailbreak-refusal, assistant-not-configured, internal-error, too-many-requests, connection-error, connection-error-with-code, and response-parsing-error messages.
* The `chat:addSuggestion` text and the `RegisterKeyMapping` description in `client/client.lua`.
* A hardcoded server-name reference inside the jailbreak refusal message in `server/server.lua`, which will keep showing the original demo server name even after you've renamed everything else in `config.lua`.

**Net effect: the interface is Spanish-only and cannot be translated by the server owner in this version.** Against the Nexus catalogue's target language set (`es`, `en`, `de`, `fr`, `it`, `pt`, `zh`), this resource currently covers one. This is the highest-priority known limitation of the current build — your own content (rules, jobs, FAQs, features, server info) can be written in any language and the AI's reply language is controlled by `Config.AISystemPrompt`, but the surrounding interface chrome cannot be changed.

***

## 🔗 Compatibility

| System                                 | How it is handled                                                                                                                                                                                                                                                    |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frameworks**                         | None used. No ESX, no QBCore, no framework export is called anywhere in the three Lua files. The resource is genuinely standalone and will run on an empty FiveM server.                                                                                             |
| **Database**                           | None. No SQL file, no `MySQL`/`oxmysql` call, no persistence. Chat history lives in a JavaScript variable and is wiped every time the menu is opened.                                                                                                                |
| **OneSync / non-OneSync**              | Works with both. The online player counter is based on the players currently in the client's own scope (see the player-count FAQ entry for the practical effect of this).                                                                                            |
| **Notifications**                      | Not used. There is no `nexus_notify` / `okokNotify` / `mythic_notify` branch and no `functions.lua`. All player feedback is rendered inside the NUI as chat bubbles.                                                                                                 |
| **TextUI**                             | Not used. The menu opens from a keybind, not from a world interaction point.                                                                                                                                                                                         |
| **Target (`ox_target` / `qb-target`)** | Not used.                                                                                                                                                                                                                                                            |
| **Keys / fuel / banking / inventory**  | Not used.                                                                                                                                                                                                                                                            |
| **Key binding**                        | `RegisterKeyMapping('helpcenter', 'Open Help Center', 'keyboard', Config.OpenKey)`. Because this is a proper key mapping, players can rebind it in FiveM's own settings and it will not fight other resources over the raw key.                                      |
| **Chat**                               | Registers one chat suggestion, `/helpcenter`. Works with the default `chat` resource; harmless if you run a custom chat that ignores `chat:addSuggestion`.                                                                                                           |
| **AI provider**                        | Google Gemini / Gemma via the `v1beta` `generateContent` REST endpoint only. There is no OpenAI, Anthropic or Ollama branch — the URL is assembled from `Config.GeminiModel` and `Config.GeminiAPIKey` against a hardcoded `generativelanguage.googleapis.com` host. |
| **Escrow**                             | Only `config/config.lua` is open (`escrow_ignore`). `client/client.lua`, `server/server.lua` and the entire `web/` UI are escrowed and cannot be edited.                                                                                                             |
| **Resource name**                      | Must be exactly `nexus_helpcenter` — NUI callbacks and `shared/_resource.lua`'s startup check depend on it.                                                                                                                                                          |
| **Other help/rules menus**             | No conflict except the keybind. If another resource claims `F1`, change `Config.OpenKey` or let the player rebind.                                                                                                                                                   |

***

## 💻 Developer API

Everything in this section was read directly out of `client/client.lua`, `server/server.lua` and `web/script.js`.

### Exports

**None.** Neither `client/client.lua` nor `server/server.lua` registers any `exports[...]` / `exports('name', fn)`. There is no client- or server-side export surface.

If you need to drive this resource from another script, use the network events below and the commands table — they are the only public surface.

### Commands

| Command       | Side   | Restricted                         | Description                                                                                                           |
| ------------- | ------ | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `/helpcenter` | Client | No (`RegisterCommand(..., false)`) | Toggles the menu: opens it if closed, closes it if open. Also bound to `Config.OpenKey` through `RegisterKeyMapping`. |

### Events — Emitted

| Event                         | Side                                   | Payload                                                                                                                     | When                                                                                                                                                                                    |
| ----------------------------- | -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus:helpcenter:askAI`      | Client → Server (`TriggerServerEvent`) | `message` _(string, ≤500 chars)_, `history` _(array of `{ role = "user"\|"assistant", content = string }`, last 8 entries)_ | Fired from the `askAssistant` NUI callback every time a player sends a chat message in the AI tab.                                                                                      |
| `nexus:helpcenter:aiResponse` | Server → Client (`TriggerClientEvent`) | `response` _(string)_, `isError` _(boolean)_                                                                                | Fired on every outcome of an AI request: successful Gemini reply (`isError = false`), or any of the validation failures / rate limit / jailbreak block / HTTP error (`isError = true`). |
| `chat:addSuggestion`          | Client (local `TriggerEvent`)          | `'/helpcenter'`, suggestion text                                                                                            | Once, at resource start, to register the chat autocomplete entry.                                                                                                                       |

### Events — Listened

| Event                         | Side                        | Payload                                                  | Purpose                                                                                                                                                                                               |
| ----------------------------- | --------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nexus:helpcenter:askAI`      | Server (`RegisterNetEvent`) | `message: string`, `history: { { role, content }, ... }` | The AI request handler. Validated server-side (length, rate limit, jailbreak filter, API key), then runs validation → rate limit → jailbreak filter → key check, then `PerformHttpRequest` to Gemini. |
| `nexus:helpcenter:aiResponse` | Client (`RegisterNetEvent`) | `response`, `isError`                                    | Forwards the reply into the NUI as `{ action = "aiResponse", response, isError }`.                                                                                                                    |
| `playerDropped`               | Server (`AddEventHandler`)  | —                                                        | Clears `playerRequests[source]`, releasing that player's rate-limit bucket.                                                                                                                           |

> ⚠️ `nexus:helpcenter:askAI` is a plain `RegisterNetEvent` with no source validation beyond the four content gates. A malicious client can call it directly with a crafted `history` array, bypassing `MAX_CLIENT_HISTORY`. The server's own `MAX_HISTORY_MESSAGES = 4` truncation and the 500-character cap on the message limit the damage, and the rate limiter caps the cost at 10 calls/minute/player — but be aware that the payload is attacker-controlled and that the `history` array itself has **no length or size validation** before it is iterated.

### NUI Callbacks (UI → Lua)

These are the resource's internal HTTP endpoints. Another resource cannot call them (NUI callbacks are scoped to the owning resource's frame), but they are documented so you understand the data flow and can reason about the ping measurement.

| Callback       | Request body                                           | Response                                    | Purpose                                                                                                                                        |
| -------------- | ------------------------------------------------------ | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `closeMenu`    | `{}`                                                   | `'ok'`                                      | Releases NUI focus and sets `isMenuOpen = false`. Called by the in-UI close path and by `Esc`.                                                 |
| `ping`         | `{}`                                                   | `'ok'`                                      | A deliberate no-op. The NUI measures the round-trip time to this callback with `performance.now()` and displays it as the "Average Ping" stat. |
| `askAssistant` | `{ message: string, history: Array<{role, content}> }` | `'ok'` (immediately, before the AI replies) | Relays the message to the server via `nexus:helpcenter:askAI`. The actual answer arrives asynchronously over `nexus:helpcenter:aiResponse`.    |

### NUI Messages (Lua → JavaScript)

| `action`     | Fields                                                                                                                                                                 | When                                                                                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openMenu`   | `serverName`, `version`, `region`, `playerCount`, `maxPlayers`, `discordLink`, `storeLink`, `forumLink`, `foundedDate`, `tagline`, `rules`, `jobs`, `features`, `faqs` | On every open. The entire config payload is re-sent each time, so editing `config.lua` and running `restart nexus_helpcenter` is enough to see changes. |
| `closeMenu`  | —                                                                                                                                                                      | On every close. Triggers the 280ms closing animation.                                                                                                   |
| `aiResponse` | `response`, `isError`                                                                                                                                                  | When an AI reply (or error) arrives from the server.                                                                                                    |

### Editable Functions (`functions.lua`)

**There is no `functions.lua` in this resource.** The `escrow_ignore` block in `fxmanifest.lua` contains exactly one entry:

```lua
escrow_ignore {
    'config/config.lua',
}
```

Consequences:

* There is no notification-system abstraction layer to edit, because the script never sends an in-world notification.
* There is no hook point for logging, permission checks or custom AI providers.
* `client/client.lua`, `server/server.lua` and all of `web/` are protected. The only customization surface is `config/config.lua` and replacing `web/logo.png`.

If you need to react to AI usage (Discord logging, abuse monitoring, analytics), hook the network events from your own resource — see the integration examples below.

### Database Schema

**None.** This resource creates no tables, ships no `.sql` file and executes no queries.

### Integration Examples

A standalone companion resource that logs every AI question to a Discord webhook and keeps its own per-player counter — no changes to `nexus_helpcenter` required:

````lua
-- nexus_helpcenter_logger/server.lua

local WEBHOOK = 'https://discord.com/api/webhooks/XXXX/YYYY'
local askCount = {}

-- Listening on the same event name attaches a second handler; the
-- original handler in nexus_helpcenter still runs normally.
RegisterNetEvent('nexus:helpcenter:askAI', function(message, history)
    local src = source
    if type(message) ~= 'string' then return end

    askCount[src] = (askCount[src] or 0) + 1

    PerformHttpRequest(WEBHOOK, function() end, 'POST', json.encode({
        embeds = {{
            title = 'Help Center — question',
            description = ('```%s```'):format(message:sub(1, 1000)),
            color = 9647850,
            fields = {
                { name = 'Player', value = GetPlayerName(src) or '?', inline = true },
                { name = 'ID',     value = tostring(src),            inline = true },
                { name = 'Asked',  value = tostring(askCount[src]),  inline = true },
                { name = 'Turns',  value = tostring(type(history) == 'table' and #history or 0), inline = true },
            },
        }},
    }), { ['Content-Type'] = 'application/json' })
end)

-- Catch the replies too, including blocked/rate-limited ones.
AddEventHandler('nexus:helpcenter:aiResponse', function() end) -- server-side no-op placeholder

-- Expose the counter so other resources can read it.
exports('GetAskCount', function(src)
    return askCount[src] or 0
end)

AddEventHandler('playerDropped', function()
    askCount[source] = nil
end)
````

Reading the answer side requires a client-side listener, because `nexus:helpcenter:aiResponse` is a server→client event:

```lua
-- nexus_helpcenter_logger/client.lua

RegisterNetEvent('nexus:helpcenter:aiResponse', function(response, isError)
    if isError then
        print(('[logger] Help Center error: %s'):format(response))
    end
end)
```

A config-only integration — pulling your live job list into the help center instead of maintaining two copies — is just Lua in `config.lua`, because it is a normal shared script:

```lua
-- config/config.lua, at the bottom
Config.Jobs = {}
for _, job in ipairs(MyServerJobDefinitions) do
    Config.Jobs[#Config.Jobs + 1] = {
        name         = job.label,
        category     = job.illegal and 'Illegal' or 'Legal',
        salary       = ('$%d - $%d'):format(job.minPay, job.maxPay),
        difficulty   = job.difficulty or 'Media',
        description  = job.description or '',
        requirements = job.requirements or {},
    }
end
```

> Note that `config/config.lua` is a `shared_script`, so it is loaded on both client and server. **Do not put secrets other than the Gemini key in it**, and be aware that the Gemini key itself is only safe because nothing in `client/client.lua` reads it — it is, however, present in the client's Lua state.

Opening the help menu from another resource (for example from a tutorial or a ped interaction) only needs the command:

```lua
-- client side
ExecuteCommand('helpcenter')
```

Showing the help center automatically the first time a player spawns:

```lua
-- client side
local shown = false
AddEventHandler('playerSpawned', function()
    if shown then return end
    shown = true
    Wait(3000)
    ExecuteCommand('helpcenter')
end)
```

***

## ❓ FAQ

**Q: The AI tab says "AI assistant is not configured. Contact an administrator." What now?** A: `Config.GeminiAPIKey` is empty or still equal to the literal placeholder key that ships with the resource. Paste a real key from Google AI Studio. Note that the guard only recognizes that one exact placeholder string, so if you blanked the key to something like `"YOUR_KEY_HERE"` you'll get a `400` HTTP error instead of this friendly message.

**Q: The AI replies "Connection error" and the console prints `Status 0 → add 'set curl_enforceWhitelist 0' to server.cfg`.** A: Exactly what it says. FiveM's cURL whitelist is blocking the request to `generativelanguage.googleapis.com`. Add `set curl_enforceWhitelist 0` to `server.cfg` and restart. If you're behind a corporate firewall or a hosting provider that blocks outbound HTTPS, you'll need them to allow that host.

**Q: I get "Connection error (code 400)" / "(code 403)" / "(code 404)".** A: `400` usually means a malformed key or a model name your key can't use; `403` means the key is invalid, revoked, or the Generative Language API isn't enabled on that Google Cloud project; `404` means `Config.GeminiModel` doesn't exist on the `v1beta` surface. Check the full error in your server console — the script prints the first 200 characters of Google's own error body, which names the problem precisely.

**Q: Players get "Too many requests" even with few people online.** A: Two different limits produce similar messages. The rate-limit message is **our own** limiter — 10 messages per minute per player, not configurable in this version. A message that instead reflects Google returning HTTP `429` is **your API quota**. Free-tier Gemini keys have low per-minute request limits, so a busy server will hit them. Either move to a paid key or switch `Config.GeminiModel` to a cheaper/faster model.

**Q: Can the AI answer in my players' language?** A: Not automatically in this version — there's no "detect and reply in the player's language" behavior. The shipped system prompt tells the model to always answer in Spanish. If you want a different language (or want it to match the player), rewrite `Config.AISystemPrompt` yourself; the response language is decided entirely by that prompt.

**Q: How do I teach the AI about my server?** A: Add the information (rules summary, job details, prices, how to apply to factions, etc.) to `Config.AISystemPrompt`. The assistant has no automatic access to `Config.Rules`, `Config.Jobs` or `Config.FAQs` — they are never sent to Google. Keep the prompt concise, since it's sent with every question and costs input tokens.

**Q: The player count shows something like `14/512` when 60 players are online.** A: Known limitation. The count comes from `#GetActivePlayers()` on the **client**, which only returns players currently streamed in around that client — not the real server population. On a busy server the number will always be too low, and on an empty map it will read `1`. Treat the player counters in this version as decorative rather than exact; the fix would require a server-side count pushed to the NUI.

**Q: Can I translate the menu into English (or another language)?** A: Only partially, in this version. Your _content_ — rules, jobs, features, FAQ, server info — is all in `config.lua` and can be written in any language, and you control the assistant's language through `Config.AISystemPrompt`. But the interface chrome (navigation labels, page headings, button text, the report guide, error messages) is hardcoded in Spanish inside the escrowed `web/` and `server/` files and cannot be changed by the server owner. There is no `locales/` folder or `Config.Locale` option in this build.

**Q: Where does the "Average Ping" number come from? Is it my real ping to the server?** A: No. It's the round-trip latency from the NUI frame to the resource's own `ping` callback and back — essentially a measure of local NUI/Lua responsiveness, usually a few milliseconds. It is not network latency to the game server. If the measurement fails it displays the literal string `< 30ms`.

**Q: The "Full Rules" button in the Information tab does nothing.** A: Correct — it's hardcoded to `href="#"` in the escrowed `web/script.js` and there is no config key for it in this version. The other three buttons (Discord, Store, Forum) are wired to `Config.DiscordLink`, `Config.StoreLink` and `Config.ForumLink`.

**Q: Why does the `color` field in `Config.Rules` do nothing?** A: Because the NUI ignores it. Rule-card gradients are assigned by position (`index % 4`), not by the `color` value. Reordering your categories changes their colors; editing `color` does not.

**Q: My perfectly innocent question got blocked with the jailbreak refusal message.** A: You hit the anti-jailbreak filter. It's a plain substring match (not whole-word), so legitimate phrases containing fragments like `ignora`, `olvida`, `actúa como`, `bypass`, `override` or `disable` are blocked too — for example a question like "how does a rookie act as a police officer?" contains the Spanish fragment for "act as" and will be refused. Every block is logged to the server console with the player's name and the full message, so you can see what tripped it. The blocklist is inside the escrow and cannot be edited in this version.

**Q: Does the assistant know my rules and job list?** A: Not automatically. It only sees `Config.AISystemPrompt` plus the last four conversation turns. `Config.Rules`, `Config.Jobs` and `Config.FAQs` are never sent to Google. If you want it to answer with your real numbers, summarize them inside `Config.AISystemPrompt`.

**Q: Is conversation history saved anywhere?** A: No. It lives in a JavaScript variable in the NUI and is reset every time the menu opens. Nothing is written to disk or to a database. Only the last 8 turns are ever sent to the server, and the server forwards at most 4.

**Q: The F1 key doesn't open the menu.** A: `Config.OpenKey` only sets the **default** binding. If the player (or another resource) already bound something to that key, FiveM keeps the player's own stored binding. Open **FiveM → Settings → Key Bindings → FiveM**, find the Help Center entry, and set it there. `/helpcenter` always works regardless.

**Q: Does it work without ESX or QBCore?** A: Yes. There is no framework dependency at all, no database and no SQL import. `ensure nexus_helpcenter` anywhere in `server.cfg` is the whole installation.

### Before opening a ticket

* Make sure the resource folder name is exactly **`nexus_helpcenter`** — any other name hard-errors at startup with a message in your console.
* Make sure you're using the latest version of the resource (current: **v1.0.5**).
* Check your **server console** first. Every failure path in this script prints a diagnostic line prefixed with `[NexusHelpCenter]`, including the raw HTTP status and the first 200 characters of Google's error body.
* Confirm you replaced the demo `Config.GeminiAPIKey` with your own key and that `set curl_enforceWhitelist 0` is in `server.cfg` if your host needs it.
* Re-read this FAQ.

***

## 📋 Changelog

### v1.0.5 — current

Version as declared in `fxmanifest.lua` (`version '1.0.5'`). The server boot message confirms it on load: `[NexusHelpCenter] Backend loaded - Nexus Help Center v1.0.5 (Gemini | Token-Optimized)`.

* Gemini-backed AI assistant with server-side API key handling.
* Token optimization: client ships the last 8 turns, server forwards the last 4, `Config.MaxTokens` caps the reply.
* Server-side anti-jailbreak substring filter (\~32 patterns) with console logging of blocked attempts.
* Per-player rate limiting at 10 requests per 60 seconds, cleaned up on `playerDropped`.
* Six-screen NUI: Home, Rules, Jobs carousel, Server Information, AI Assistant, searchable FAQ, plus the Report modal.
* Measured NUI round-trip ping shown in the Information grid.
* Resource-name validation on startup via `shared/_resource.lua`.

> **Documentation note:** an earlier internal draft of this document (labeled version 1.1.0) described a full locale system (7 languages), configurable rate-limiting and jailbreak-pattern lists, a configurable `Config.Temperature`, whole-word/phrase jailbreak matching with accent folding, and a configurable "Full Rules" link. None of those are present in this 1.0.5 build — this document has been corrected to match the code actually shipped. If a future release adds them, this section will be updated accordingly.

No earlier changelog history is present in the resource.
