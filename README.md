# Alkeyya

The Alkeyya marketing site, migrated from static HTML/CSS to Next.js 15 (App
Router) with TypeScript. The design is a verbatim port: `app/globals.css` is the
original `styles.css` byte-for-byte, and every page renders pixel-identical
markup.

The original static site is preserved untouched in `legacy/`.

## Setup

```bash
npm install
cp .env.example .env.local   # then fill in the real values
npm run dev                  # http://localhost:3000
```

Production:

```bash
npm run build
npm run start
```

`npm run typecheck` runs `tsc --noEmit`.

## Environment variables

All of these are **server-side only** — none are prefixed `NEXT_PUBLIC_` except
the site URL, so the Brevo key is never sent to the browser.

| Variable | Required | Purpose |
| --- | --- | --- |
| `BREVO_API_KEY` | yes | Brevo API key. Dashboard → **SMTP & API** → **API Keys** → *Generate a new API key*. |
| `BREVO_SENDER_EMAIL` | yes | The `from` address. Must be a verified sender or a verified domain in Brevo, or the send is rejected with a 400. |
| `BREVO_SENDER_NAME` | no | Display name on the notification. Defaults to `Alkeyya Website`. |
| `CONTACT_RECIPIENT_EMAIL` | yes | Where submissions are delivered. |
| `NEXT_PUBLIC_SITE_URL` | no | Base for canonical/OG URLs. Defaults to `https://alkeyya.com`. |
| `BREVO_API_URL` | no | Test-only override pointing the route at a stub. Leave unset in production. |

Put them in `.env.local` for local work (already gitignored), and in your host's
environment settings for production. Never commit the real key.

## Testing the contact form

**Against a local stub** — exercises the whole route without spending a send:

```bash
# 1. a throwaway server that stands in for Brevo
cat > /tmp/brevo-stub.cjs <<'EOF'
require('http').createServer((req, res) => {
  let b = ''; req.on('data', c => b += c);
  req.on('end', () => { console.log('RECEIVED:\n', b); res.writeHead(201, {'content-type':'application/json'}); res.end('{}'); });
}).listen(4000, () => console.log('stub on :4000'));
EOF
node /tmp/brevo-stub.cjs

# 2. in another terminal, point the app at it
BREVO_API_KEY=test \
BREVO_SENDER_EMAIL=no-reply@alkeyya.com \
CONTACT_RECIPIENT_EMAIL=info@alkeyya.com \
BREVO_API_URL=http://localhost:4000/v3/smtp/email \
npm run dev
```

Then submit the form at http://localhost:3000/contact — the stub prints the exact
payload that would have gone to Brevo.

**Against Brevo for real** — put the real values in `.env.local`, drop
`BREVO_API_URL`, run `npm run dev`, and submit the form. The email lands at
`CONTACT_RECIPIENT_EMAIL`; replying to it goes straight back to the enquirer.
Failures are logged server-side with Brevo's response body.

You can also hit the route directly:

```bash
curl -X POST http://localhost:3000/api/contact \
  -H 'content-type: application/json' \
  -d '{"name":"Test","email":"test@example.com","service":"Website Development","message":"Hello"}'
```

Expected: `{"ok":true}` on success, `400` with a field message on a validation
failure, `502` if Brevo rejects or is unreachable.

## Notes

- Old `.html` URLs (`/index.html`, `/services.html`, `/contact.html`) are
  permanently redirected to the new routes in `next.config.ts`, so existing
  inbound links and search rankings survive.
- The contact form carries a hidden honeypot field (`company`). Submissions with
  it filled are answered `200 {"ok":true}` and silently dropped, so a bot has
  nothing to tune against.
