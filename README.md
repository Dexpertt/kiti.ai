# Kiti AI — AI Image Generator

Kiti AI is a text-to-image generation platform: users sign up, get 5 free
credits, describe an image, and Kiti AI generates it with Google's Gemini
image model. Every generation costs 1 credit; credits and images are tied to
the user's account.

This project reuses the provided HTML/CSS/JS template as-is for its public
pages, and adds a real Node.js/Express backend plus four new pages
(dashboard, generate, history, account) built in the same visual style.

## What's real vs. what's stubbed

**Fully implemented and tested end-to-end:**
- Signup / login / logout with hashed passwords and secure sessions
- Exactly 5 free credits granted server-side on signup
- Image generation via Gemini, with atomic credit deduction, refund-on-failure,
  and duplicate-click protection
- Per-user image history with ownership checks (no IDOR)
- Zero-credit blocking (no Gemini call is made when a user is out of credits)
- Credit transaction ledger for auditing
- Protected pages that redirect unauthenticated visitors

**Architected but requires your credentials to go live:**
- **Payments.** `/api/checkout` creates a `pending` purchase row and returns a
  clear "not configured" response — it never fakes a successful payment or
  grants credits. `/api/webhooks/payment` verifies an HMAC signature and is
  idempotent, but it needs a real provider (Stripe, etc.) wired into
  `server/routes/billing.js` where the `-- Real provider integration goes
  here --` comment is. See "Payment setup" below.
- **Gemini.** The integration is complete and was tested against the "no API
  key configured" path (clean error, credit refunded). It has **not** been
  tested against a live Gemini key/quota, since none was provided.

## Requirements

- Node.js 18+ (uses the built-in `fetch` API)
- npm

## Installation

```bash
npm install
cp .env.example .env
```

## Environment variables

Edit `.env`:

| Variable | Required | Notes |
|---|---|---|
| `IMAGE_PROVIDER` | No | `auto` (default) — Gemini first, NVIDIA as fallback. Or `gemini` / `nvidia` to pin one |
| `GEMINI_API_KEY` | Yes, for generation to work | See "Gemini API key setup" |
| `GEMINI_IMAGE_MODEL` | No | Defaults to `gemini-2.5-flash-image` |
| `NVIDIA_API_KEY` | No | Strongly recommended as the fallback — see "NVIDIA NIM API key setup" |
| `NVIDIA_IMAGE_MODEL` | No | Defaults to `stabilityai/stable-diffusion-3-medium` |
| `PROVIDER_COOLDOWN_MS` | No | How long a failed provider is skipped. Defaults to `300000` (5 min) |
| `DATABASE_URL` | No | Defaults to a local SQLite file at `data/kitiai.db` |
| `SESSION_SECRET` | Yes | Any long random string — `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `FREE_SIGNUP_CREDITS` | No | Defaults to `5` |
| `CREDIT_PER_GENERATION` | No | Defaults to `1` |
| `PORT` | No | Defaults to `3000` |
| `PAYMENT_PROVIDER` / `PAYMENT_SECRET_KEY` / `PAYMENT_WEBHOOK_SECRET` | No | Leave blank until you have a provider — see "Payment setup" |

Never commit `.env` — it's already in `.gitignore`.

## Choosing an image provider

Kiti AI can generate images with either **Gemini** or **NVIDIA NIM**
(build.nvidia.com), controlled by one env var:

```
IMAGE_PROVIDER=auto     # default: Gemini, falling back to NVIDIA
IMAGE_PROVIDER=gemini   # Gemini only
IMAGE_PROVIDER=nvidia   # NVIDIA only
```

Both providers implement the exact same contract inside the app (atomic
credit check, refund-on-failure, ownership-checked storage) — switching is
just this one variable plus the matching API key below.

### Automatic fallback (why `auto` is the default)

Free-tier keys run out. A single Gemini key allows a limited number of image
generations per day, and once it's spent every request fails with a quota
error until midnight Pacific.

With `IMAGE_PROVIDER=auto` the server tries Gemini first and, if Gemini
returns a quota / rate-limit / rejection error, silently retries the same
prompt against NVIDIA and serves that instead. The `model` recorded against
each image in history is whichever provider actually served it, so you can
always see what came from where.

A provider that just failed is put in a **cooldown** for
`PROVIDER_COOLDOWN_MS` (default 5 minutes) and skipped entirely until then —
otherwise every request would burn a full round-trip re-confirming a key you
already know is dead.

The one error that does *not* trigger a fallback is a prompt blocked by the
safety filter: no other provider will accept it either, so retrying would
just waste the fallback's quota. `server/services/imageProvider.js` holds the
chain; `node scripts/test-fallback.js` verifies it with a stubbed network
call, so you can confirm the wiring without spending any quota.

### Gemini API key setup

1. Get an API key from Google AI Studio (https://aistudio.google.com/app/apikey).
2. Confirm your account/key has access to an image-generation-capable model.
   `GEMINI_IMAGE_MODEL` defaults to `gemini-2.5-flash-image`; check Google's
   current model list before relying on this, since availability changes.
3. Add the key to `.env`:
   ```
   IMAGE_PROVIDER=auto
   GEMINI_API_KEY=your_real_key_here
   ```

### NVIDIA NIM API key setup (free tier, no credit card)

1. Create a free account at https://build.nvidia.com and generate an API key
   from the NVIDIA Developer Program (a couple of minutes, no card needed).
2. Add it to `.env`:
   ```
   IMAGE_PROVIDER=auto
   NVIDIA_API_KEY=nvapi-your_real_key_here
   NVIDIA_IMAGE_MODEL=stabilityai/stable-diffusion-3-medium
   ```
3. The free tier has real rate/credit limits (NVIDIA's own docs describe a
   request-per-minute cap and a total credit allowance that both vary by
   model and change over time) — fine for personal/demo traffic, not for
   production volume. If *both* providers are exhausted, `/api/generate`
   returns a clean 429 "rate-limited" error and refunds the credit; it never
   silently fails.
4. You can swap `NVIDIA_IMAGE_MODEL` for any other image-generation model
   listed at https://build.nvidia.com/models (filter by "Image Generation")
   — as long as it exposes the same `ai.api.nvidia.com/v1/genai/<model>`
   shape. Some models use a different request schema; if you pick one that
   doesn't match, `server/services/nvidia.js` is where you'd adjust the
   request body.

Either way, the key is only ever read server-side
(`server/services/gemini.js` / `server/services/nvidia.js`) and is never sent
to the browser or embedded in any frontend file.

## Regenerating the site images

The template ships its content images as flat grey boxes stamped with their
dimensions — the files all exist, but every card on every page looks the same.
`npm run images` replaces all 43 of them with unique artwork at the exact
dimensions the CSS expects, using whichever providers you've configured:

```
npm run images                    # generate everything still missing
npm run images -- --dry-run       # list what would be generated, no quota spent
npm run images -- --only blog-img # redo just the ones matching a name
npm run images -- --force         # regenerate ones already done
```

Prompts live in `scripts/site-image-manifest.js`, separate from the generation
logic — edit a prompt there and re-run `--only <file>` to iterate on one image
without touching anything else.

Progress is recorded in `scripts/.generated-images.json`, so if you run out of
daily quota partway through, just re-run the same command later and it
continues where it stopped. `IMAGE_DELAY_MS` (default 2500) controls the pause
between requests — raise it if you see rate-limit errors.

## Database setup

No separate step needed — `server/db/index.js` creates the SQLite file and
runs `server/db/schema.sql` automatically on first start.

To reset the database during development:
```bash
rm -rf data/
```

## Development server

```bash
npm run dev
```

Visit http://localhost:3000. Static pages (home, pricing, about, etc.) are
public; `/dashboard.html`, `/generate.html`, `/history.html`, and
`/account.html` require login and redirect to the homepage's login modal
otherwise.

## Production build

There's no separate build step (the frontend is static HTML/CSS/JS). For
production:

```bash
npm install --omit=dev
NODE_ENV=production npm start
```

Set `NODE_ENV=production` so session cookies are marked `secure` (requires
serving over HTTPS — put this behind a reverse proxy like nginx or a platform
that terminates TLS for you).

## Production deployment notes

- Put the app behind HTTPS. `app.set('trust proxy', 1)` is already set for
  running behind a reverse proxy.
- SQLite is fine for a single-instance deployment. For multiple instances,
  swap `DATABASE_URL` to a Postgres connection and update `server/db/index.js`
  accordingly (the query patterns are plain SQL and port easily).
- Generated images are stored on local disk at `data/generated/<user_id>/`.
  For multi-instance or larger scale, swap `server/services/gemini.js`'s
  `fs.writeFileSync` calls for an S3-compatible object storage client and
  update `server/routes/images.js` to stream/redirect from there instead of
  `res.sendFile`.
- The `express-session` store here is `better-sqlite3-session-store`, backed
  by the same SQLite file. That's fine for one instance; for multiple
  instances behind a load balancer, move sessions to Redis.

## Deploying on Hostinger

Important reality check first: this project has a real backend (auth,
SQLite database, credit ledger, session cookies) that needs a **long-running
Node.js process** — it is not a static site. A plain "zip upload" only works
on hosting meant for static HTML/PHP; it will not run this server by itself.
Which Hostinger setup works depends on your plan:

**If you have Hostinger's Node.js-capable plan (Business/Cloud with hPanel):**
1. In hPanel, go to **Websites → your site → Advanced → Node.js**, click
   "Create Application".
2. Set **Application root** to the folder you'll upload the project into,
   **Application startup file** to `server/index.js`, and **Node.js version**
   to 18 or newer.
3. Upload the project (zip via File Manager, then extract — everything
   except `node_modules`, which you'll install on the server) into that
   application root.
4. In the same Node.js panel there's an **npm install** button (or open the
   built-in terminal and run `npm install --omit=dev`).
5. Add your environment variables (`GEMINI_API_KEY` or `NVIDIA_API_KEY`,
   `IMAGE_PROVIDER`, `SESSION_SECRET`, etc.) in the panel's **environment
   variables** section — don't rely on uploading `.env` itself.
6. Start/restart the application from the panel. Hostinger will proxy your
   domain to the Node process automatically.

**If you have Hostinger VPS:**
1. SSH in, install Node.js 18+ (`curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt install -y nodejs`).
2. Upload/extract the project (SFTP, or `git clone` if you push it to a repo).
3. `cd kiti-ai && npm install --omit=dev`, create `.env` with your real
   values (this is safe on a VPS since it's your own server).
4. Run it under a process manager so it survives reboots/crashes:
   `npm install -g pm2 && pm2 start server/index.js --name kiti-ai && pm2 save && pm2 startup`.
5. Put nginx in front as a reverse proxy to `localhost:3000` and get HTTPS
   with `certbot --nginx`. Set `NODE_ENV=production` in `.env` so session
   cookies are marked secure.

**If you only have Hostinger's basic shared hosting (no Node.js option):**
It genuinely can't run this backend. Two honest options: upgrade to a plan
above, or split the deployment — host the static frontend anywhere (even
Hostinger shared hosting) and run `server/` on a Node-friendly host (Render,
Railway, Fly.io all have free/cheap tiers made for exactly this) — then
point the frontend's API calls at that backend's URL. This is more setup
than a single zip upload, but it's the honest path if the shared plan can't
run Node.

### Will the images show up correctly once hosted?

Yes — **all the image/CSS/JS assets are already inside `public/assets/` in
this project**; nothing needs to be added separately. I ran an HTTP-level
check of every single asset reference across every page through the actual
running server and all of them load correctly. The only way images break
after hosting is if the deploy step doesn't upload the `public/assets/`
folder alongside the HTML, or the app isn't actually running as a Node
process (see above) — as long as you deploy the whole project folder and
the server process is running, the images will display exactly as they do
locally.

## Payment setup

No payment provider is wired up by default — no credentials were supplied,
and the project intentionally never fakes a successful charge.

To go live with a provider (example: Stripe):

1. Set `PAYMENT_PROVIDER=stripe`, `PAYMENT_SECRET_KEY=sk_...`, and a
   `PAYMENT_WEBHOOK_SECRET` in `.env`.
2. In `server/routes/billing.js`, inside `POST /checkout`, replace the
   `-- Real provider integration goes here --` block with a call to create a
   Checkout Session, passing `purchaseId` as `client_reference_id`, and
   return `{ url: session.url }`.
3. Point Stripe's webhook at `POST /api/webhooks/payment`, and adapt the
   signature verification in that route to Stripe's actual scheme (Stripe
   uses its own signed-header format, not the generic HMAC-over-raw-body
   used here as a placeholder).
4. Credits are granted only inside that webhook handler, only once per
   purchase (checked via `purchase.status !== 'pending'`), after the
   signature check passes.

Credit packs themselves live in `server/config/plans.js` — edit prices or add
packs there; nothing else hard-codes them.

## Testing this yourself

Quick manual checks worth re-running after any change:

```bash
# Signup grants exactly 5 credits
curl -c cookies.txt -X POST http://localhost:3000/api/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","email":"you@example.com","password":"password123","confirmPassword":"password123"}'

# Credits after a failed generation should be unchanged (no GEMINI_API_KEY set)
curl -b cookies.txt -X POST http://localhost:3000/api/generate \
  -H "Content-Type: application/json" -d '{"prompt":"a dragon"}'
curl -b cookies.txt http://localhost:3000/api/user/credits
```

See "What's real vs. what's stubbed" above for what was actually verified
during development.

## Troubleshooting

- **"Image generation is not configured on this server yet"** — set
  `GEMINI_API_KEY` in `.env` and restart.
- **"The configured Gemini API key was rejected"** — the key is invalid or
  lacks access; double check it in Google AI Studio.
- **"The configured model ... is not available for this account"** — your
  key doesn't have access to that model; check current model availability
  and update `GEMINI_IMAGE_MODEL`.
- **Login redirect loop** — make sure cookies aren't blocked and
  `SESSION_SECRET` is set (an unset secret still works but logs a warning and
  is insecure).
- **Styling looks broken** — confirm `public/assets/` was copied in full;
  the app serves it via `express.static`.
