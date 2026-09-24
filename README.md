# Sage

Sage holds a spoken mock job interview with you, listens to your answers, asks real follow-up
questions, and gives you back a scored report on how you did.

You give it a role, a level and your resume. It runs the interview out loud in the browser, adapts
its questions to what you actually said, and writes the report to your account when the call ends.

**Live at [sageinterview.co](https://www.sageinterview.co).** One interview needs no account.

## How it works

- **Planning and scoring run on Claude; the live talking runs on Vapi.** `claude-sonnet-4-6` splits
  the role into three topic areas before the call and writes the scored report after it. The
  conversational turn itself is `gpt-4.1` inside Vapi's WebRTC bridge, because that path is
  latency-bound and Vapi owns the audio loop.
- **Next.js 14 and TypeScript on Vercel, with Clerk for accounts and Supabase for storage.**
  Completed sessions and their evaluations sit behind Postgres row-level security, and every policy
  scopes rows to the user who owns them.
- **No key and no prompt reaches the browser.** The client fetches a 120-second HS256 JWT scoped to
  one capability and locked to the request origin, instead of holding a long-lived Vapi key. The
  interview prompt is assembled server-side and the browser never sees it.

<!-- TODO: GIF here - a 20 second clip of a live interview, question through spoken answer. -->
<!-- TODO: screenshot here - the scored report screen at the end of a session. -->

## Run it

Requires Node 20 or newer, pnpm, and accounts for Clerk, Supabase, Vapi, Anthropic and OpenAI.
From a clean clone:

```bash
pnpm install
pnpm test                    # 16 tests across 3 suites, no keys needed
cp .env.example .env.local   # fill in the keys listed there
pnpm dev                     # https://localhost:3000
```

`pnpm test` runs against mocked model responses, so it passes before any key exists. `pnpm dev`
serves over HTTPS (`next dev --experimental-https`) because `getUserMedia` needs a secure context.
Apply the migrations in `supabase/migrations/` to a fresh Supabase project before the first run.

## How a session runs

```
resume + role + level
        │
        ▼
POST /api/realtime/initialize        claude-sonnet-4-6
        │                            splits the role into exactly 3 topic areas
        ▼
POST /api/realtime/vapi-token        HS256 JWT, 120s TTL, origin-locked
        │
        ▼
POST /api/realtime/start-call        builds the system prompt server-side,
        │                            creates a per-interview Vapi assistant
        ▼
   live conversation                 gpt-4.1 over Vapi's WebRTC bridge
        │                            Deepgram transcription w/ keyterm boosting
        ▼
POST /api/realtime/conclude          claude-sonnet-4-6
                                     → scored report, persisted to Supabase
```

Planning and scoring run on Claude via the Anthropic SDK. The live conversational turn runs on
`gpt-4.1` inside Vapi, because that path is latency-bound and Vapi owns the audio loop.

## Production decisions worth reading

**The Vapi key never reaches the browser.** The Web SDK originally needed a long-lived public key
shipped to the client. It now calls `/api/realtime/vapi-token`, which mints a 120-second HS256 JWT
scoped to a single capability, web call creation, locked to the request origin, with
`allowTransientAssistant` disabled. A leaked token is useless within two minutes and can't be used
to create arbitrary assistants.

**The interview prompt is built server-side and never ships to the client.** All prompt
construction lives in `lib/agent/buildVapiSystemPrompt.ts`, called only from
`/api/realtime/start-call`. The browser sends a plan and gets back an assistant id; it never sees
the instructions, so they can't be read or edited from the client.

**Transcription is primed per interview.** Deepgram consistently misheard resume- and
JD-specific proper nouns: company names, frameworks, tools it had never encountered. Each session
now extracts named entities from the interview plan and passes them as Deepgram
[keyterms](https://developers.deepgram.com/docs/keyterm), so the transcriber is biased toward the
vocabulary that session will actually contain. Extraction filters sentence-initial verbs and
stopword-adjacent noise so real proper nouns aren't crowded out of the keyterm budget.

**Cold start was measured, then cut.** Each interview needs the base assistant's non-model config
(voice, transcriber, analysis plan). That was a blocking round trip to the Vapi API on every start;
it's now held in a module-level cache, and the token fetch runs in parallel with it.

**Auth and limits.** Clerk guards the app; the `realtime/*` routes are deliberately open so an
anonymous visitor can take one interview without an account. Those routes are rate limited at 10
requests/minute, keyed by Clerk user id when present and by IP otherwise.

**Persistence.** Completed sessions and their evaluations are written to Supabase behind row-level
security. Every policy scopes rows to their owning user.

## Stack

Next.js 14 (App Router) · TypeScript · Tailwind · Clerk · Supabase · Vapi (WebRTC) ·
Anthropic SDK (`claude-sonnet-4-6`, `claude-haiku-4-5`) · OpenAI (`gpt-4.1` via Vapi, `whisper-1`,
`tts-1`) · Jest · deployed on Vercel

## Tests

```bash
pnpm test
```

Jest covers the two LLM-backed routes that parse model output, `/api/realtime/initialize` and
`/api/realtime/conclude`, plus `extractJson`, the tolerant parser both depend on. Model output is
the least trustworthy input in the system, so that's where the tests are.

## Current state

Honest notes, so nothing here is a surprise:

- **`src/app/page.tsx` is 2,189 lines.** The landing page, the live session UI, and the report
  screen all render from it. It's the largest piece of debt in the repo, and it's under a one-way
  ratchet: the contributor rules in `CLAUDE.md` cap new components at ~150 lines and require that
  `page.tsx` only ever shrink. New UI goes to `src/components/interview/`, which is where
  `InterviewForm`, `PreInterviewInstructionsModal`, and `SaveStatusToast` have already been pulled.
- **`/api/realtime/evaluate` is built but not wired up.** It scores a single exchange with
  `claude-haiku-4-5` and exists to enable mid-call adaptation. Nothing calls it yet, so
  `ExchangeRecord.quality` is a placeholder until `conclude` re-derives real scores after the call.
- **`useSpeech` is a legacy hook from the pre-Vapi architecture.** Its transcript loop is dead, but
  four of its return values are still load-bearing in `page.tsx`, so it can't be deleted cleanly yet.
  `/api/stt` and `/api/tts` belong to that older path.
- **The rate limiter is in-memory**, which means it's per-instance on serverless rather than a true
  distributed limit. Fine for current traffic, wrong shape for real scale.

