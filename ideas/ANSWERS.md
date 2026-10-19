# Answers to the questions in `README.md`

> TL;DR
> - **Q1 (Cursor / Antigravity)**: ✅ Yes, both support a **`stop` hook** in `hooks.json` that runs a shell command when the agent finishes → just `curl` our API.
> - **Q2 (separate Windows / Mac apps?)**: ❌ No. Use **one codebase with Tauri v2** (Rust core + web UI). Performance is near-native, and the bottleneck is the network, not the UI. Build/test on **macOS first**.
> - **BE**: **TypeScript + Hono on Cloudflare Workers**, with **Durable Objects** (realtime WebSocket + per-user queue + snooze timers) and **D1** (SQLite) for config. Free, always on, no cold start, free `*.workers.dev` subdomain.
> - **FE**: **Tauri v2 + React + TypeScript**. Rust handles the WebSocket, TTS and command execution.
> - **Auth**: **Google** (OIDC). The BE only *verifies* Google ID tokens; it never stores passwords. Phase 1 (internal use) can ship with static tokens first.

---

## Q1 — Can Cursor IDE and Antigravity IDE call our API when the agent finishes?

**Yes, both of them can.** Each IDE has a lifecycle-hook system that runs a shell command when the agent loop stops.

### Cursor — `hooks.json`, event `stop`

- Location: global `~/.cursor/hooks.json` (recommended, covers every project) or per project `.cursor/hooks.json`.
- The `stop` event fires when an agent run **completes, aborts, or errors**. The JSON on stdin includes `status` (`completed` | `aborted` | `error`), `loop_count` and `workspace_roots`.
- `"version": 1` is required.

```json
{
  "version": 1,
  "hooks": {
    "stop": [{ "command": "node ~/.noti/cursor-stop.js" }]
  }
}
```

`~/.noti/cursor-stop.js` (cross-platform, no `jq` needed):

```js
let raw = "";
process.stdin.on("data", (c) => (raw += c));
process.stdin.on("end", async () => {
  const p = JSON.parse(raw || "{}");
  const ws = (p.workspace_roots?.[0] || "").split(/[\\/]/).pop();
  await fetch(process.env.NOTI_URL + "/v1/notify", {
    method: "POST",
    headers: { Authorization: `Bearer ${process.env.NOTI_TOKEN}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      source: "cursor",
      message: `Cursor agent ${p.status ?? "finished"} in ${ws}`,
      level: p.status === "error" ? "error" : "success",
    }),
    signal: AbortSignal.timeout(5000),
  }).catch(() => {});
});
```

### Antigravity — `hooks.json`, event `Stop`

- Location: global `~/.gemini/config/hooks.json` (all projects) or per workspace `.agents/hooks.json`.
- The `Stop` event fires when the execution loop ends. The stdin JSON includes `terminationReason` (`model_stop` | `max_steps_exceeded` | `error`), `fullyIdle`, `conversationId` and `workspacePaths`.
- Commands run through `cmd /c` on Windows and `sh -c` on macOS. The hook **must print JSON on stdout**, so print `{}` to let the agent stop normally.
- ⚠️ Hooks run **synchronously and block the agent** (default timeout 30s). Keep the HTTP call short (5s timeout), or fire-and-forget.
- Tip: only notify when `fullyIdle === true`, so you don't get pinged while background tasks are still running.

```json
{
  "noti-receiver": {
    "Stop": [
      { "type": "command", "command": "node ~/.noti/antigravity-stop.js", "timeout": 10 }
    ]
  }
}
```

The script is the same as the Cursor one: map `terminationReason` to `level`, use `source: "antigravity"`, and **always end with `console.log("{}")`**.

> The BE will host both scripts under `/integrations/` so setup is a copy-paste job (see BE plan).

---

## Q2 — Separate Windows and Mac apps for better performance?

**Recommendation: one cross-platform codebase with Tauri v2. Don't build two native apps.**

| Option | Perf / RAM | Effort | Verdict |
|---|---|---|---|
| Native Swift (mac) + C#/WinUI (win) | Best (~20–40 MB) | **2× code**, 2 languages | Overkill |
| **Tauri v2** (Rust + system webview) | Near-native (~30–60 MB idle, ~10 MB installer) | 1 codebase | ✅ **Pick this** |
| Electron | Heavy (~150–300 MB) | 1 codebase | Too heavy for an always-on tray app |
| Flutter desktop | Good | 1 codebase, Dart | OK, but weaker tray/shell/OS plugins |

Why separate native apps don't make it faster:
- Latency is decided by **BE → WebSocket → client**, which is the same in every option. Tauri handles the message in Rust in microseconds.
- The things that matter for this app (tray icon, native notifications, TTS, running *allow-listed* shell commands, autostart) are all **official Tauri v2 plugins** or small Rust crates that call the native OS APIs (`AVSpeechSynthesizer` on macOS, WinRT/SAPI on Windows).
- Tauri's shell plugin has a **built-in command allow-list (scope)**, which is exactly what your "FE has a list of allowed CLI" requirement needs.

**Mac first**: develop and test on macOS, but the same code builds for Windows with `tauri build` (CI can build both).

> ⚠️ If you actually meant **iOS (iPhone)**, not macOS: iOS apps can't keep a WebSocket open in the background, so the phone would need **APNs push**. That requires the paid Apple Developer Program ($99/yr). Tauri v2 can target iOS later. The BE design already supports adding a "push channel" per device.

---

## Open question 1 — Language and framework (BE)

**TypeScript + [Hono](https://hono.dev) on Cloudflare Workers.**

- One language (TS) across the BE and the FE UI.
- Hono is small and fast, built for Workers, and has first-class validation (`zod`) and OpenAPI generation.
- Cloudflare **Durable Objects** fit this problem very well:
  - **One Durable Object per user** = that user's realtime hub. It holds the WebSocket connections of all their devices (using the *Hibernation API*, so idle sockets cost nothing).
  - It has **built-in SQLite storage**, which serves as the persistent inbox/queue. If the FE is offline, notifications are stored and **replayed when the FE reconnects**.
  - It has **Alarms**, a native timer used for **snooze** ("send again in 5m").
- So you **don't need a separate queue service** (no Redis, no RabbitMQ). The Durable Object is the queue and is consistent per user, which makes scaling to many users natural: one object per user, spread out automatically.

Alternatives I considered:

| Stack | Why not (as primary) |
|---|---|
| NestJS/Express + Postgres + Redis on Render free | Render free **sleeps after 15 min**, cold start ~30–60s, and WebSockets drop on sleep |
| Supabase (Postgres + Realtime + Edge Functions) | Good option. Free project **pauses after 7 days of inactivity**, and snooze timers are awkward (needs pg_cron) |
| Firebase (FCM + Firestore + Functions) | Cloud Functions need the paid Blaze plan |
| Go/NestJS on an **Oracle Cloud Always-Free VM** | Truly free and always on, but you run the server yourself (OS patching, TLS, process manager). **Plan B** if you ever leave Cloudflare |

## Open question 2 — Free deployment + domain

**Cloudflare Workers free plan** → `https://noti-reciever.<your-account>.workers.dev` (free subdomain with HTTPS; no domain needed).

Free limits are far above what one user needs:

| Resource | Free limit | Our expected use (1 user) |
|---|---|---|
| Worker requests | 100k / day | ~few hundred |
| Durable Object requests (incl. WS messages, alarms) | 100k / day | ~1–2k |
| DO duration | 13k GB-s / day | Very small (hibernation) |
| DO SQLite storage | 5 GB / account | KBs–MBs |
| D1 | 5 GB, 5M rows read / day | Very small |

Deploy with `wrangler deploy` from GitHub Actions on every push to `main`.

## Open question 3 — FE tech stack

**Tauri v2 + React + TypeScript + Vite** (UI), with **Rust** in the Tauri core for:
- WebSocket client (`tokio-tungstenite`). It runs in Rust, not in the webview, so it isn't throttled when the window is hidden.
- TTS (`tts` crate → native voices).
- Executing allow-listed commands (Tauri shell plugin with scope).
- Plugins: `notification`, `shell`, `opener`, `autostart`, `single-instance`, `store`, `updater`, `deep-link`, `global-shortcut`.

Details are in [FE_PLAN.md](./FE_PLAN.md).

## Open question 4 — AuthN / AuthZ

**Is auth needed?** For ingestion: **yes, always** (it's a public API). For users: only when you go multi-user. Plan:

1. **Phase 1 (internal, ship fast)**
   - **Ingest tokens**: long random tokens (`nrt_...`), stored **hashed** (SHA-256) in D1, **one per integration** (Jenkins, GitLab, Cursor, ...) so each can be revoked on its own. This is your "fixed private token", just revocable.
   - **Device token** for the FE, created once by a CLI/admin endpoint protected by an `ADMIN_SECRET`.
   - Every table already has `user_id` (a single seeded user), so nothing needs a rewrite later.
2. **Phase 2 (multi-user) → Google (your #1 priority)**
   - FE: OAuth 2.0 **Authorization Code + PKCE** with a loopback redirect (Google "Desktop app" client), done in the system browser.
   - BE: **verifies the Google ID token** against Google's JWKS (`jose` library). No passwords and no custom identity logic. It then creates the user if needed and issues a revocable **device token**.
   - Users then create their own ingest tokens in the FE settings.
   - AuthZ is simple ownership: every query is scoped by `user_id`. Optional `role: admin` for you.
3. **Webhook-specific verification** (on top of tokens): GitHub `X-Hub-Signature-256` HMAC, GitLab `X-Gitlab-Token`.

Why not Facebook / GitHub / Microsoft first: Google is your stated priority and the most common. The verification code is generic OIDC, so adding Microsoft/GitHub later is just another provider config.

---

## Extra things you should know (risks / gotchas)

1. **Teams/Outlook via Power Automate — licensing** ⚠️
   The generic **HTTP action is a Premium connector**. With a standard M365 license you may not be able to call our API from a flow. Options, in order:
   1. Check whether your company license includes Premium (many enterprise ones do), then use *"When a new email arrives (V3)"* / *"When I'm @mentioned"* → **HTTP POST** to `/v1/notify`.
   2. **Microsoft Graph API** from our BE: subscribe to mail/chat change notifications with a Microsoft OAuth login (`Mail.Read`, `Chat.Read` delegated). This may need **tenant admin consent**, depending on your company's policy.
   3. Windows only: the FE could read Teams/Outlook **toast notifications** through `UserNotificationListener` (needs user permission). macOS has no public API for this.
2. **Company policy / DLP**: forwarding work email or chat content to a personal server may break company data policies, and Power Automate DLP rules can block it. Consider sending **only metadata** (sender + subject, no body).
3. **Jenkins / GitLab / GitHub — prefer webhooks over editing every pipeline**:
   - GitHub: one **org/repo webhook** for `workflow_run` events.
   - GitLab: a **project/group webhook** for "Pipeline events".
   - Jenkins: a **shared library** step `notifyNoti()` in `post { always { } }`, or the *Notification* plugin.
   - The BE provides **adapter endpoints** that convert those payloads to `{source, message, level}`. You keep your simple `source + message` API, but you don't have to touch 30 pipelines.
4. **Notification storms**: after the FE has been offline for a long time, replaying 50 notifications and *speaking* all of them is painful. The FE should **collapse the backlog** ("12 missed: 3 errors, 9 success") and only speak errors.
5. **Executing commands from a remote message is a security risk.** The BE must **never send raw shell strings**. It sends `commandId + args`. The FE runs only commands in its **local** allow-list and validates args against a regex. If the BE is ever compromised, it still can't run arbitrary code on your machine.
6. **Code signing**: an unsigned macOS app shows a Gatekeeper warning (right-click → Open, once). That's fine for internal use. Notarization needs the $99 Apple Developer account. Windows shows a SmartScreen warning.
7. Naming: `reciever` → `receiver` typo in the repo name. You can rename later with `gh repo rename noti-receiver-be` (GitHub keeps a redirect).

See [BE_PLAN.md](./BE_PLAN.md) and [FE_PLAN.md](./FE_PLAN.md) for the implementation plans.
