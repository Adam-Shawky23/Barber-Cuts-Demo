# Setup and deployment guide

A web app for the shop. A customer takes a selfie (or uploads one), picks a style
(Bald, Rasta / Dreads, Skin fade, Buzz cut, Afro, Mohawk, Braids, Pompadour) or types
their own, and the AI shows them with that haircut. They can save favourites to a
lookbook and show the barber. Nothing is saved: no accounts, no database, photos are not stored.

The AI part is free. It uses a Hugging Face "Space" (a free hosted AI model).

## 1. Run it on your computer

You need [Node.js](https://nodejs.org) (version 22.12 or newer).

1. Open a terminal in this folder.
2. `npm install`
3. Copy `.env.example` to a new file called `.env` (this holds your settings).
4. `npm run dev`
5. Open the address it prints (usually http://localhost:5173).

To run the finished version instead: `npm run build` then `npm start`, and open
http://localhost:8787.

## 2. Settings (`.env`)

| Setting | What it is |
| --- | --- |
| `HF_SPACE` | Which AI Space to use. Default: `black-forest-labs/FLUX.1-Kontext-Dev` |
| `HF_ENDPOINT` | Which function of that Space to call. Default: `/infer` |
| `HF_TOKEN` | Your free Hugging Face token. **Required for the online (Render) site**; optional only for a quick try on your own computer |
| `PORT` | Port for the server. Default 8787 |
| `HF_FALLBACK_SPACES` | Optional. Other Hugging Face Spaces to try (see "Backup AI services" below) |
| `CF_ACCOUNT_ID`, `CF_API_TOKEN` | Optional, **free**. Cloudflare Workers AI, used when Hugging Face is busy or out of free quota |
| `FAL_KEY` | Optional, paid. fal.ai key, used after Cloudflare |
| `GEMINI_API_KEY` | Optional, paid. Google Gemini key, used after fal.ai |

The token stays on the server. It is never sent to the browser or written to logs.

### Get a free Hugging Face token

1. Create a free account at https://huggingface.co.
2. Go to Settings, then Access Tokens.
3. Click "Create new token", choose type **Read**, give it any name, and create it.
4. Copy the token into `.env` as `HF_TOKEN=hf_...`

Without a token the free allowance runs out after a couple of tries, and customers will
see "The AI is busy right now. Try again in a minute." With a token you get a larger
daily allowance. Even then the free tier can sometimes be busy; waiting a minute usually fixes it.

**For the online site a token is required.** The anonymous allowance is counted per internet
address, and Render's servers share their addresses with many other apps, so without a token
the online site will almost always say the AI is busy. Skipping the token is only fine for a
quick try on your own computer.

### Fair-use limit

To stop one phone from using up the free AI allowance, the server allows at most 2 looks
being made at the same time and 10 new looks per minute **per internet connection**. Everyone
on the shop's Wi-Fi shares one internet connection, so the whole shop shares this limit. If it is
reached, customers see the "busy" message and can try again a moment later.

## 3. Put it online (Render, free)

Customers need a real web address, and the phone camera only works on **HTTPS**
(a plain `http://` address on your Wi-Fi will not open the camera, though uploading a photo still works).
Render gives you HTTPS automatically.

1. Put this project on GitHub.
2. On https://render.com, choose New, then Web Service, and connect the GitHub repo.
3. Build command: `npm ci --include=dev && npm run build`
4. Start command: `npm start`
5. Plan: Free.
6. Under Environment, add `NODE_ENV` = `production` and `HF_TOKEN` (required, see above),
   plus `HF_SPACE` / `HF_ENDPOINT` only if you change them.
7. Deploy, then open the address Render gives you on your phone to test the camera.

The free Render plan goes to sleep when unused, so the first visit can take about a minute to load.

### Put it on customers' phones like an app

The site works in Safari (iPhone/iPad) and Chrome (Android). It can also be added to the home
screen, where it opens full screen with its own icon:

- **iPhone / iPad:** open the site in Safari, tap Share, then "Add to Home Screen".
- **Android:** open the site in Chrome, tap the ⋮ menu, then "Add to Home screen" (or "Install app").

The phone's back button and swipe-back move between screens without losing the looks.

## 4. Backup AI services (more looks when the free quota runs out)

When the Hugging Face Space says it's busy or the free daily allowance is used up, the server
automatically tries the next service in this order:

1. The main Space (`HF_SPACE`)
2. Any extra Spaces in `HF_FALLBACK_SPACES`
3. **Cloudflare Workers AI (free)** if `CF_ACCOUNT_ID` and `CF_API_TOKEN` are set: FLUX.2 [klein],
   from the same company as the main model. The free allowance is 10,000 "neurons" a day, roughly 90 looks (estimated from
   Cloudflare's price list),
   and it refills every day at midnight UTC. No card needed.
4. **fal.ai (paid)** if `FAL_KEY` is set: the same FLUX Kontext model as the main Space. Paid per image.
5. **Google Gemini (paid)** if `GEMINI_API_KEY` is set. Gemini's image model is paid in the API
   (its free period has ended). Looks may differ a bit from FLUX.

Add only the keys you want; with none, it works exactly as before. To stay free, set just the
Cloudflare settings. A service that says it's busy is skipped for 5 minutes, so later customers go
straight to the backup instead of waiting. When the server starts it prints the order it will use,
e.g. `AI services, in order: hf:... → cloudflare`. On Render, add the keys under Environment.

- **Cloudflare (free):** create a free account at https://dash.cloudflare.com. Your Account ID is on the
  Workers & Pages page. For the token: My Profile, then API Tokens, then Create Token, and use the
  "Workers AI" template. Put them in `CF_ACCOUNT_ID` and `CF_API_TOKEN`.
- **fal.ai key:** sign up at https://fal.ai, add credit, then Dashboard, then Keys. Check their pricing page for the current price per image.
- **Gemini key:** sign in at https://aistudio.google.com/apikey and create a key. Image editing is billed, so add billing in Google AI Studio.
- **Extra Spaces** use your `HF_TOKEN` too. Another free (ZeroGPU) Space shares the same daily
  allowance as the main one, so it only helps when one Space is overloaded or down, not when the
  allowance is used up. Format: `owner/space|/endpoint|image_param`, comma-separated.

All keys stay on the server, like the Hugging Face token. The selfie is sent to whichever service
makes the look.

## 5. Switching to a different or paid AI

Change `HF_SPACE` (and `HF_ENDPOINT` if that Space uses a different function name) in `.env`
or in Render's Environment settings, then restart. For a paid or private Space, use a token
from the account that owns or pays for it. No code changes are needed as long as the Space
takes an input image and a text prompt.

### About the model choice

`FLUX.1-Kontext-Dev` was only lightly tested: 1 portrait, 2 styles (bald, rasta), and both kept the
face, glasses, clothes and background well. Skin fade and other portraits were not tested.
To test a model or style yourself:

`npx tsx scripts/try-model.ts <space> <endpoint> <photo.jpg> "<style phrase>"`

(`HF_TOKEN` is read from the environment; `IMAGE_PARAM=image` changes the image field name for Spaces that use it.)

## Developer notes

- `npm test` runs the tests, `npm run build` builds the site into `dist/`.
- Server: `server/`, client: `src/`. Image data is never logged.
