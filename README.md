# Project Notes: Nova Suraksha Voice Agent

Everything about this project in one place — what it is, the technologies, how
every file works, how one turn flows end to end, and the concepts behind it.

---

## 1. What this project is

A **voice agent** called Priya that books hospital appointments. You talk into
your microphone, it listens, thinks, and talks back through your speakers. It
runs entirely in the terminal — no web UI, no phone line, no telephony.

It is built as a **cascading pipeline**: several separate models chained one
after another, each waiting for the one before it.

```text
 You speak
    │
    ▼
┌───────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────────┐
│  Mic  │──▶│ VAD │──▶│ STT │──▶│ LLM │──▶│ TTS │──▶│ Speaker │
└───────┘   └─────┘   └─────┘   └─────┘   └─────┘   └─────────┘
  audio     detect    speech    think     text       you hear
  frames    speech    to text   + reply   to speech  the reply
```

| Block | Full name | Job here |
| --- | --- | --- |
| VAD | Voice Activity Detection | Decides when you started and stopped talking |
| STT | Speech To Text | Turns your voice into text (Sarvam Saaras) |
| LLM | Large Language Model | Reads the text, decides what to say and which tools to call (Sarvam 105B) |
| TTS | Text To Speech | Turns the reply into audio (Sarvam Bulbul) |

**Why "cascading" matters:** each block waits for the previous one, so every
slow block adds directly to how long you wait before hearing a reply. That is
why latency is the central concern of this project.

The alternative architecture is **speech-to-speech** — a single model that
takes audio in and produces audio out (like GPT-4o realtime). It is lower
latency because there are no handoffs, but you lose the ability to inspect and
control the text in the middle, which is exactly where the booking rules and
tool calls live. For a business agent that must never invent a fee, the
cascading approach is the right trade.

---

## 2. Technologies used

### Services (all Sarvam AI)

| Layer | Model | Protocol | Why |
| --- | --- | --- | --- |
| STT | `saaras:v3` | WebSocket | Streaming, Indian languages, servers in India |
| LLM | `sarvam-105b-conversations` | HTTP streaming, OpenAI-compatible | Tool calling, fast first token |
| TTS | `bulbul:v3`, voice `priya` | WebSocket | Streaming audio out, Indian English voice |

All three are reached with one `SARVAM_API_KEY` from `.env`.

The LLM speaks the **OpenAI API format**, which is why the code uses the
`openai` Python library pointed at a different `base_url`:

```python
client = AsyncOpenAI(api_key=config.API_KEY, base_url="https://api.sarvam.ai/v1")
```

### Python libraries

| Library | Used for |
| --- | --- |
| `sounddevice` | Microphone input and speaker output (wraps PortAudio) |
| `webrtcvad-wheels` | Voice activity detection, runs **locally**, no network |
| `websockets` | The STT and TTS WebSocket connections |
| `openai` | HTTP client for the LLM (Sarvam is OpenAI-compatible) |
| `python-dotenv` | Loads the API key from `.env` |
| `asyncio` | Standard library — runs everything concurrently |

No framework. No LiveKit, no Pipecat, no Vapi. The whole pipeline is written by
hand so every step is visible and understandable.

---

## 3. Core concepts

These are the ideas the code is built on. Your assignment lists them as things
you should be able to explain out loud.

### VAD (Voice Activity Detection)

Software that answers "is this 20 ms of audio speech or not?" Two common ones:

- **WebRTC VAD** — tiny, runs locally, instant, no network. Used here.
- **Silero VAD** — a small neural net, more accurate (better at rejecting
  background noise), slightly heavier.

WebRTC VAD has 4 aggressiveness modes (0 relaxed → 3 strict). This project uses
mode 2 (`config.VAD_MODE`).

### Endpointing and the silence timeout

**Endpointing** is deciding that the user has finished their turn. This project
does it by counting silence: after `SILENCE_MS` (500 ms) of continuous
non-speech, the turn is over.

This single number is a real trade-off:

- **Too short** → the agent interrupts you when you pause to think mid-sentence.
- **Too long** → the agent feels slow and unresponsive.

300–500 ms is the usual range. 500 ms is used here, at the slow-but-safe end.

### Partial vs final transcripts

Streaming STT sends two kinds of results:

- **Partial (interim)** — a live guess while you are still talking, which can
  change as more audio arrives ("I want to" → "I want to book").
- **Final** — the settled transcript for a chunk, which will not change.

Sending audio *while* the user talks means that when they stop, most of the
transcription work is already done — you only wait for the last bit.

### TTFT (Time To First Token)

How long the LLM takes to produce its **first** token, not its whole reply.
This is what matters for voice, because you can start speaking sentence one
while the model is still writing sentence three. Smaller/faster models have
lower TTFT. Shorter prompts and shorter history also lower it.

### Token streaming

The LLM sends tokens as it generates them rather than one complete response at
the end. This is what makes "start talking before the model has finished
thinking" possible.

### TTFB (Time To First Byte)

The TTS equivalent — how long until the **first chunk of audio** arrives.
A WebSocket TTS beats a plain HTTP one because:

- the connection is already open (no DNS + TCP + TLS handshake per request),
- audio streams back in chunks as it is synthesized, instead of the server
  rendering the whole sentence and then sending one big file.

### Barge-in

When the user starts talking while the agent is still speaking, the agent must
**shut up immediately** and listen. Without it, a voice agent feels robotic and
frustrating. This is the hardest part of the project to get right and is
covered in detail in section 6.

### The key idea: never wait when you can stream

```text
Slow:  wait for all STT → wait for all LLM → wait for all TTS → play
Fast:  STT streams → LLM streams tokens → each finished sentence goes
       straight to TTS → audio chunks play as they arrive
```

---

## 4. The files

```text
config.py      all tunable settings in one place
audio_io.py    Mic (frames into a queue), Player (plays bytes, instant stop)
vad.py         turns 20 ms frames into "start" / "end" events
stt.py         Sarvam STT WebSocket client
llm.py         Sarvam chat streaming + tool calls + sentence splitter
tts.py         Sarvam TTS WebSocket client
tools.py       booking functions — every business rule is enforced here
prompt.py      Priya's system prompt, emergency keyword check
latency.py     per-turn timings, average and worst at exit
agent.py       joins everything together with asyncio

step1_mic.py … step5_tts.py   one script per pipeline stage, for testing
step6_e2e.py   full scripted conversation with no microphone
harness.py     the machinery step6 uses
test_rules.py  offline tests of the booking rules
```

### `config.py` — one place for every knob

```python
RATE = 16000            # 16 kHz, the standard for speech
FRAME_MS = 20           # audio handled in 20 ms pieces
FRAME = 320             # 16000 * 20 / 1000 samples per frame
BYTES_PER_FRAME = 640   # int16 = 2 bytes per sample

VAD_MODE = 2            # 0 relaxed … 3 strict
START_MS = 100          # this much speech = user started
SILENCE_MS = 500        # this much silence = user finished
PREROLL_MS = 300        # audio kept from just before speech started

HISTORY_MESSAGES = 12   # only recent turns are sent to the LLM
```

**One audio format everywhere.** 16 kHz, mono, 16-bit PCM, 20 ms frames — from
the microphone all the way to the speaker. No resampling anywhere in the
pipeline, which removes an entire category of "why is it noise/silence" bugs.
Changing the sample rate means editing this file only.

### `audio_io.py` — microphone and speaker

**`Mic`** opens a `sounddevice.RawInputStream`. The sound card calls
`_on_audio` on **its own thread**, which is not the asyncio thread — so the
frame is handed across safely:

```python
def _on_audio(self, data, frames, time_info, status):
    # runs in the sound card thread, not in asyncio
    self.loop.call_soon_threadsafe(self.queue.put_nowait, bytes(data))
```

`call_soon_threadsafe` is the bridge between the two worlds. The agent then
just reads `await mic.queue.get()`.

**`Player`** holds a `bytearray` buffer guarded by a lock. The sound card
callback `_fill` drains it, zero-padding when it runs dry:

```python
def stop(self):
    with self.lock:
        self.buf.clear()      # instant silence — this is what barge-in needs
```

Two things here are load-bearing:

- `stop()` clears the buffer, so audio stops *immediately*. If audio were
  written to a file and played back, you could not cut it off mid-word.
- `busy()` returns `len(self.buf) > 0` — the agent uses this to know whether
  Priya is currently making noise.
- `on_start` is a one-shot callback fired the moment real audio first reaches
  the sound card. That is what marks the `playback_start` latency point.

### `vad.py` — turn detection

Tiny and frame-counted (no clocks):

```python
def feed(self, frame):
    is_speech = self.vad.is_speech(frame, config.RATE)
    if is_speech:
        self.speech_ms += config.FRAME_MS; self.silence_ms = 0
    else:
        self.silence_ms += config.FRAME_MS; self.speech_ms = 0

    if not self.speaking and self.speech_ms >= config.START_MS:
        self.speaking = True;  return "start"
    if self.speaking and self.silence_ms >= config.SILENCE_MS:
        self.speaking = False; return "end"
    return None
```

Note it counts *consecutive* speech/silence — a single stray frame resets the
counter, which filters out clicks and pops.

### `stt.py` — speech to text over WebSocket

- Connects once, at startup, and stays connected (a warm connection saves
  ~100–300 ms of handshake on every turn).
- `send()` **batches 5 frames = 100 ms** per message rather than sending every
  20 ms frame separately — far fewer, larger messages is more efficient.
- Each message is a small WAV (44-byte header + PCM), base64-encoded, because
  that is the format this API expects.
- `final_text()` sends a `flush` signal, then waits briefly for the transcript.
  If pieces have already arrived it only waits 0.4 s; if none have, it waits up
  to 1.5 s.

```python
async def send(self, frame):
    self.pending.extend(frame)
    if len(self.pending) >= config.BYTES_PER_FRAME * 5:   # 100 ms
        await self._send_pending()
```

### `llm.py` — streaming replies and tool calls

`stream_reply` is an **async generator**. It yields as the model produces
output:

- `("text", "some words")` — text to speak, as it arrives
- `("tools", [...])` — at the end, if the model wants to call functions

Tool-call arguments arrive **split across many chunks**, so they are
accumulated by index before being used:

```python
for tc in delta.tool_calls or []:
    c = calls.setdefault(tc.index, {"id": "", "name": "", "args": ""})
    if tc.function.arguments:
        c["args"] += tc.function.arguments     # JSON arrives in pieces
```

**`split_sentences`** is the piece that makes the agent feel fast. It cuts
completed sentences off the front of a buffer so each one can go to TTS while
the model is still writing:

```python
for m in re.finditer(r"[.?!]\s", buf):
    ...
    if last_word in ABBREV:      # {"dr", "mr", "mrs", "ms", "no"}
        continue                 # "Dr. Rao" is not two sentences
```

The abbreviation guard matters: without it, "Dr. Sanjay Rao is free." would be
spoken as "Dr." then "Sanjay Rao is free." with an unnatural gap.

### `tts.py` — text to speech over WebSocket

The protocol is: open socket → send a `config` message → then `text` + `flush`
per sentence. Audio comes back as base64 chunks and goes straight to the
player.

Two mechanisms worth understanding:

**The `idle` latch.** The server sends `{"type":"event","data":{"event_type":"final"}}`
after each flush finishes. The code counts flushes sent vs completions received:

```python
self.flushes += 1        # in say()
self.idle.clear()
...
self.completions += 1    # in _receive(), on a "final" event
if self.completions >= self.flushes:
    self.idle.set()
```

This is how anything can know "Priya has genuinely finished speaking" rather
than guessing with a timer.

**`detach()`** synchronously orphans the current socket:

```python
def detach(self):
    old, self.ws = self.ws, None     # _receive() drops frames from now on
    ...
```

`_receive` checks `if ws is not self.ws: break`, so nulling `self.ws`
immediately stops audio from a socket being abandoned. Doing this
asynchronously would let in-flight audio refill the buffer that barge-in just
cleared — see section 6.

### `latency.py` — measuring everything

Six named marks per turn:

```python
ORDER = ["vad_end", "stt_final", "llm_first_token",
         "first_sentence_ready", "tts_first_audio", "playback_start"]
```

The important subtlety:

```python
self.t0 = now - config.SILENCE_MS / 1000
```

Zero is when the user **actually stopped speaking**, not when VAD noticed —
which is 500 ms later. Without this backdating, the numbers would silently hide
the endpointing wait and look 500 ms better than reality.

---

## 5. One turn, end to end

Trace what happens when you say *"I want to book with the heart doctor."*

1. **Mic** delivers 20 ms frames into an `asyncio.Queue`, continuously.
2. `Agent.step(frame)` feeds each frame to the **VAD**.
3. Before speech is confirmed, frames go into `preroll` — a 300 ms ring buffer.
   This exists because VAD needs 100 ms of speech before it says "start", so
   without preroll the first syllable would be lost.
4. VAD returns **`"start"`**. The preroll is flushed into STT, then live frames
   stream to STT as you keep talking.
5. You stop. 500 ms later VAD returns **`"end"`**. A `Turn` is created and
   `handle_turn` is launched as its own task.
6. `stt.final_text()` flushes and returns the transcript → mark `stt_final`.
7. **Emergency check first** — `is_emergency(text)` is a plain keyword scan
   that runs *before* the LLM is ever called. Emergencies never depend on model
   behaviour.
8. Otherwise `think_and_speak` sends `[system prompt] + history` to the LLM and
   streams the reply.
9. First text token → mark `llm_first_token`. Text accumulates; every time
   `split_sentences` finds a complete sentence it goes straight to TTS → mark
   `first_sentence_ready`.
10. If the model asked for a tool, the agent says a filler ("One moment, let me
    check.") so you do not hear dead silence, runs the function, appends the
    result, and loops. Up to 4 tool rounds.
11. TTS audio chunks arrive → mark `tts_first_audio` → into the player buffer.
12. The sound card pulls the first bytes → mark `playback_start`. **This is the
    number that matters** — it is when you actually hear Priya.
13. The reply is appended to history, which is trimmed to the last 12 messages.
14. The timing report prints from its own task.

### The tool loop

```python
for _ in range(4):                 # max 4 tool rounds
    async for kind, value in llm.stream_reply(messages, TOOLS):
        if kind == "text":
            ... speak each finished sentence ...
        else:
            calls = value
    if not calls:
        break
    if not self.spoken:
        await self.speak("One moment, let me check.", turn)   # no dead air
    for c in calls:
        result = run_tool(c["name"], c["args"])
        messages.append({"role": "tool", "tool_call_id": c["id"], ...})
```

---

## 6. Barge-in — the subtle part

When speech starts while the agent is busy, there are **two different
situations** that look identical to the VAD:

| Situation | What actually happened | Right response |
| --- | --- | --- |
| **Pause** | Priya has not said anything yet. You just paused mid-sentence to think. | Keep your words, join them with what comes next |
| **Interrupt** | Priya is already talking. You are cutting her off. | Stop the audio immediately and listen |

The code distinguishes them by checking whether Priya has spoken yet:

```python
running = self.reply_task and not self.reply_task.done()
if running and not self.spoken:
    # PAUSE: keep the text, merge it with the next part
    self.carry = ...
    self.reply_task.cancel()
    return

# INTERRUPT: stop everything
print("[BARGE-IN] stopping Priya")
if running:
    self.reply_task.cancel()
self.tts.detach()      # synchronous — see below
self.player.stop()     # instant silence
asyncio.create_task(self.tts.reset())
```

So saying *"I want to book…"* [pause] *"…with the heart doctor"* is understood
as one sentence, not two fragments, and the half-sentence is never sent to the
LLM on its own.

**Why `detach()` must be synchronous.** `tts.reset()` is a coroutine. If the
socket were only abandoned inside that task, then between `player.stop()` and
the task actually running, `_receive` would still consider the socket current
and would keep calling `player.play()` — refilling the buffer that was just
cleared. Priya would keep talking after being interrupted, unpredictably.
`detach()` nulls `self.ws` immediately, so the abandoned socket's frames are
dropped from that instant.

Three things must all happen for a clean barge-in:

1. Cancel the reply task (stop generating more)
2. Detach the TTS socket (stop new audio arriving)
3. Clear the player buffer (stop audio already queued)

Miss any one and the agent keeps talking over you.

---

## 7. The business layer

### Where the rules live

The system prompt tells the model how to behave, but **every rule is enforced
in Python** in `tools.py`. The LLM can hallucinate; the code is the final
check. `validate_slot()` is the single source of truth, reused by
`check_slots`, `book` and `reschedule`:

```python
def validate_slot(doc, date_str, time_str, rows, skip_id=None):
    problem = date_problem(doc, day)        # past / >14 days / Sunday / doctor's days
    if lunch_a <= minutes(time_str) < lunch_b:  return "No appointments during lunch…"
    if time_str not in all_slots(doc):          return "That time is not a valid slot…"
    if when < datetime.now() + min_ahead:       return "Slot must be at least 30 minutes…"
    if time_str in taken_times(...):            return "That slot is already booked."
```

### The six tools

| Tool | Input | Returns |
| --- | --- | --- |
| `list_doctors(dept)` | department (optional) | doctors in that department |
| `check_slots(doctor_id, date)` | doctor, date | free times, or why there are none |
| `book(name, age, phone, doctor_id, date, time, visit)` | patient + slot | booking id + fee, or an error |
| `find_booking(phone)` | phone | that phone's upcoming bookings only |
| `cancel(booking_id)` | booking id | ok, or an error |
| `reschedule(booking_id, date, time)` | booking id + new slot | ok, or an error |

Rules enforced in code: 14-day booking window, 30-minute minimum lead time, no
Sunday, no lunch hour (13:00–14:00), 15-minute slots, one appointment per
patient per doctor per day, children under 14 routed to Pediatrics, follow-ups
free within 7 days of a first visit with the same doctor, cancel/reschedule
only up to 2 hours before.

Data lives in `data/hospital_data.json` (8 doctors, hospital details) and
bookings are appended to `data/bookings.json`. No database.

### The prompt

`build_prompt()` is rebuilt **every turn** because it embeds the current time
and a rolling 15-day calendar:

```text
NOW: Saturday 2026-09-19 23:26
NEXT 15 DAYS: Sat 2026-09-19, Sun 2026-09-20, Mon 2026-09-21, …
```

That calendar is how "next Wednesday" becomes `2026-09-23`. The model does the
date arithmetic by reading it off a list, which it is far more reliable at than
calculating dates from scratch.

The voice style rules matter as much as the business rules:

```text
Reply in 1 or 2 short sentences. Warm, calm, polite.
No lists, no markdown, no emojis, no symbols.
Write Doctor instead of Dr.
Say times like eleven fifteen AM. Say fees like nine hundred rupees.
Say phone numbers digit by digit in two groups of five.
```

You cannot speak `**bold**` or a bulleted list. Everything must be written for
the ear.

### Emergencies bypass the model entirely

```python
EMERGENCY_WORDS = ["chest pain", "can't breathe", "heavy bleeding",
                   "unconscious", "accident", "heart attack", "stroke", …]

if is_emergency(text):
    await self.speak(EMERGENCY_LINE, turn)     # LLM never called
```

A keyword scan is dumber than the model but it cannot be talked out of it, and
for "my father has chest pain" that is exactly the property you want.

---

## 8. Testing

The problem: testing a voice agent normally requires a human with a
microphone. Two levels solve most of it.

### Offline — `python test_rules.py`

40 checks, no API calls, instant, free. Covers every booking rule and edge
case, sentence splitting, and VAD. Uses a frozen clock so the "30 minutes
ahead" rule can be tested at any hour:

```python
class FakeDatetime(datetime):
    @classmethod
    def now(cls, tz=None):
        return frozen
tools.datetime = FakeDatetime
```

### End to end with no microphone — `python step6_e2e.py`

The trick: **Sarvam TTS speaks the user's lines too.** That audio is fed into
the agent as 20 ms mic frames, so the real VAD, STT, LLM, tools and TTS all
run with no human present.

```text
Voice (TTS, "rahul")  →  PCM  →  Pump  →  agent.step(frame)  →  real pipeline
                                                                      ↓
                                              CapturePlayer → out/*.wav
```

Three scenarios: an emergency, a barge-in, and a complete booking.

Key design points:

- **Frames are fed at true wall-clock rate**, never faster. `latency.py`
  backdates `t0` by 500 ms assuming that silence really took 500 ms — feeding
  faster would corrupt every measurement.
- **Turn completion uses a three-condition latch**, not a `sleep()`:
  `reply_task.done()` AND `tts.idle.is_set()` AND `not player.busy()`. Each
  alone is wrong — the task finishes before audio arrives, the player is
  momentarily empty *between sentences*, and the TTS completion event fires
  once per sentence rather than once per reply.
- **`--dry` first.** It synthesizes each line, runs the real VAD over it
  offline, and checks the STT transcript — catching the two things that
  actually break a scripted run (a line with a pause over 500 ms splitting into
  two turns, and a phone number spoken as words instead of digits) without
  spending LLM credits.
- Tests point `tools.BOOKINGS` at `out/` so real data is never touched.

**This does not replace a real microphone test.** Synthesized speech is cleaner
than a human voice, and barge-in *feel* plus VAD tuning under background noise
can only be judged live.

---

## 9. Latency results

Measured over 7 turns, two consecutive runs:

| point | avg | worst |
| --- | --- | --- |
| vad_end | 500 ms | 500 ms |
| stt_final | ~760 ms | 1036 ms |
| llm_first_token | ~1900 ms | 2971 ms |
| tts_first_audio | ~2000 ms | 3467 ms |
| **playback_start (total)** | **~2000 ms** | **3492 ms** |

Best single turn: 1428 ms. The target was under 1 second, so this does not meet
it. Where the time goes:

| stage | cost | tunable? |
| --- | --- | --- |
| endpointing wait | 500 ms | Yes — `config.SILENCE_MS`, one line |
| STT final after VAD end | ~257 ms | Somewhat |
| **LLM first token** | **~1090 ms** | Yes — prompt size |
| TTS first byte | ~575 ms | No, upstream |
| playback start | ~16 ms | Already minimal |

The LLM dominates. The same model returns its first token in ~780 ms against a
two-line prompt, versus ~1090 ms here — the difference is the large system
prompt (hospital details + 8 doctors + 15-day calendar). Shortening it is the
biggest available lever, and it trades directly against the model's ability to
resolve dates and quote fees correctly.

See `latency.md` for the full write-up, including a measurement that looked
like a 3-second improvement and turned out to be TLS connection setup on a cold
first call — a good reminder that one sample is not a measurement.

---

## 10. Things worth knowing

- **Use headphones.** There is no acoustic echo cancellation, so on speakers
  the microphone hears Priya and the agent talks to itself.
- **VAD fires on background noise.** At `VAD_MODE=2`, ambient room noise alone
  triggered a speech-start in testing. Mode 3 or Silero VAD would help.
- **The TTS socket times out** if left idle for about a minute. The keepalive
  ping is silently ignored by the server, so the socket does close — but
  `say()` detects it and reconnects, costing one small hiccup.
- **`data/bookings.json` is the whole database.** Delete it to reset.
- Run the pipeline stages in order (`step1_mic.py` → `step5_tts.py`) when
  debugging; each one isolates a single block.

---

## 11. Interview questions, answered

**What is a cascading voice pipeline? What is the other type?**
Separate STT, LLM and TTS models chained in sequence, each waiting on the last.
The alternative is speech-to-speech: one model, audio in and audio out, lower
latency but much less control over the text in the middle — which is where tool
calls and business rules live.

**Where does latency come from?**
Endpointing wait (~500 ms), STT final transcript (~250 ms), network to the LLM,
LLM first token (~1090 ms, the biggest), collecting the first sentence, TTS
first byte (~575 ms), playback start (~16 ms).

**Why is streaming important?**
Without it you wait for the full transcript, then the full reply, then the full
audio — serially. With it, the model starts writing while you are still being
transcribed, and TTS starts speaking sentence one while sentence three is still
being generated. It turns a sum of latencies into an overlap.

**What is VAD and what happens if the silence timeout is wrong?**
Voice Activity Detection decides whether a frame is speech. Too short a timeout
and the agent cuts you off when you pause to think; too long and it feels
sluggish. 300–500 ms is the usual range.

**What is barge-in and how did you build it?**
Stopping the agent the moment the user speaks over it. Three things must happen
together: cancel the LLM/reply task, detach the TTS socket synchronously so no
more audio arrives, and clear the player buffer so queued audio stops. It also
distinguishes a genuine interruption from the user merely pausing mid-sentence
— in the pause case the partial text is kept and merged with what follows.

**What is TTFT and TTFB?**
Time To First Token — how long until the LLM's first token. Time To First Byte
— how long until the first chunk of TTS audio. Both matter far more than total
generation time, because playback starts at the first piece.

**Why should voice replies be short and plain?**
You cannot hear markdown, bullet points or emoji, and a long reply means a long
wait before the useful part. Numbers and symbols have to be written the way
they are spoken: "nine hundred rupees", not "Rs.900".

**How did you measure latency, and what was the best improvement?**
`time.perf_counter()` marks at six named points per turn, with zero backdated
to when the user actually stopped speaking rather than when VAD noticed.
Average and worst print at exit. The most valuable change was not a speed-up
but a correctness one: the timing report used to run inside the reply task,
which kept the agent "busy" for up to 5 seconds and turned the user's next
sentence into a barge-in that cancelled the report before it printed.
