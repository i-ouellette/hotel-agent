# 🏨 Hotel Spot

**One prompt in, one email out.** Type something like *"cheapest hotel in Toronto on Nov 2 for 2 people"*. Hotel Spot finds the cheapest hotels nearby, has an AI voice agent **phone the 3 cheapest at the same time**, and emails you the cheapest one that actually has a room. The email includes the price, the call transcript and the call recording.

The agent never books anything. It only asks about availability and price.

Built in one day by a team of 4 at the **TechTO "Fast & Frugal" hackathon** (Oct 2026). 📊 [Pitch deck](Hotel_Spot_Deck.pdf)

<!-- Optional: add a screenshot of the Streamlit UI or the result email here -->
<!-- ![Hotel Spot UI](docs/screenshot.png) -->

## How it works

```
prompt ─► parse (regex) ─► search (SerpApi) ─► 3 parallel calls (ElevenLabs + Twilio) ─► extract answers ─► one email
```

1. **Parse** the city, dates and guests with regex, using no LLM. Backboard is only a fallback for unusual prompts.
2. **Search** Google Hotels through SerpApi, filter to within 20 km of the city, and look up phone numbers in parallel.
3. **Call** the 3 cheapest hotels **at the same time**. The voice agent handles phone menus, hold music and voicemail.
4. **Extract** each answer, cheapest source first: the agent's own tool call, then a regex over the transcript, then an LLM only as a last resort.
5. **Email** the cheapest hotel with a room, along with its transcript and an MP3 of the call.

## Results

- **~2:45 end to end** on a real run (San Diego, 3 nights), from prompt to email in the inbox
- **about 1,900 LLM tokens (~$0.05)** for all 3 calls. Parsing, search, extraction and email use **0 tokens** on a normal run.

## My role

I owned the **results and email layer**:
- **`tool_server.py`**: a thread-safe endpoint that records the voice agent's live `record_availability` tool calls from 3 simultaneous calls
- **`extractor.py`**: a zero-token regex extractor that reads availability and spoken prices ("thirty-four a night") from transcripts when the tool call doesn't arrive
- **`email_service.py`**: builds the report email, validates it against the organizers' required format, attaches the call recording, and sends through Resend with a Gmail SMTP fallback

Teammates: Richard (search + UI), Zach (prompt parsing + LLM fallback), Jordan (voice agent).

**Stack:** Python · ElevenLabs · Twilio · SerpApi · Backboard · Resend · Streamlit · pytest

| Doc | Owner | Covers |
|---|---|---|
| [docs/architecture.md](docs/architecture.md) | Lead | System design, data flow, token strategy, corrections to the original plan |
| [docs/ws1-discovery.md](docs/ws1-discovery.md) | Richard (T1) | SerpApi search, phone resolution, one-shot UI |
| [docs/ws2-backboard.md](docs/ws2-backboard.md) | Zach (T2) | Backboard prompt parsing + call plans, fallback extraction, token telemetry |
| [docs/ws3-voice.md](docs/ws3-voice.md) | Jordan (T3) | ElevenLabs agent, 3 parallel outbound calls, IVR/hold/voicemail, transcripts |
| [docs/ws4-email.md](docs/ws4-email.md) | Isaac (T4) | Resend email (availability + transcripts), extraction, deck |
| [docs/integration-and-demo.md](docs/integration-and-demo.md) | Everyone | Phase timeline, integration order, test matrix, demo runbook, open questions |

## Repo layout

```
config.py          # all env settings, single source of truth
telemetry.py       # Run object: stage timers + token counters -> run_telemetry.json
search_service.py  # WS1
agent_brain.py     # WS2 (prompt parsing + Backboard fallback)
voice_agent.py     # WS3 (agent setup + call lifecycle)
extractor.py       # WS4 (zero-token availability extraction)
email_service.py   # WS4
pipeline.py        # orchestration: parallel call fan-out/join + timer
main.py            # CLI: python main.py --city "Toronto"
app.py             # Streamlit one-shot UI
tests/             # extractor + prompt parsing unit tests
.env.example
```
