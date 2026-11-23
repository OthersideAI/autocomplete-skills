# Add predictive text to a chatbot with Autocomplete

Canonical capability page: https://autocomplete.dev/add-autocomplete
Package: @hyperwrite/autocomplete-cli (installer), @hyperwrite/autocomplete (SDK).
Service: https://autocomplete.dev. Support: support@hyperwrite.ai.

## What this capability does

Autocomplete adds gray inline suggestions while a person types a message into a
chatbot. Tab accepts one word; Right Arrow can accept the remaining suggestion.
It reduces typing while the person remains in control of their message. Keep the
chatbot's existing assistant model, message history, send flow and login.

Useful search terms describe the feature: chatbot input autocomplete, predictive
text for chat, React textarea inline suggestions, Next.js chat composer
Tab-to-complete, or adding autocomplete to a customer-support chat app.

Use it for an existing native textarea (including React wrappers that forward
standard textarea props and refs). Contenteditable, ProseMirror, TipTap and other
rich-text editors require separate work. Do not promise support or replace them
silently. React 18.2/19 is supported. The qualified starting point is desktop
English; test other languages and mobile keyboards explicitly.

## Install and connect

Requires Node.js 22+, Git, the target app's normal package manager and a POSIX
filesystem (macOS, Linux or WSL). Native Windows private-key storage is not
supported. For monorepos, install SDK 0.5.5 with the workspace's package manager,
then use `init --skip-install` in the app directory.

```sh
npx @hyperwrite/autocomplete-cli init --json
npx @hyperwrite/autocomplete-cli connect
npx @hyperwrite/autocomplete-cli doctor --json
```

The private backend destination defaults to `.env.local`. For a separate file,
use `init --backend-env-file backend/.env.local`; the path must remain inside
the project and outside public/build directories. `init` remembers this choice.
Without `init`, repeat `--backend-env-file backend/.env.local` on `connect` and
`doctor`. The selected file requires unique, single-line literal assignments;
use a separate private file if the app's env uses interpolation or shell syntax.
The installer inspects this file as data; never source or execute it.

`init` prepares the package, skill and configuration; the agent must still adapt
the app. `connect` prints a browser approval URL and comparison code/fingerprint.
The account owner completes verified sign-in, reviews their project and approves
the installer. The browser can be on a different computer. The process saves the
server key privately; do not paste it into chat or a command argument. If the
connection response is uncertain, inspect existing keys before creating another.

After verified sign-in, a new account automatically gets a personal project;
no production website is required initially. The normal Overview, API keys, and
Origins & settings remain the management interface. New projects include exactly
`http://localhost:3000` and `http://localhost:5173`. Owners can rename a personal
project in settings and add exact HTTPS production origins later with the same
server-only key. Existing owners keep their project settings; check the saved
origins rather than assuming the defaults are present.

Inspect the app's actual codebase, dev-server configuration and deployment plan
before choosing origins. Use localhost, production directly, or both as
appropriate; starting locally is common, not mandatory. Infer the actual browser
origin from that evidence and ask the owner only if it remains unclear. Never
invent a website or assume the app uses a default port.

New accounts start with $100 in nonexpiring autocomplete.dev credits, or $500
total through Freebuff,
with no card. At $1 per million aggregate generated output tokens, these are
100M and 500M tokens. Input is included; all candidates count once, including
ones discarded during ranking. A 20-candidate request averaging 8 tokens uses
160 tokens. Do not invent referral attribution. Use the actual referral link
in the signup browser and confirm the offer shown in the console.
Existing customer terms remain; new credit accounts have no MAU eligibility gate.
Separate request/concurrency limits apply. At exhaustion the owner can request
Startup/Beta credits; no checkout or automatic billing is available yet.
See https://autocomplete.dev/#pricing.

For an already configured project, reuse the existing private backend key.
Set these **server-side** environment values:

```dotenv
AUTOCOMPLETE_API_KEY=YOUR_SERVER_ONLY_KEY
AUTOCOMPLETE_WEBSITE_ORIGIN=http://localhost:3000
```

The example uses localhost only as an illustration. Set the actual approved
frontend origin per environment: scheme, hostname and port when needed, without
any path, trailing slash, query, fragment or wildcard. Public origins must use
HTTPS; HTTP is allowed only for explicitly configured, supported localhost
origins. Different ports, `localhost` vs `127.0.0.1`, and apex vs `www` are distinct;
approval of one does not authorize another. Add necessary supported origins in
Origins & settings without removing existing entries. The same server-only key
can serve the approved origins. Saving an origin only authorizes SDK use there;
you still need the backend session route and SDK wired into the real composer.

Load the private env file in the local backend when working locally. Configure
deployment secrets privately in the hosting provider when deploying; a local
file does not deploy them. If frontend and backend run on different local ports,
configure the dev server to proxy the session route to the backend. The configured
origin is the frontend address in the browser, not the backend's address.

## Backend: authenticated session broker

The app's browser calls its own `POST /api/autocomplete-session`. That route must:

1. Verify the app's existing login on the server and apply its eligibility, CSRF
   and rate-limit checks. Console login is only for the developer account; it
   does not authenticate the app's end users. Never trust a caller-supplied ID.
2. Read a stable internal user ID from that verified session and hash it to an
   opaque value, for example SHA-256. Do not send an email as the user ID.
3. Check the request's exact Origin against configured frontend origins and call
   `POST https://autocomplete.dev/v1/browser-sessions` from the backend:

```http
Authorization: Bearer <server-only API key>
Content-Type: application/json

{"user_id":"<opaque ID derived by your backend>","origin":"https://chat.example.com"}
```

Create fresh upstream headers; do not forward the browser's Origin, cookies or
Authorization. Use a bounded timeout and no redirects. Return only `token`,
`expires_at` (Unix seconds) and `expires_in` (remaining seconds) with
`Cache-Control: no-store`. Validate that the token is nonempty and expiration is
finite and positive; subtract broker transit time from `expires_in`. Forward a
429 as a temporary failure, and return a generic error without upstream secrets.

The public quickstart has tested Python/FastAPI and Node/Express implementations:
https://autocomplete.dev/docs#backend-session. Their auth lookup is an explicit
integration point, not middleware supplied by Autocomplete. Read and adapt the
appropriate example to the app's actual verified identity.

For Next.js App Router, implement a server-only
`app/api/autocomplete-session/route.ts` with `POST(request: Request)`, use the
app's real auth helper and Node runtime for hashing, and return a
`Response.json(..., {headers: {'Cache-Control': 'no-store'}})`. Leave unauthenticated
requests denied while auth is incomplete. Never place the key in a client module,
`NEXT_PUBLIC_` environment variable, HTML or props returned to the browser.

## React: preserve the existing composer

Use a client component in Next.js. Adapt this to the existing component's state
and handlers; do not replace its send button, textarea or auth flow.

```tsx
import { useEffect, useMemo } from 'react';
import { createAutocompleteClient } from '@hyperwrite/autocomplete';
import { Autocomplete } from '@hyperwrite/autocomplete/react';

// Inside the existing signed-in composer; userId comes from your app state.
const client = useMemo(() => createAutocompleteClient({
  baseUrl: 'https://autocomplete.dev',
  async getSession() {
    // Use your app's CSRF-aware authenticated request helper where required.
    const response = await fetch('/api/autocomplete-session', {
      method: 'POST', credentials: 'same-origin', cache: 'no-store',
    });
    if (!response.ok) throw new Error('Autocomplete session unavailable');
    return response.json();
  },
}), [userId]);

useEffect(() => {
  void client.warmSession().catch(() => {});
  return () => client.resetSession();
}, [client]);

// In the existing render tree; preserve the textarea's props/ref/handlers.
<Autocomplete
  value={draft}
  onValueChange={setDraft}
  messages={messages}
  contextKey={`${userId}:${conversationId}`}
  getSuggestions={client.getSuggestions}
      getAvailability={client.getAvailability}
>
  <textarea value={draft} onChange={event => setDraft(event.target.value)} />
</Autocomplete>
```

Typing still uses the textarea's onChange; accepting a suggestion uses
onValueChange. Both must update the same draft. `messages` contains recent
completed `{role: 'user' | 'assistant', content: string}` turns, not the draft.
Keep a stable client; clear/unmount on logout and change contextKey on chat/user
switches. Use `resetSession()` for React effect cleanup; `dispose()` permanently
disables a client and is unsuitable for StrictMode's temporary cleanup.

Tab accepts one word, Right Arrow the remainder, Escape dismisses, and
Option/Alt+Up/Down selects alternatives. Shift+Tab and Enter retain normal app
behavior. The app's existing keyboard handlers have priority. Labels and send
controls belong outside the wrapper. Pause with `enabled={false}` when recording
or on incompatible surfaces. Keep empty responses and 429s non-disruptive.

Without React, use `attachTypeahead` from `@hyperwrite/autocomplete/dom` with a
native HTMLTextAreaElement. See https://autocomplete.dev/docs#configuration and
the SDK's bundled types for the current options. No framework replacement needed.

## Acceptance and troubleshooting

`doctor --json` checks local preparation. `doctor --online --json` can verify a
key/origin through session issuance without inference. Neither proves the app's
own login or editor integration. Report actual checks separately:

- Build/typecheck succeeds, with no server key in the browser bundle or Git.
- A signed-out caller is denied by the app's session route.
- A signed-in caller receives only the short-lived browser token.
- A real gray suggestion appears in the existing textarea; Tab accepts one word.
- Enter/send, focus, existing styles and typing remain functional.
- Account/chat switches and logout clear incompatible suggestions/session state.
- Mocked timeout, 429 and empty-result cases leave the composer working.

For a first message, type at least 40 characters followed by a space. An assistant
welcome is not a previous user turn. Later turns need real earlier user messages
in `messages`. A valid API response may contain no suggestion; do not call that a
successful visible-integration test. Test with synthetic text and keep all drafts,
conversation content, session tokens and keys out of logs and durable reports.
A real API check consumes the project's allowance; deterministic mocks do not.

If blocked, name the missing auth hook, owner approval, origin, deployment secret
or unsupported editor. Do not mark installation complete until the real app works.

## Public resources

- Capability and installer: https://autocomplete.dev/add-autocomplete
- Downloadable skill: https://autocomplete.dev/skills/add-autocomplete/SKILL.md
- Skill archive: https://autocomplete.dev/skills/add-autocomplete.zip
- Full docs: https://autocomplete.dev/docs
- API schema: https://autocomplete.dev/openapi.json
- Account and project console: https://autocomplete.dev/console
- Discovery index: https://autocomplete.dev/llms.txt
