# Architecture

Gmail/Outlook/iCloud inbox triage. Scores messages (sender domain mismatch, bulk patterns, urgency language, unsubscribe headers) and one-taps junk, unsubscribes, archives. Web + iOS/macOS native, all OAuth. No scheduled scanning, no background polling, just user-invoked triage.

## How it runs

User signs in with Google/Microsoft/Apple OAuth (native uses system browser via `ASWebAuthenticationSession`). Worker exchanges tokens with the provider, verifies IMAP/API access, stores session in KV with provider tag. Web/native UIs call `/api/messages` (lists inbox with scores) and `/api/action` (archive/delete/unsubscribe). Worker dispatches to provider-specific logic: Gmail and Outlook use REST APIs; iCloud uses raw IMAP over cloudflare:sockets. Scoring rules (sender domain mismatch, generic bulk greeting, urgency language, unsubscribe headers) are shared. Unsubscribe uses RFC 8058 one-click (POST to List-Unsubscribe URL).

| File | What it owns |
|---|---|
| `worker.js` | Cloudflare Worker. OAuth endpoints (`/auth/start`, `/auth/callback`, `/auth/native`), message listing (`/api/messages`), actions (`/api/action`, `unsubscribe/archive/delete`). Dispatches on provider (Gmail/Outlook/iCloud) to matching logic. Shared scoring rules across all three. |
| `landing/index.html` | Inbox UI (Connect button, message list, action buttons), marketing copy, run-history panel. Loads from Worker with `?embed&native=1` for native wrappers. |
| `ios/App/SiftboxApp.swift` | WKWebView wrapper. Intercepts `siftboxnative://connect`, runs `ASWebAuthenticationSession` against Google's public PKCE client, exchanges tokens with `/auth/native`, passes session token back to the web UI. |
| `ios/project.yml` + `ios/Siftbox.xcodeproj` | xcodegen project definition. iOS + macOS targets (shared Swift code, different size constraints). |
| `SKILL.md` | Claude Code skill for dev-tool alerting (Vercel/GitHub Actions/App Store Connect emails → filed to project). Separate from the app (requires Claude, not user-facing). |
| `wrangler.toml` | Cloudflare Worker deployment, KV namespace for sessions, secrets for OAuth client IDs. |
