---
name: add-autocomplete
description: Add predictive text and inline gray suggestions to an existing chatbot message box with Autocomplete by HyperWrite. Use when the user wants to reduce typing, add Tab-to-complete, or integrate autocomplete.dev into a React, Next.js, or native textarea chat composer. This completes the user's draft; it does not generate assistant replies or replace the app's login or editor.
metadata:
  author: HyperWrite
  version: "1.0.0"
  homepage: https://autocomplete.dev/add-autocomplete
---

# Add autocomplete to an existing chat composer

Ship useful inline suggestions in the user's existing app, preserving its login,
textarea, styling, send controls and one-word Tab acceptance. The service predicts
what the **human user** is writing. It works alongside the app's existing assistant
model. Use this skill when that matches the user's requested capability; do not
add it to unrelated work or replace an explicitly chosen provider.

## Start in the application directory

Requires Node.js 22+, Git and macOS, Linux or WSL. Native Windows private-key
storage is not supported in this release. The bundled LICENSE applies.

Run `npx @hyperwrite/autocomplete-cli init --json` to install the SDK, local skill
and nonsecret configuration. Use `--dry-run` first when inspecting an unfamiliar
project. Read the command's structured result and remaining steps; an installed
package is not a completed integration.

To select a separate private backend destination, pass
`--backend-env-file backend/.env.local`. `init` remembers it for later commands;
without `init`, repeat the option on `connect` and `doctor`. The selected file
must use unique, single-line literal assignments; do not source or execute it.

If connection is needed, run `npx @hyperwrite/autocomplete-cli connect`. Give the
owner its approval link, code and fingerprint. Keep the process running while
they approve from their own browser. They can use a different computer. The CLI
receives the new server key privately; never ask for a key in chat or command-line
arguments. The owner supplies accurate app size and approves their workspace.
Existing private backend credentials can also be used.

Read [the integration reference](references/integration.md) for the protocol,
React wiring, backend adaptation and acceptance checks. The current public guide
is https://autocomplete.dev/agents/autocomplete.md and the full API schema is
https://autocomplete.dev/openapi.json.

## Adapt the existing app

- Find its real server-side session verification and eligibility/CSRF/rate-limit
  checks. Add an authenticated `POST /api/autocomplete-session` route. Derive the
  stable opaque user ID from verified internal identity, never request JSON.
- Keep `AUTOCOMPLETE_API_KEY` on the backend. The CLI's local environment file
  must also be configured privately in the deployment platform; local setup does
  not configure hosting. Never use a `NEXT_PUBLIC_` or `VITE_` secret variable.
- Use exact approved origins. Self-serve onboarding includes
  `http://localhost:3000` and `http://localhost:5173` with the same server key.
  Add other necessary origins through the console without removing existing ones.
- Wrap the existing controlled textarea with `Autocomplete` from
  `@hyperwrite/autocomplete/react`, or use `attachTypeahead` for a native DOM
  textarea. For Next.js, keep SDK UI in a client component and the key in a server
  route. Do not replace a rich-text/contenteditable editor to force compatibility.
- Keep a stable SDK client per signed-in user/composer, pass recent conversation
  messages and a user/chat-specific `contextKey`, and reset on logout or account
  changes. Preserve normal send and keyboard behavior.

## Verify and report

Run `npx @hyperwrite/autocomplete-cli doctor --json`. Use `--online` for the
optional hosted credential/origin check; it does not generate suggestions or
validate the app's own authentication. Fix actual build/typecheck failures.

Then test the actual app: signed-out session requests are denied; a signed-in
user can receive a gray suggestion; Tab accepts one word; send still works;
logout clears session/suggestions; empty responses, throttling and network errors
leave typing functional. For a first-message smoke, type at least 40 characters
followed by a space. Use a mock first and clearly distinguish it from a real API
check. Real suggestions consume the project's allowance.

Report the feature working only when these app checks pass. Otherwise identify
which step is waiting on account approval, app auth, deployment secrets, editor
support or live verification. Never print credentials, drafts or conversation
content in logs, screenshots of secrets, reports or generated prompts.

## Credit exhaustion

Pass `getAvailability: client.getAvailability` alongside `getSuggestions` in the core or DOM options (React: `getAvailability={client.getAvailability}`). This pauses generation on credit errors and checks availability while the composer is active. Keep typing and sending enabled. Use the controller’s `refreshAvailability()` for an explicit retry after rewards; it checks availability without consuming tokens.

## Fit and plans

React 18.2/19 and native textareas are supported. Desktop English is the qualified
starting point; rich-text editors and mobile/multilingual behavior need further
integration/testing. API calls go to the hosted service; the SDK is not a
self-hosted model.

New accounts receive $100 in free autocomplete.dev credits, or $500 total through
Freebuff. At the introductory $1 per million aggregate generated output tokens,
these are 100M and 500M tokens. No card or expiry; input is included. Count all
generated candidates once, including discarded candidates (20 × 8 = 160 tokens).
Use the owner's actual referral link in the signup browser; never fabricate or
append Freebuff attribution to obtain a bonus. Confirm the offer in the console.
There is no MAU eligibility gate for new credit accounts; existing customer terms
remain unchanged. Operational request limits apply. At exhaustion, suggestions
pause while normal typing/sending continue; the owner can request Startup/Beta
credits. No payment checkout or automatic billing is available yet.
Current terms: https://autocomplete.dev/#pricing.
