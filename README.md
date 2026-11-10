# Roshan Rozgar — روشن روزگار

**Hunar Ko Rozgar, Rozgar Ko Roshan Mustaqbil.**

Roshan Rozgar is a hyper-local marketplace that connects households in Islamabad and Rawalpindi with nearby skilled workers: plumbers, electricians, AC technicians, carpenters, painters, mechanics, appliance repairers and general handymen. It was built for the CUI Tech Fest **AI WebForge** hackathon.

Both sides use the same flow:

| Outcome | How the app does it |
|---|---|
| **1. Onboarding without forms** | Workers sign up through an 11-step wizard that uses voice and pictures. They say their name aloud (speech-to-text, with AI extraction of the name in Latin and Urdu script). The phone number is entered on a big keypad. Trade, skills, area, experience, price and days are chosen by tapping large illustrated cards. Every step can be read aloud, and there is a typed fallback whenever voice fails. Customers sign up with their phone number and name. |
| **2. Discovery** | Workers who are available are listed first, sorted by distance and then rating. Filters cover service, distance, "available now only" and minimum rating. Free text and voice search are supported, and "pankha nahi chal raha" matches Electrician. There is a list/map toggle. Busy and off workers are visibly distinct. |
| **3. Communication & price** | Every booking has a chat with voice notes (recorded audio plus transcript), quick replies in both languages, and a "listen" button on incoming messages. Price offers use a +/- stepper: accept or counter, and the earlier offer is marked as replaced. The agreed price is stored on the booking. |
| **4. Booking with consistent state** | Confirming a booking flips the worker from Available to Busy atomically. A partial unique index in SQLite guarantees that a worker can have only one active booking. If two customers confirm at the same moment, one succeeds and the other gets a clear "just booked by someone else" screen. Busy workers cannot set themselves Available while on a booking. Marking the job done frees them. |
| **5. Trust** | After the customer confirms completion they can leave a review: stars, quality, professionalism and timeliness, tags, written text, an optional voice review and an optional photo. Reviews are only possible on completed bookings, one per booking. Ratings and job counts are always computed from the database, so a new review changes the profile immediately. Profiles state plainly what is and isn't verified. |

---

## Quick start

Requirements: **Node.js 18.18 or newer** (20 LTS recommended). No database server and no native build are needed.

```bash
npm install
npm run dev            # http://localhost:3000
```

The SQLite database is created and filled with sample data the first time the server starts. It is stored at `./data/roshan.db`, and uploads go in `./data/uploads/`.

Production build:

```bash
npm run build
npm start
```

To reset to the clean sample data, stop the server first and then run:

```bash
npm run reset
```

### Try it quickly (judges)

1. Open `/start`, choose **I need a service**, and pick the sample account **Ayesha Khan**. No code is needed for sample accounts.
2. Open a second browser or a private window, go to `/start`, choose **Log in with phone** under workers, and pick **Muhammad Imran** (plumber).
3. As Ayesha:
   - Search "sink leaking".
   - Open Imran's profile and choose **Request service**.
   - Describe the problem by voice or text, then send.
4. As Imran:
   - The request appears on the dashboard and is announced aloud.
   - Choose **I can do it**, then **Offer price**.
5. As Ayesha: **Accept** the offer, then **Confirm booking**. Imran's dashboard switches to "On a booking — Busy".
6. As Imran: choose **Start work**, then **Work is done**.
7. As Ayesha: choose **Yes, the work is complete**, then **Rate this job**. The review appears on Imran's profile straight away.
8. To see the double-booking guard: Hamza Siddiqui already has a pending request to Imran. Agree a price on both jobs and confirm both. Only the first confirm succeeds. `npm run test:e2e` runs this race automatically.

To try worker sign-up, open `/worker/onboard`. One-time codes are shown on screen.

### Optional: turn on the AI services

Copy `.env.example` to `.env.local`, add **one** key and restart:

| Provider | Speech-to-text | Job understanding / name / skill | Photo hint | Text-to-speech |
|---|---|---|---|---|
| `GROQ_API_KEY` (free tier) | whisper-large-v3 | llama-3.3-70b-versatile | llama-4-scout-17b-16e-instruct | browser |
| `OPENAI_API_KEY` | whisper-1 | gpt-4o-mini | gpt-4o-mini | tts-1 |
| `GEMINI_API_KEY` | gemini-2.5-flash | gemini-2.5-flash | gemini-2.5-flash | browser |

With no key, the app still works end to end, and the UI labels the source of every suggestion. See "What is real, mocked or seeded" below.

### Tests

```bash
npm start                                 # or npm run dev, in another terminal
BASE=http://localhost:3000 npm run test:e2e
npm run reset                             # stop the server first; the test writes data
```

`scripts/e2e.mjs` makes 28 API-level checks across both journeys:

- discovery
- request
- offer and counter-offer
- the concurrent-confirm race
- worker status lock
- address and phone privacy before booking
- the job state machine
- review rules and aggregation
- worker onboarding

The UI flows were also checked with Playwright: worker onboarding in Urdu on a phone viewport, and the full customer journey on desktop.

---

## Tech stack

- **Next.js 14 (App Router), React 18.** Plain JavaScript with no UI framework. The CSS design system is ported from our Figma file and prototype.
- **SQLite via [sql.js](https://github.com/sql-js/sql.js)** (WebAssembly). There is no native compilation or ORM engine download, so `npm install` works everywhere. The database runs in memory and is written to `data/roshan.db` after every change. sql.js is synchronous, so each transaction runs without interleaving inside the Node process.
- **Browser APIs:**
  - Web Speech API (`SpeechRecognition` with `ur-PK` / `en-IN`, and `speechSynthesis`)
  - MediaRecorder for voice notes
  - Geolocation for "use my location", which picks the nearest sector
- **Fonts:** Plus Jakarta Sans and Noto Nastaliq Urdu (Google Fonts). The interface fully supports right-to-left Urdu.

### Project layout

```
app/                 pages (App Router) and app/api/* route handlers
components/          UI: Providers (state, i18n), ui.jsx, voice.js, Avatar, Icon, pages/*
lib/db.js            sql.js setup, schema (incl. one_active_booking_per_worker index), transactions
lib/server.js        sessions, queries, booking state machine, reviews
lib/ai.js            pluggable AI providers + fallbacks
lib/classify.js      offline keyword classifier (fallback, clearly labelled)
lib/seed.js          sample data (all flagged seeded=1)
scripts/e2e.mjs      API end-to-end test
```

### Booking state machine

```
requested ─(worker replies)→ negotiating ─(offer accepted)→ agreed ─(customer confirms; worker available→busy)→ confirmed
confirmed → on_way → in_progress → done (worker freed) ─(customer confirms)→ completed ─(review)→ reviewed
requested/negotiating/agreed → declined (worker) | cancelled (customer)
confirmed/on_way → cancelled (either side; worker freed)
```

---

## AI services used, and why

AI is used only where it removes a real barrier: literacy, language, typing, and knowing what is wrong.

1. **Speech-to-text (Urdu / Roman Urdu / English).** Used for worker onboarding (name, extra skills), job descriptions, chat voice notes and voice reviews.
   - Server-side: Whisper (Groq or OpenAI) or Gemini audio, when a key is set.
   - Otherwise: the browser's built-in speech recognition (Chrome/Edge, `ur-PK`).
2. **Job understanding (LLM).** Turns a spoken or typed description into:
   - a service category
   - a short title in English and Urdu
   - a summary, urgency and extracted details
   - up to two clarifying questions with tap-to-answer options
   - a safety tip, only for gas, sparking, flooding or shock risk

   The output is always shown as a suggestion and never as a diagnosis or price.
3. **Photo hint (vision model).** Describes what is visible in a problem photo and suggests a category, labelled as a guess.
4. **Name and skill extraction.** Turns "mera naam Waqas Ahmed hai" into `Waqas Ahmed` / `وقاص احمد`.
5. **Text-to-speech.** Reads out onboarding steps, job requests and incoming messages:
   - OpenAI TTS when that key is set
   - otherwise the browser's `speechSynthesis`, using an Urdu voice if the device has one, or else Roman Urdu read by an en-IN voice

Every AI output in the UI carries a tag showing where it came from: provider name, "Browser speech recognition", or "Keyword match (no AI key)".

### AI coding assistants used

- **Claude (Anthropic)**, through Claude's Cowork mode, was used heavily. It worked on:
  - the design system and Figma file
  - the HTML prototype
  - this application's code
  - the seed data
  - the test scripts
  - this README

  All code was reviewed and run by the team. The team made the product, scope and design decisions.

---

## What is real, mocked or seeded

**Real (works as described):**

- accounts and sessions (httpOnly cookie)
- discovery with distance
- requests
- chat with text, voice notes, photos and offers
- price agreement
- the booking lock (transaction plus unique index)
- the job state machine
- reviews and rating aggregation
- file uploads
- English/Urdu UI with RTL
- browser speech recognition and speech synthesis
- the AI provider integrations, when a key is supplied

**Mocked:**

- **One-time codes (OTP).** No SMS is sent. The code is generated and checked on the server, but it is shown on screen and labelled "Demo only". A real deployment would plug in an SMS gateway.
- **Sample account login.** `/start` offers one-tap login for 5 seeded accounts, for judges. This is limited to seeded accounts.

**Seeded sample data** (in `lib/seed.js`, every row flagged `seeded = 1`, and shown with a "Sample data" or "Sample profile" tag in the UI):

- 11 workers and 10 customers. All names and phone numbers (`03xx-555xxxx`) are invented.
- 20 past completed jobs with reviews, which give the initial ratings.
- One active booking, which is why Shahid Mehmood shows as Busy: a real booking row, not a hard-coded flag.
- One pending request from Hamza to Imran.
- Avatars are illustrations, not photos of real people.
- Landing-page testimonials are marked "Sample testimonial".

**Approximate:** locations are sector-centre coordinates with small offsets. Distances are straight-line estimates. The map view is a schematic diagram with no map tiles.

---

## Known limitations

- **Out of scope by design:**
  - No payments. Earnings shown to workers are informational: the sum of agreed prices.
  - No government ID (CNIC) verification. Profiles say so explicitly.
  - No native mobile app.
- **Browser support for voice:**
  - Without an AI key, voice input needs a browser with the Web Speech API (Chrome, Edge, Samsung Internet).
  - On Firefox and some Safari versions the mic is hidden or falls back to typing and tapping.
  - Browser recognition in Chrome sends audio to Google's service.
- **Urdu text-to-speech** depends on the device having an Urdu voice. Otherwise the Roman Urdu version is read out with an Indian-English voice, unless OpenAI TTS is configured.
- **AI providers:**
  - The OpenAI-compatible path (OpenAI and Groq) was tested against a mock server that mimics the API format.
  - The Gemini adapter follows the public REST format but was not tested with a live key during the hackathon.
  - AI output quality for Urdu depends on the provider and model.
- **Updates are polled** every 3–8 seconds; there are no WebSockets. There are no push or SMS notifications. A worker hears a spoken alert for new requests only while the dashboard is open.
- **Single server process:**
  - The in-memory SQLite and file persistence suit a demo or small pilot, not horizontal scaling. For production, swap `lib/db.js` for Postgres. The SQL is standard, including the partial unique index.
  - Stop the server before `npm run reset`; otherwise the running server writes its in-memory copy back.
- **No rate limiting or abuse protection** on OTP or uploads (8 MB limit, images and audio only).
- **Cancellation** has no penalty or dispute flow. "Report a problem" is not implemented.

---

## Team & licence

COMSATS University Islamabad — AI WebForge hackathon entry. Icons are adapted from [Lucide](https://lucide.dev) (ISC licence).
