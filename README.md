# Nova Suraksha Voice Agent — Complete Project Notes

Everything about this project, written so you can explain any part of it in an
interview: what it is, every technology and why it was chosen, every file and
what it does, how one conversation flows end to end, the hard problems and how
they were solved, and the real bugs that were found and fixed along the way.

---

## Contents

1. What this project is
2. Technologies and why each one
3. Every file, and what happens when you run it
4. One conversation, end to end
5. The hard parts
6. The business layer
7. Latency
8. Testing a voice agent without a microphone
9. Bugs found and fixed (and how each was diagnosed)
10. Interview questions, answered
11. Glossary

---

## 1. What this project is

**Priya** is a voice agent for a fictional hospital, Nova Suraksha
Multispeciality Hospital in Hyderabad. You talk to her through a microphone and
she talks back through your speakers. She can book, cancel and reschedule
appointments with eight doctors across seven departments, answer questions about
timings, fees and the address, route emergencies to the emergency line, and
refuse to give medical advice.

It runs entirely in a terminal. No web page, no phone line, no telephony.
And it is built **without any voice-agent framework** — no LiveKit, no Pipecat,
no Vapi. Every step of the pipeline is written by hand in Python so that every
step can be understood, measured and tuned.

### The pipeline

```text
 You speak
    │
    ▼
┌───────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────────┐
│  Mic  │──▶│ VAD │──▶│ STT │──▶│ LLM │──▶│ TTS │──▶│ Speaker │
└───────┘   └─────┘   └─────┘   └─────┘   └─────┘   └─────────┘
  20 ms      "are     speech    decide     text      you hear
  frames     they     to text   what to    to audio  the reply
             talking?"          say, call
                                tools
```

| Stage | What it does | Runs where |
| --- | --- | --- |
| Mic | Captures audio in 20 ms frames | Your laptop |
| VAD | Decides when you started and stopped talking | Your laptop (no network) |
| STT | Turns your speech into text | Sarvam cloud, streaming over WebSocket |
| LLM | Reads the text, decides what to say, calls booking functions | Sarvam cloud, streaming over HTTP |
| TTS | Turns the reply into speech | Sarvam cloud, streaming over WebSocket |
| Speaker | Plays audio the instant it arrives | Your laptop |

This is a **cascading pipeline**: separate models, one feeding the next. Every
slow stage adds directly to how long you wait for a reply, which is why latency
is the central engineering concern of the whole project.

The alternative is **speech-to-speech** — a single model that takes audio in and
produces audio out. It is faster because there are no handoffs, but there is no
text in the middle to inspect, and the text in the middle is exactly where the
booking rules, the tool calls and the "never invent a fee" guarantees live. For
a business agent, cascading is the right choice.

---

## 2. Technologies and why each one

### The three AI services — all from Sarvam AI

| Layer | Model | Protocol | Why this one |
| --- | --- | --- | --- |
| STT | `saaras:v3` | WebSocket, streaming | Indian-English and Indian languages; servers in India (low network latency); streams so transcription runs *while* you talk |
| LLM | `sarvam-105b-conversations` | HTTP streaming, OpenAI-compatible | Supports tool calling (needed for booking); streams tokens; the only Sarvam chat model currently offered |
| TTS | `bulbul:v3`, voice `priya` | WebSocket, streaming | Indian-English voice; audio streams back in chunks so playback starts before the sentence is finished |

One API key (`SARVAM_API_KEY`, in `.env`) covers all three.

The LLM speaks the **OpenAI API format**, so the standard `openai` Python
library is used as the client, pointed at Sarvam's server:

```python
client = AsyncOpenAI(api_key=config.API_KEY, base_url="https://api.sarvam.ai/v1")
```

### Python and libraries

| Library | Used for | Why |
| --- | --- | --- |
| Python 3.10 | Everything | |
| `asyncio` (standard library) | Running mic, STT, LLM and TTS at the same time on one thread | A voice agent is all network waiting — async fits it exactly, with no thread-safety headaches |
| `sounddevice` | Microphone in, speaker out | Thin wrapper over PortAudio; gives raw PCM frames via a callback |
| `webrtcvad-wheels` | Voice activity detection | Google's WebRTC VAD; tiny, instant, runs locally with no network |
| `numpy` | Frame loudness (RMS) | Already a dependency of sounddevice |
| `websockets` | The STT and TTS connections | Async-native WebSocket client |
| `openai` | The LLM connection | Sarvam is OpenAI-compatible, so the official SDK works |
| `python-dotenv` | Loads `.env` | Keeps the API key out of code and out of git |

That is the whole dependency list — six packages. No framework.

### Concepts you should be able to explain

**VAD — Voice Activity Detection.** Software that answers "is this 20 ms of
audio speech or not?" The two common choices are WebRTC VAD (tiny, local, fast,
somewhat trigger-happy) and Silero VAD (a small neural network, more accurate,
heavier). This project uses WebRTC VAD in its strictest mode *plus* a loudness
gate, because measured in a quiet room the looser modes called most silence
"speech" (section 9, bug 5).

**Endpointing.** Deciding the user has *finished* their turn. Done here by
counting silence: after `SILENCE_MS` (450 ms) of continuous non-speech, the
turn is over. Too short cuts people off mid-thought; too long feels sluggish.

**Streaming.** Every stage sends its output as it is produced rather than at
the end. STT transcribes while you talk. The LLM sends tokens as it writes.
TTS sends audio chunks as it synthesizes. This turns a *sum* of waits into an
*overlap* of waits, and is the single biggest latency technique in the project.

**Partial vs final transcripts.** Streaming STT sends provisional text while
you talk (which can change) and settled text when you stop (which won't).

**TTFT — Time To First Token.** How long until the LLM's *first* token, not the
whole reply. For voice this is what matters, because Priya can start speaking
sentence one while sentence three is still being written.

**TTFB — Time To First Byte.** The TTS equivalent: time until the first chunk
of audio. A WebSocket TTS beats HTTP because the connection is already open and
audio streams back in pieces.

**Barge-in.** Interrupting the agent mid-sentence. The agent must stop talking
immediately and listen. Section 5 covers how, and why it is the hardest part.

**Tool calling (function calling).** The LLM does not book anything itself. It
emits a structured request — `book(name=..., doctor_id=..., ...)` — the code
runs the real Python function, and the result goes back to the model. Every
business rule is enforced in that Python, never trusted to the model.

---

## 3. Every file, and what happens when you run it

```text
voice-agent/
│
│   RUN THESE
├── agent.py            the agent: python agent.py
├── step1_mic.py        test stage 1: mic and speaker
├── step2_vad.py        test stage 2: speech start/end detection
├── step3_stt.py        test stage 3: live speech to text
├── step4_llm.py        test stage 4: type to the LLM, watch it stream
├── step5_tts.py        test stage 5: type text, hear it spoken
├── step6_e2e.py        the whole agent, scripted, no microphone needed
├── test_rules.py       44 offline checks of the booking rules, free
├── calibrate.py        is the VAD sane in this room at this volume?
│
│   THE PIPELINE (imported by the above)
├── config.py           every tunable number in one place
├── audio_io.py         Mic (frames in) and Player (audio out, instant stop)
├── vad.py              turns frames into "start" / "end" events
├── stt.py              Sarvam speech-to-text client
├── llm.py              Sarvam chat client + sentence splitter
├── tts.py              Sarvam text-to-speech client
├── tools.py            the six booking functions; all rules live here
├── prompt.py           Priya's instructions; the emergency keyword check
├── latency.py          per-turn timing marks and the summary table
├── transcript.py       writes every conversation to a timestamped file
├── harness.py          the machinery that lets step6 run without a mic
│
│   DATA
├── data/hospital_data.json   the hospital and eight doctors
├── data/bookings.json        every booking (JSON, the source of truth)
├── data/appointments.txt     every booking event, one readable line each
├── data/conversations/       one timestamped transcript per session
├── data/cache/               fixed phrases pre-synthesized to audio
├── out/                      test output only; wiped on every test run
│
│   CONFIG AND DOCS
├── .env                SARVAM_API_KEY=...   (never committed)
├── .env.example        the template for .env
├── .gitignore          keeps .env, venv, data files and test output out of git
├── requirements.txt    the six dependencies
├── README.md           how to run, the pipeline, the build order
├── latency.md          measured numbers, what was tried, what the floor is
├── notes.md            this file
└── CLAUDE.md           architecture notes for an AI coding assistant
```

### Running the agent

#### `agent.py` — `python agent.py`

The program. On start it:

1. Opens the STT and TTS WebSockets and warms the LLM connection, all at once.
2. Loads the fixed phrases (opening line, emergency line, "One moment.") from
   `data/cache/`, synthesizing any that are missing.
3. Detects whether you are on speakers or headphones from the output device
   name and prints which mode it chose.
4. Opens a new transcript file under `data/conversations/`.
5. Speaks the opening line and starts listening.

Then it loops forever: read a 20 ms frame from the mic, feed it through VAD,
stream speech to STT, and when a turn ends, launch a task to transcribe, think,
call tools and speak. Ctrl+C closes every connection cleanly, prints the
latency table for the session, and says `Bye!`.

The `Agent` class is the state machine. Its methods, in the order a turn uses
them:

| Method | Role |
| --- | --- |
| `setup()` | connections, phrase cache, speaker detection, opening line |
| `step(frame)` | one frame through VAD; the per-frame state transition |
| `barge_in()` | stop Priya, or merge a pause — section 5 |
| `handle_turn(turn)` | one user turn: transcript → emergency check → reply |
| `think_and_speak(turn)` | the LLM loop with tool calls, up to 4 rounds |
| `speak(text, turn)` | one sentence to TTS |
| `say_cached(text, turn)` | a pre-synthesized phrase straight to the speaker |
| `close()` | shut down every connection and device |

### The step scripts — the assignment's build ladder

Each one exercises a single stage in isolation. They exist so that when
something breaks, you can find *which* stage without running the whole agent.

| Script | Run it and you get |
| --- | --- |
| `step1_mic.py` | Records 5 seconds, prints the shape, byte count and loudest sample, plays it back. Proves the sound card works and shows what "16 kHz, mono, int16" means in bytes. |
| `step2_vad.py` | Prints `SPEECH START` / `SPEECH END` as you talk. Tune `SILENCE_MS` here. |
| `step3_stt.py` | Streams your voice to Sarvam STT, prints pieces as they arrive and the final text with the time it took after you stopped. |
| `step4_llm.py` | A text chat: type a line, watch the reply stream token by token, see the TTFT in ms. |
| `step5_tts.py` | Type a sentence, hear it in Priya's voice, see the TTFB in ms. |

### Testing

#### `test_rules.py` — `python test_rules.py`

44 checks of the booking logic, sentence splitting and VAD. Free, instant, no
API, no microphone. It writes to `out/`, never to the real data files.

Covers every rule in the business brief: Sunday refused, lunch hour refused,
more than 14 days ahead refused, past dates, 9-digit phones, unknown doctors,
days a doctor doesn't work, children under 14 routed to Pediatrics, one
appointment per patient per doctor per day, taken slots, the free follow-up
within 7 days, the 2-hour cancel/reschedule cutoff, `find_booking` never
leaking another patient's data, and that every booking event is written to
`appointments.txt` in order.

One detail worth knowing: the "30 minutes ahead" rule depends on the current
time, so the test **freezes the clock** to a Monday at 10:00 by swapping in a
fake `datetime` class. Otherwise it could only run during OPD hours.

#### `step6_e2e.py` — `python step6_e2e.py`

The whole agent, driven by a script, **with no microphone**. Section 8 explains
the trick. Three scenarios:

- **emergency** — "my father has chest pain and is sweating a lot" → the
  emergency line is spoken, the LLM is never called, no booking is made.
- **barge-in** — Priya is asked a long question; mid-answer she is interrupted.
  Asserts the audio buffer is empty on the interrupting frame and the recording
  is silent afterwards.
- **booking** — four lines that complete a real booking with Dr. Sanjay Rao,
  then asserts the row in `bookings.json`: right doctor, ₹900, status booked.

Prints the latency table at the end and saves Priya's audio as WAV files in
`out/`. Flags: `--dry` runs only the VAD and STT checks on the scripted lines
(no LLM cost — run this first), `--only booking` runs one scenario, `--loud`
plays through the speakers instead of recording silently.

#### `calibrate.py` — `python calibrate.py`

Measures whether the VAD behaves in *your* room at *your* volume. Records
6 seconds of ambient noise, then plays speech through the speakers while
recording the mic, and runs the real `TurnDetector` over both. Reports the
noise floor, the echo level, and whether either produces a false "speech
start". Run it in a quiet room; it cannot tell a person talking nearby from
you, and no VAD can.

### The pipeline modules

#### `config.py`

Every tunable number, with a comment explaining each. Audio format (16 kHz,
20 ms frames, 640 bytes per frame), VAD settings, speaker-mode settings, the
Sarvam model names, history length, the instant phrases, cache and log paths.
Change the sample rate here and nothing else needs touching, because every
module imports its numbers from here.

#### `audio_io.py`

**`Mic`** opens an input stream. The sound card calls back on *its own thread*
every 20 ms; the frame is handed to the asyncio loop with
`loop.call_soon_threadsafe(queue.put_nowait, frame)`. That one line is the
bridge between the audio thread and the async world.

**`Player`** holds a byte buffer the sound card drains. Three things about it
matter:

- `stop()` clears the buffer, so audio stops *instantly*. If audio were written
  to a file and played, you could not cut it off mid-word. Barge-in depends on
  this.
- `busy()` says whether anything is queued — how the agent knows Priya is
  currently making sound.
- `watch_next(callback)` fires when audio queued *from now on* actually reaches
  the sound card, even if something earlier is still playing. It is how the
  latency log marks the moment the *answer* starts rather than the moment a
  cached phrase ahead of it does.

#### `vad.py`

`TurnDetector.feed(frame)` returns `"start"`, `"end"` or `None`. A frame is
speech only if **two** independent tests agree: `webrtcvad` says so, *and* the
frame's loudness (RMS) clears a gate. The gate adapts: it is the larger of
`MIN_RMS` and four times the rolling noise floor (the quietest fifth of the last
3 seconds). Consecutive speech frames for `START_MS` fire "start"; consecutive
silence for `SILENCE_MS` fires "end". Callers can raise the gate and the start
time for one frame at a time — that is how speaker mode works.

#### `stt.py`

`SarvamSTT`. Connects once and stays connected. `send(frame)` batches five
frames into one 100 ms message (fewer, larger messages are cheaper than one per
frame). Each message is a tiny WAV — 44-byte header plus PCM — base64 encoded,
because that is what the API expects. `final_text()` sends a `flush` and waits
briefly for the transcript. Handles the server closing idle sockets by
reconnecting and re-sending the chunk that failed.

#### `llm.py`

`stream_reply(messages, tools)` is an async generator. It yields
`("text", piece)` as words arrive, `("tool_start", None)` the instant the model
begins a tool call (so a filler can play while the arguments stream in), and
`("tools", calls)` at the end if the model wants functions run. Tool-call
arguments arrive as JSON *fragments* across many chunks and are reassembled by
index.

`split_sentences(buffer)` cuts finished sentences off the front of the
accumulating text so each can go to TTS immediately. It knows "Dr." is not the
end of a sentence.

#### `tts.py`

`SarvamTTS`. The protocol: open the socket, send a `config` message (voice,
language, sample rate), then `text` + `flush` per sentence. Audio comes back
base64-encoded and goes straight into the `Player`.

Two mechanisms worth understanding:

- **The `idle` latch.** The server sends a `final` event after each flush. The
  client counts flushes sent against completions received; `idle` is set only
  when they match. This is how anything can know "Priya has completely finished
  speaking" without guessing with a timer.
- **`detach()`** synchronously orphans the current socket so that audio still
  in flight from it is dropped the instant a barge-in happens. `reset()` then
  closes it and opens a fresh one.

`render(text)` synthesizes to bytes instead of the speaker — used to build the
phrase cache.

#### `tools.py`

The six functions the LLM can call, and every business rule:

| Function | Does |
| --- | --- |
| `list_doctors(dept)` | doctors in a department, or all |
| `check_slots(doctor_id, date)` | free 15-minute slots that day, or why there are none |
| `book(name, age, phone, doctor_id, date, time, visit)` | validates everything, writes the booking, returns id and fee |
| `find_booking(phone)` | that phone's upcoming bookings — nobody else's |
| `cancel(booking_id)` | up to 2 hours before |
| `reschedule(booking_id, date, time)` | same rules as booking |

`validate_slot()` is the single source of truth for "can this slot be booked",
reused by `check_slots`, `book` and `reschedule`. `book`, `cancel` and
`reschedule` each append one readable line to `data/appointments.txt` and save
`data/bookings.json`. The LLM only ever sees the function schemas (`TOOLS`) and
JSON results; it never touches the files.

#### `prompt.py`

`build_prompt()` assembles Priya's system prompt **every turn**: the current
date and time, a rolling 15-day calendar, the hospital details, the doctor
roster, the booking flow, the rules and the voice style. The calendar is how
"next Wednesday" becomes `2026-09-23` — the model reads it off a list rather
than calculating. `is_emergency(text)` is a plain keyword check that runs
before the LLM is ever called.

#### `latency.py`

Records named moments in each turn — `vad_end`, `stt_final`, `filler_start`,
`llm_first_token`, `first_sentence_ready`, `tts_first_audio`, `playback_start`
— with `time.perf_counter()`. Zero is when the user *actually stopped talking*:
VAD only notices `SILENCE_MS` later, so `t0` is backdated by that much. Without
this the numbers would hide the endpointing wait and look 450 ms better than
they are. Prints a per-turn report and an average/worst table at exit.

#### `transcript.py`

`Transcript` writes one file per session under `data/conversations/`. Every
user line, everything Priya said (including replies that were cut off), every
tool call with its result, barge-ins, pauses and STT socket events — each
stamped `[YYYY-MM-DD HH:MM:SS]` and flushed immediately so nothing is lost on a
crash. When something goes wrong in a conversation, read this file first.

#### `harness.py`

Everything `step6_e2e.py` needs to drive the agent without a microphone:
a fake mic, a recording player, a `Voice` that synthesizes the *user's* lines
with Sarvam TTS and caches them, a `Pump` that feeds frames at true wall-clock
rate, and the turn-completion latch. Section 8.

### Data files

| File | What it is |
| --- | --- |
| `data/hospital_data.json` | Hospital details and the eight doctors: id, name, department, days, hours, fee. Edit this to change the roster. |
| `data/bookings.json` | Every booking as JSON, with `booked_at`. The source of truth. Delete it to reset. |
| `data/appointments.txt` | The same events, one line each: `BOOKED`, `RESCHEDULED`, `CANCELLED`, with patient, doctor, slot and fee. For reading. |
| `data/conversations/` | One transcript per session, named by start time. |
| `data/cache/` | The opening line, emergency line and filler as raw PCM. Built on first run; deleting it just means they get synthesized again. |
| `out/` | Test output. Wiped at the start of every test run except `out/voices/`, the cached user-voice lines that keep re-runs free. |

---

## 4. One conversation, end to end

What happens when you say *"I want to book with the heart doctor on Wednesday."*

1. The **mic** delivers 20 ms frames into an asyncio queue, continuously.
2. `Agent.step()` feeds each frame to the **VAD**.
3. Before speech is confirmed, frames go into `preroll`, a 300 ms ring buffer.
   VAD needs 200 ms of speech before it says "start", so without preroll the
   first syllable would be lost.
4. VAD returns **`"start"`**. The preroll is flushed into STT, then live frames
   stream to STT as you keep talking.
5. You stop. 450 ms later VAD returns **`"end"`**. A `Turn` is created for
   timing and `handle_turn()` is launched as its own task.
6. `stt.final_text()` sends a flush and returns the transcript → `stt_final`.
7. The transcript is written to the conversation log and appended to history.
8. **Emergency check first.** `is_emergency()` is a keyword scan. If it hits,
   the cached emergency line plays immediately and the LLM is never called.
9. Otherwise the system prompt is rebuilt with today's date and calendar, the
   last 40 messages of history are attached, and the **LLM** is streamed.
10. The model decides it needs availability and begins a tool call. The
    instant the first tool-call chunk arrives, the cached **"One moment."**
    plays → `filler_start`. The arguments finish streaming; `check_slots("D03",
    "2026-09-23")` runs; the result goes back to the model.
11. The model now writes text. First token → `llm_first_token`. Text
    accumulates; the moment `split_sentences` finds a complete sentence it goes
    to **TTS** → `first_sentence_ready`. The model keeps writing sentence two
    while sentence one is being synthesized.
12. TTS audio chunks arrive → `tts_first_audio` → into the player buffer.
13. The sound card pulls the first bytes of the answer → `playback_start`.
    **This is the number that matters** — when you actually hear Priya.
14. The reply is logged and appended to history. The timing report prints from
    a separate task so it can never delay the next turn.

Then the flow, by design of the prompt, is **availability first**: Priya offers
the doctor, date and two or three times before asking for anything personal.
Only once you pick a slot does she collect name, age, phone and visit type —
skipping anything you already said — read it all back, get a clear yes, and
call `book()`. Then: *"Your appointment is confirmed. Please come fifteen
minutes early. Take care!"*

---

## 5. The hard parts

### Barge-in, and telling a pause from an interruption

When speech starts while the agent is busy, two very different things could be
happening, and they look identical to the VAD:

| Situation | What actually happened | Right response |
| --- | --- | --- |
| **Pause** | Priya hasn't said anything yet; you paused mid-sentence to think | Keep your words, merge them with what comes next |
| **Interrupt** | Priya is talking; you are cutting her off | Stop the audio instantly and listen |

The code tells them apart by whether Priya has spoken yet this turn:

```python
if running and not self.spoken:
    # PAUSE: keep the partial text, cancel the reply, merge with the next part
    self.carry = ...
    self.reply_task.cancel()
    return

# INTERRUPT
self.reply_task.cancel()             # stop generating
old = self.tts.detach()              # stop new audio arriving, synchronously
self.player.stop()                   # drop audio already queued
asyncio.create_task(self.tts.reset(old))
```

So *"I want to book…"* [pause] *"…with the heart doctor"* is understood as one
sentence, and the half-sentence never reaches the LLM on its own.

All three steps of the interrupt must happen, and the detach must be
*synchronous*: if the socket were only abandoned inside the background task,
audio still in flight would refill the buffer that was just cleared and Priya
would keep talking for a moment after being interrupted. That exact bug was
found and fixed (section 9, bug 2).

### Working without headphones

On speakers, the mic hears Priya. There is no acoustic echo cancellation in
this project, and adding real AEC means compiling C libraries on Windows. So
this was measured instead. At normal volume, the echo at the mic peaks at RMS
6500 — as loud as a real voice — so no simple loudness threshold can separate
them.

But the *shape* is different. Echo is **bursty**; speech is **sustained**. At
RMS ≥ 800, echo never ran more than 8 frames (160 ms) in a row, while real
speech sustained 29–41 frames. So while Priya is talking, plus a 400 ms tail
for what is still coming out of the speaker, a barge-in must be loud for
**300 ms straight**. Echo never qualifies. A person who speaks up does. The
trade: on speakers you interrupt with a firm voice, not a mumble. On
headphones there is no echo path, so the normal sensitive gate applies. The
agent picks the mode from the output device name at startup.

### VAD that survives a real room

`webrtcvad` mode 2 called **81 %** of a quiet room's silence "speech" (measured:
RMS ~70, nobody talking). Mode 3 called it 1 %. So the project uses mode 3 and
adds a loudness gate that must *also* pass. The gate adapts to the room — four
times the rolling noise floor — so it sits low in a quiet room and rises when a
fan is on. `START_MS` is 200 ms so a keystroke or a tap cannot start a turn.
None of this can filter out a *person* talking near the mic; that is speech.

### Memory that lasts a whole booking

Voice turns are short — "Yeah", "Okay", "Ages 22". A booking collects seven
things over seven to ten exchanges. The history was originally trimmed to 12
messages (six exchanges), and in a real conversation the chosen doctor and time
were trimmed out of the model's memory before the phone number arrived; Priya
then asked which doctor the caller wanted. It is 40 now. Only spoken text goes
into history — tool calls and results do not — so 40 messages is about 1000
tokens, roughly 130 ms of extra TTFT mid-conversation. Correctness wins.

### Knowing when a reply is *finished* (the harness)

To drive the agent with a script you must know when it has completely finished
answering, or the next line lands on top of the reply and triggers an
unintended barge-in. No single signal is right: the reply task returns before
any audio has arrived; the player buffer is momentarily empty *between*
sentences; the TTS completion event fires once per sentence, not once per
reply. The harness requires all three at once — task done **and** TTS idle
**and** player empty — and the ordering makes it airtight: once the task is
done no new sentence can be sent, so once TTS is idle no new audio can arrive,
so once the player is empty it stays empty.

---

## 6. The business layer

### Where the rules live

The prompt tells the model how to *behave*. Every rule is *enforced* in
`tools.py`. The model can be wrong; the code is the final check. From the
brief:

| Rule | Enforced in |
| --- | --- |
| Book from today up to 14 days ahead | `date_problem()` |
| Same-day only if the slot is 30+ minutes away | `validate_slot()` |
| One appointment per patient per doctor per day | `book()` |
| No Sunday, no lunch hour (13:00–14:00) | `date_problem()`, `validate_slot()` |
| 15-minute slots within each doctor's hours | `all_slots()` |
| Children under 14 go to Pediatrics | `book()` |
| Follow-up within 7 days of a first visit is free | `book()` |
| Cancel or reschedule only up to 2 hours before | `cancel()`, `reschedule()` |
| Phone must be 10 digits | `book()` |
| Never see another patient's booking | `find_booking()` filters by phone |

### The prompt

Three sections matter most:

**Availability first.** When the caller names a department, a doctor or a
symptom, call `check_slots` for each matching doctor and offer names, date and
times *before* asking for personal details. Then collect only what the caller
has not already said. (An earlier wording, "Collect: name, age, phone…", made
the model interrogate first — one line changed that.)

**Rules.** No medical advice, no medicine names. Emergency words mean stop
booking and give the emergency number. Never invent doctors, times or fees. Never
take payment. Never share other patients' details.

**Voice style.** One or two short sentences. No lists, no markdown, no emojis —
you cannot hear `**bold**`. "Doctor" not "Dr.". Times as "eleven fifteen AM",
fees as "nine hundred rupees", phone numbers digit by digit in two groups of
five. Everything written for the ear.

### Emergencies bypass the model

"chest pain", "can't breathe", "unconscious", "accident", "heart attack" and a
few more are matched by a plain keyword scan before the LLM runs. It is dumber
than the model, but it cannot be talked out of it, and for "my father has chest
pain" that is exactly the property you want. The response is pre-synthesized
and plays in about 0.8 seconds.

---

## 7. Latency

Measured with `step6_e2e.py` (synthesized speech through the real pipeline).
Zero is when the user stopped speaking.

| Moment | Typical | What it is |
| --- | --- | --- |
| ~1.7 s | `filler_start` | "One moment." — cached, plays the instant a tool call starts |
| 1.7–3.0 s | `playback_start` | the actual answer |
| ~0.8 s | emergency, total | the whole emergency line, cached |

Best real-answer turns land at about **1.65 s**. That is the floor, and it was
measured stage by stage:

| Stage | Measured | What was tried |
| --- | --- | --- |
| LLM first token | ~950 ms | Only one usable model exists; `sarvam-105b` is a reasoning model that streams thinking and never text; smaller Sarvam models are deprecated. No prefix caching (repeating an identical prompt does not speed up). Prompt size costs ~160 ms for the full 700 tokens. |
| TTS first byte | ~450 ms | Already at the fastest settings the API accepts. |
| STT final | ~250 ms | Wait cap tightened; rarely hit. |
| Endpointing | 450 ms | Swept 250–500 ms against real speech: below 450, natural phrase pauses split one sentence into two or three turns. |

None of the three services can be made faster from this side, so the agent
covers the gap instead:

- **Cached phrases.** The opening, the emergency line and the filler are
  synthesized once into `data/cache/` and played with no round trip. Emergency
  turns went from ~1.5 s to ~0.8 s.
- **The filler fires on the first tool-call chunk**, not after the tool round
  finishes — about a second earlier on booking turns. It was shortened from
  "One moment, let me check." (2 s of audio) to "One moment." (0.7 s), because
  the long one was still playing when the answer arrived and the answer queued
  behind it — measured as a 600 ms delay.
- **LLM warm-up at startup.** The first HTTPS call pays ~2.5 s of TLS setup;
  it now happens before the opening line, not on the caller's first turn.
- An instant "Okay." at 0.75 s was built and measured to work (`config.ACK`),
  but a word before *every* reply got tiresome in a real conversation. It ships
  off.

A lesson from the measuring: the first probe appeared to show one setting at
180 ms against 3320 ms for another. That was entirely TLS setup on a cold
connection — a warm-connection re-test with interleaved trials showed no
difference at all. One sample is not a measurement.

Full detail in `latency.md`.

---

## 8. Testing a voice agent without a microphone

The problem: testing normally needs a human talking. The solution here has
three layers.

**Offline** — `test_rules.py`: pure Python, 44 checks, free, instant.

**Mic-less end to end** — `step6_e2e.py`. The trick: **Sarvam TTS speaks the
user's lines too**, in a different voice. That audio is fed into the agent as
20 ms frames, so the real VAD, STT, LLM, tools and TTS all run with nobody
present. Priya's output audio is recorded to WAV instead of (or as well as)
the speaker.

```text
Voice (TTS, "rahul")  →  PCM  →  Pump  →  agent.step(frame)  →  real pipeline
                                                                      ↓
                                              CapturePlayer  →  out/*.wav
```

Design decisions that make it deterministic rather than flaky:

- Frames are fed at **true wall-clock rate**. Feeding faster would corrupt every
  latency number, because the log assumes the 450 ms of endpointing silence
  really took 450 ms.
- Turn completion uses the three-condition latch from section 5, not a sleep.
- `--dry` synthesizes each scripted line and runs the real VAD over it *offline*
  before spending anything on the LLM. It catches the two things that actually
  break a scripted run: a line with a pause over 450 ms splitting into two
  turns, and a phone number being spoken as words instead of digits.
- Tests point every data path at `out/` so the real files are never touched,
  and `out/` is wiped on each run so it only ever shows the last one.

**Real microphone** — the final test, which only a person can do: run
`python agent.py` and talk. Synthesized speech is cleaner than a human voice;
barge-in *feel* and behaviour in a noisy room can only be judged live.

---

## 9. Bugs found and fixed (and how each was diagnosed)

These are the most valuable things to be able to talk about, because each one
is a real root cause found with evidence, not a guess.

**1. The pipeline forgot the booking mid-conversation.**
*Symptom:* after the caller chose "12 noon with Dr. Ramesh" and gave name, age
and phone, Priya asked "which department or doctor do you need?"
*Diagnosis:* counted the messages in the transcript between the choice and the
phone number — 18. History was trimmed to 12. Reproduced exactly in a text
replay at 12; booked correctly at 40.
*Fix:* `HISTORY_MESSAGES = 40`.

**2. Barge-in leaked audio.**
*Symptom:* Priya sometimes kept talking for a moment after being interrupted.
*Diagnosis:* `player.stop()` cleared the buffer, but the TTS socket was only
abandoned inside a background task scheduled for the next loop tick. Until it
ran, the socket's reader still passed its identity check and refilled the
buffer.
*Fix:* a synchronous `detach()` that nulls the socket before `stop()`. Verified
by the harness: buffer 0 bytes on the interrupting frame, recording silent
after the cut.

**3. Every barge-in leaked a socket, and Ctrl+C printed a traceback.**
*Symptom:* `RuntimeError: Event loop is closed` at exit, sometimes twice.
*Diagnosis:* enumerated the SSL transports still alive after shutdown — one
`websockets` connection whose close had started but never finished. It
appeared only in runs with a barge-in. Cause: `barge_in()` called `detach()`
and discarded the socket it returned; `reset()` then called `detach()` again
and found nothing. The socket was orphaned with its reader blocked forever. A
21-turn session had leaked six.
*Fix:* pass the detached socket into `reset(old)`. Plus a SIGINT handler that
cancels only the main task so `close()` runs before asyncio tears everything
down. Tested by delivering a real `KeyboardInterrupt` to the running agent:
zero transports alive, no traceback.

**4. A "bug" that wasn't.**
*Suspected:* `reasoning_effort=None` was being sent as literal `null` and might
break every LLM call.
*Diagnosis:* it was sent as `null` (verified by intercepting the request), but
Sarvam accepts it, and five interleaved warm-connection trials showed no
latency difference between omitting it, `None` and `"low"`. The dramatic
difference in the first probe was cold-connection TLS setup.
*Fix:* removed as a no-op. Lesson: verify before fixing.

**5. VAD hallucinated speech in a quiet room.**
*Symptom:* the opening line was barged-in before the caller said anything;
then `START / PAUSE / END` looped and nothing reached the LLM.
*Diagnosis:* recorded 6 s of the actual room (RMS median 69 — quiet) and ran
each VAD mode over it: mode 2 flagged 81 % of frames as speech, mode 3 flagged
1 %.
*Fix:* mode 3 plus an adaptive loudness gate. Verified with `calibrate.py`:
zero false starts.

**6. The filler delayed the thing it was covering for.**
*Symptom:* on tool turns, `playback_start` came 550–650 ms after
`tts_first_audio`.
*Diagnosis:* "One moment, let me check." is 2 s of audio; the answer arrived
1.2 s in and queued behind it.
*Fix:* "One moment." (0.7 s). Gap now 10–40 ms.

**7. A shorter silence timeout fragmented speech.**
*Symptom:* at `SILENCE_MS = 300`, one scripted line produced three turns.
*Diagnosis:* swept 250–500 ms against eleven real speech samples; natural
phrase pauses run to ~450 ms.
*Fix:* 450 ms — the honest floor, and 50 ms better than the original.

**8. Pausing twice lost the first half of the sentence.**
*Diagnosis:* the pause-merge overwrote the carried text instead of appending.
*Fix:* accumulate.

**9. Sockets died quietly.** STT dropped the audio chunk it was carrying on a
reconnect; TTS errors left the idle latch stuck forever; a reconnect could
recurse without bound. Each fixed; each now visible in the transcript.

**10. Small things that bite.** The microphone emoji in the "Listening" line
crashed on non-UTF-8 consoles. The phone number "9876543210" was synthesized
as "nine billion eight hundred seventy-six million…" until the test script
spelled it out digit by digit. The `.gitignore` ignored `.venv/` but the folder
was `venv/`.

---

## 10. Interview questions, answered

**What is a cascading voice pipeline, and what is the alternative?**
Separate STT, LLM and TTS models in sequence, each feeding the next. The
alternative is speech-to-speech: one model, audio in, audio out. It is faster
but there is no text in the middle to inspect — and the text in the middle is
where the booking rules and tool calls live. For a business agent that must
never invent a fee, cascading is the right trade.

**Where does the latency come from?**
Endpointing wait (450 ms — deciding you've finished), STT final (~250 ms), LLM
first token (~950 ms, the biggest), collecting the first sentence, TTS first
byte (~450 ms), playback (~16 ms). About 1.65 s for a real answer, which is the
floor of these three services.

**Why is streaming important?**
Without it: wait for the full transcript, then the full reply, then the full
audio — a sum. With it: the model writes while you're being transcribed, and
TTS speaks sentence one while sentence three is still being written — an
overlap. It is the single biggest latency technique.

**What is VAD, and what happens if the silence timeout is wrong?**
Voice Activity Detection: is this frame speech? Too short a timeout and the
agent cuts you off when you pause to think; too long and it feels sluggish.
Measured against real speech, natural phrase pauses run to ~450 ms, so that is
the setting. Below it, sentences split into multiple turns.

**What is barge-in, and how did you build it?**
Stopping the agent the moment the user talks over it. Three things at once:
cancel the reply task, detach the TTS socket *synchronously* so no more audio
arrives, clear the player buffer so queued audio stops. It also tells a real
interruption from the user merely pausing mid-sentence — in the pause case the
partial text is kept and merged. And on speakers, where the mic hears the
agent, a barge-in must be loud *and sustained* for 300 ms, because echo is loud
but bursty.

**What are TTFT and TTFB?**
Time To First Token — how long until the LLM's first token. Time To First Byte
— how long until the first TTS audio chunk. Both matter more than total time,
because playback starts at the first piece.

**Why should voice replies be short and plain?**
You cannot hear markdown, bullets or emoji, and a long reply means a long wait
before the useful part. Numbers and symbols must be written as spoken: "nine
hundred rupees", not "Rs.900".

**How did you measure latency, and what was the best improvement?**
`time.perf_counter()` marks at named points per turn, with zero backdated to
when the user actually stopped talking rather than when VAD noticed. Average
and worst print at exit. Measured every service directly and found the ~1.65 s
floor is theirs. The biggest wins were around it: cached phrases (emergency
1.5 s → 0.8 s), the filler on the first tool-call chunk, and the LLM warm-up
that took 2.5 s off the first turn.

**How did you test it without a microphone?**
Had the TTS speak the user's lines in a different voice, fed that audio in as
mic frames, and let the real pipeline run. The hard part was knowing when a
reply had *finished* — no single signal is right, so three are required at
once. Plus 44 offline checks of the rules, free.

**How do you make sure the model doesn't book something invalid?**
It can't. It only emits a function call; `book()` validates every rule in
Python and returns an error the model must relay. The model never touches the
data files.

**How does it handle an emergency?**
A keyword scan before the LLM runs. Dumber than the model, but it cannot be
talked out of it, and it plays a pre-synthesized line in 0.8 s.

**What would you do next?**
Real acoustic echo cancellation so speakers don't need a firm voice. Silero VAD
for noisier rooms. Sarvam's `codemix` STT mode for Telugu-English. And a
smaller, faster LLM if one becomes available — the 950 ms first token is the
wall.

---

## 11. Glossary

| Term | Meaning |
| --- | --- |
| PCM | Raw audio samples, no compression. Here: 16-bit signed integers, 16 000 per second, one channel. |
| Frame | 20 ms of audio = 320 samples = 640 bytes. The unit everything works in. |
| RMS | Root mean square — a frame's loudness. Quiet room ~70, speech ~2000, max 32 767. |
| Sample rate | Samples per second. 16 kHz is the standard for speech. |
| VAD | Voice Activity Detection — speech or not, per frame. |
| Endpointing | Deciding the user has finished their turn. |
| Preroll | The 300 ms of audio kept from just before VAD confirmed speech, so the first syllable isn't lost. |
| STT / ASR | Speech to text / automatic speech recognition. |
| TTS | Text to speech. |
| LLM | Large language model. |
| TTFT / TTFB | Time to first token (LLM) / time to first byte (TTS). |
| Tool calling | The model requesting a function; the code runs it and returns the result. |
| Barge-in | Interrupting the agent mid-reply. |
| AEC | Acoustic echo cancellation — subtracting the speaker's output from the mic input. Not in this project. |
| WebSocket | A persistent two-way connection; used for STT and TTS so audio streams both ways with no per-request setup. |
| asyncio | Python's single-threaded concurrency for I/O-bound work. |
| Event loop | The scheduler that runs coroutines when the thing they were waiting for is ready. |
| Latch | A condition that, once true, stays true — how the harness knows a reply is over. |
