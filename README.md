# Backseat

A companion that watches a screen you share and talks about it live: reacting to
what just happened and, true to the name, pushing for what it wants to see next.

Extracted from [Sei](https://github.com/sei-studio/sei), where it ships as of
v0.5.4. This is the real implementation, mirrored file for file at the same
paths, not a rewrite. It is not a standalone app: it expects an Electron host
with a chat brain, a character persona, a memory store, and a voice call.

Almost every number below was measured rather than chosen, and several sections
describe working code that was deleted anyway. Those are the parts worth reading.

## Architecture

![Backseat architecture](docs/architecture.png)

<sub>Diagram source: [`docs/architecture.tex`](docs/architecture.tex) (TikZ). Rebuild with `tectonic docs/architecture.tex && pdftocairo -png -r 200 -transp -singlefile docs/architecture.pdf docs/architecture`.</sub>

The unit of work is a **tick**: one image grid, the grid from the previous look,
the local transcript of the last few seconds, the title of the window being
watched, and the reason the tick fired. A tick is the only thing that reaches
the model, and every tick that reaches the model produces a spoken line.

## When it talks

Four wakes, in strict priority: `user > start > jolt > idle`. Higher preempts
lower, one turn is in flight at a time, and a tick that cannot preempt is
**dropped, never queued**. A queued reaction describes a moment that has passed
and reads as confusion rather than lateness.

| Wake | Fires when | Notes |
| --- | --- | --- |
| `user` | The player spoke or typed | Always answered, ignores the speak gap |
| `start` | 1.8 s after the share opens | Once per session |
| `jolt` | A local loudness or colour discontinuity | No model in the loop |
| `idle` | A scheduled timer | The steady state |

`idle` is a **shifted exponential** over [12 s, 60 s], mean ~28 s. The
distribution matters more than the numbers: past the floor an exponential has a
constant hazard rate, so the wait is memoryless and the player cannot learn the
rhythm. A uniform draw has the opposite property, where the longer the silence
runs the more overdue the next line feels, which is the metronome effect this
avoids.

`start` exists because a session used to open with silence. Nothing has jolted
yet and the idle floor is 12 s, so a player who pressed Share and waited got a
companion that appeared not to have noticed. It is 1.8 s rather than 0 because
at 10 Hz the frame ring has no history to composite instantly, and because the
picker is still dismissing over the thing being shared.

`MIN_SPEAK_GAP_MS` (8 s) drops `jolt` and `idle`, never `user`. This is the only
silence in the system, and it is deliberately mechanical: **a rule that never
looks at a moment cannot misjudge one.**

## The image grid

Still-image models cannot watch video, so six frames are stitched into one
picture and the model is told to read them in order. Layout, frame count and
ordering reproduce the IG-VLM paper exactly: six frames beat 4/9/12/16/20, and
near-square grids beat wide ones, so it is 3 rows by 2 columns of 16:9 cells
(a 32:27 canvas) filled row-first.

**Frames are log-spaced, not uniform.** The grid takes the nearest sample to
each of `[6, 3, 1.5, 0.75, 0.375, 0.1875]` seconds ago, off a uniform 10 Hz JPEG
ring. Six frames at 1 Hz cannot show a sequence. A dodge-then-fire is about
600 ms end to end, so at one sample a second it is one frame or none, and
"you dodged then fired" was never recoverable from the pixels the model was
given. Geometric spacing keeps the same six-second reach while putting the last
three cells inside the final second, which is where an action-and-consequence
pair actually lives. Verified on real footage before it was built: on one
Valorant grid the HUD round timer reads 1:13 / 1:10 / 1:08 / 1:07 / 1:07 / 1:07
across the cells and the ammo counter reads 5 / 1 / 6 / 6 / 5 / 4, resolving
fire, reload, emerge, aim, fire into three distinct states inside the last
600 ms.

**Size is pinned to the model, and there is a test for it.** Haiku 4.5 is a
standard-tier vision model: long edge ≤ 1568 px *and* ≤ 1568 visual tokens at
`ceil(w/28) * ceil(h/28)`. The largest legal 32:27 grid is **1204×1008 (cells
602×336) = 1548 tokens**. Oversize is a silent server-side downscale rather than
an error, so the test asserts both the cap and that budget is not being left on
the table.

**Identical cells collapse.** Consecutive cells showing the same picture are
dropped down to one and the canvas is resized to the survivors, so a paused
video costs **264 visual tokens instead of 1548** while a firefight still costs
six cells. This came from the companion itself, which, asked what was happening,
wanted to know why it had been shown "six identical YouTube frames". It was a
fair question: six identical cells claim six sampled moments and carry the
information of one. Because the grid is now variable-size, each tick states its
own frame ages; the cached contract can no longer describe the shape.

**The previous grid rides along as memory.** Chat history is text, so until
recently each look was the companion's entire visual world: it could read its
own last line but had no idea what it had been looking at when it wrote it. From
the second look on, the turn carries the previous grid at half linear size
(602×504, 396 tokens) with its age. It sits after the cache breakpoint, so it
never invalidates the prefix. It exists to make repetition visible, which became
the dominant failure mode the moment silence was removed.

## The jolt arm

Two detectors, both local, both measured against a **rolling baseline rather
than an absolute**, so a loud game and a quiet game behave alike. The kernels
live in `signals.ts` as pure functions over explicit state, which is what lets
the offline sim run the same code that ships instead of a re-derivation of it.

- **Gain**: a jump of 18 dB over the trailing median loudness, roughly an 8×
  amplitude step. An explosion in a quiet room, not gunfire during a firefight.
- **Colour**: the largest change over a 4×3 split of a 32×18 thumbnail, taken at
  **two lookbacks** (1.0 s and 2.5 s, max of the two), against a bar of
  `median + 4 × MAD` with a 0.15 floor.

Every part of that colour rule replaced something simpler that did not work. A
whole-frame mean erases any localised change at any threshold, so the arm only
ever fired on hard scene cuts; block-max measures a change against the area it
actually covers. A single one-second lookback misses a walk through a doorway,
which takes one to three seconds and never looks like more than panning inside
any one-second window. And no fixed number can work: on the test clip the
block-max delta's own median is 0.313, so a fixed 0.34 fires continuously in a
shooter and never in a calm game.

**The two arms hold separate refractory clocks, and separate periods**: gain 20 s,
colour 6 s. The moment colour got sensitive it started swallowing confirmed gain
events. The period split came later, off a Reels recording with six verified
swipes where every gap was under 20 s, so the shared clock, not the picture, was
choosing which ones were noticed. A run of scene changes is a run of different
subjects; a run of loudness spikes is one scene. The 6 s floor is set by the
2.5 s lookback, since a change stays inside that window for 2.5 s after it ends
and a shorter period double-counts it (measured: 5 s re-fired, 3 s re-fired).

`JOLT_COLOR_MAD` is the one-line sensitivity dial.

## Sound stays local

Screen audio has exactly **two consumers, both local**: the gain arm above, and
a streaming transcript from a small local Whisper model. **No audio byte reaches
a remote model.** Whatever the platform source, audio is normalised to 16 kHz
mono PCM (`pcm.ts`, pure and tested), so the platform difference is contained to
the source.

The transcript is a ring of timed segments, not per-tick transcription: Whisper
chews 3 s chunks continuously, so a tick only waits a bounded moment (1.2 s cap)
for the in-progress tail instead of transcribing 6 s on demand and putting 1–2 s
of latency in front of every line. The window's text rides the tick framed in
the prompts as quoted game audio: never the player, never instructions.

**Capturing it.** Windows uses Chromium's desktop loopback. On macOS that path
returns a track labelled "System audio" carrying digital silence. Measured on
macOS 26.4 / Electron 42 / Chromium 148, with both loopback feature flags
enabled and verified applied, in every request shape. Electron documents
loopback as Windows-only and that matches. But the OS itself is fine, which is
how OBS records desktop audio, so macOS ships a small Swift helper
(`native/mac-audio-tap`) that captures system audio through ScreenCaptureKit and
streams PCM to the renderer. It needs no install and no new permission (it rides
the Screen Recording grant the picker already forced), and its filter **excludes
the app's own audio**, so the companion never hears its own voice. Windows
loopback cannot exclude, and occasionally transcribing its own line is the
accepted cost there.

Source order: Windows loopback → the mac tap → a virtual output device such as
BlackHole → video only. Without sound the gain arm never fires and there is no
transcript; the grid and the colour arm are unaffected.

## What is being watched

Every tick carries `shareLabel`: the shared window's current title, or on a
whole-screen share the frontmost window's, re-read every 5 s because a tab
switch changes the screen under a fixed source id.

**What it replaced was working, and was removed anyway.** An earlier version ran
a full OCR pass over every other frame: a bundled Swift `VNRecognizeTextRequest`
helper on macOS, tesseract.js elsewhere. It was good: whole phrases at ~72 ms,
94 of 94 frames, 23 words a frame against Tesseract's 6. It answered the wrong
question. A HUD full of numbers does not distinguish a game from a stream of
that game, and four words of window title do, for one window enumeration every
5 s instead of the most expensive local work in the pipeline. Do not re-add OCR
without a specific thing it is for that the title cannot carry.

## What it says

**It always speaks.** The contract has been through three positions on silence.
Sanctioning it as "the normal outcome" produced a mute companion: five ticks in
a live session, five silent turns. Delegating the decision to a per-tick note
measured 68% of turns producing a line, and reviewing that run the silences were
not taste, they were error. A scheduled look at a smoke going down mid-site with
four rounds left produced nothing. There is no wording of "stay quiet when
it is right to" that a small model applies at a human's bar rather than at its
own much higher one, so the option is gone.

**Lines must not narrate.** Speaking every time exposed what the lines actually
were: "you just got caught", "you just used a skill", "health is dropping". All
true, all describing a screen the player is looking at. The contract now spends
the line on what the player does *not* have (an opinion, a question, a want),
and the thing that moved it was four BAD/GOOD contrast pairs, not more
instruction: 0/10 lines asked anything before, 10/10 after, median 35 words down
to 20.

**When a rule is not holding, name the sentence, not the rule.** The general ban
on narration did not stop the companion opening with "you went from X to Y" on a
short-video feed, and stating in prose what a feed is did not either. Two
BAD/GOOD pairs quoting that exact sentence did: 3/6 to 0/6 on the feed, and
ablated on unrelated Valorant footage over the same eleven looks, 5/11 openings
of that shape down to 1/11 with median length 27 words down to 21.

**Em dashes are stripped in code, not asked away.** "Do not use em dashes" sat
in the contract for a day and the model wrote one in eight of ten lines.
`stripDashes` fixes it after the fact. It matters more here than in chat because
these lines are spoken aloud, and a dash is not a sound.

**Attaching tools suppresses speech.** This is the finding worth carrying
somewhere else. Same prompt, same grids, n=60 per condition:

| Tools attached | Turns that produced a line |
| --- | --- |
| none | 60/60 (100%) |
| `remember` only | 47/60 (78%) |
| `save_clip` only, current wording | 43/60 (72%) |
| both, what backseat ships | 41/60 (68%) |
| `save_clip` only, original wording | 37/60 (62%) |

Two effects. The large one is structural: a tool array costs about a fifth of
the lines whatever it says, so the clip feature is not free and the 32-point gap
is the honest price of shipping it. The small one is wording. A tool description
that reads as a general judgement about how interesting moments usually are
("most good moments are not clip-worthy") leaks from "do not use the tool" to
"do not speak"; scoping the rarity to the file rather than to the moment
recovers about 6 of the 38 lost points, which at this sample size is
directional, not proven. An explicit "tools do not gate speech" paragraph in the
contract was tried and did not help (43 against 41), so this is not fixable by
asking. **Suspect it first whenever a surface goes quiet.**

## Cost

Every tick carries a fresh 1548-token image at a 12–60 s cadence, so the cache
layout *is* the cost model rather than hygiene. The last cache breakpoint sits on
the final history message, not on the image message (which is unique forever and
can never be read back), and the history window is **anchored rather than slid**,
so appending a line does not change `message[0]` and invalidate the whole prefix.
Confirmed live: `cacheRead` steady at ~9k from the second tick, `cacheWrite` near
zero.

One tool array is used for every tick kind. Ticks land seconds apart, well inside
the cache TTL, so per-kind arrays would invalidate the prefix almost every turn
for nothing.

## Clips

`save_clip` writes the last 15 seconds to disk and attaches it to the chat line
that asked for it. A WebM segment is only decodable from its own header, so the
tail of a chunk list is not a clip: **two recorders staggered by half a period**
mean the longest-running one always yields a complete file containing the
requested window. The honest cost is that a saved clip runs 15–30 s rather than
exactly 15. Two MediaRecorders encoding 720p60 for a whole session is the single
most expensive thing in the pipeline, so it sits behind `CLIPS_ENABLED` and is
the first dial to turn if capture costs too much.

## Two buffers, not one

Conflating them was the first design's mistake. The frame ring only needs one
grid's span plus latency slack, so it is **9 s**. The 15 s belongs solely to clip
capture. When clipping is off, the whole MediaRecorder path disappears and the
retained window is the 9 s ring alone.

## What was removed, and why

Three components were deleted rather than repaired. Each is worth stating,
because a diff cannot explain why working code left.

- **A small VLM salience gate**, asked every 6 s whether anything significant
  had happened, with a learned per-session threshold. Replaced by the idle
  schedule. Measured end to end it said yes to almost everything, and the
  narration-novelty scheme meant to fix it carried 0.037 of real temporal signal
  against 0.25 of pure resampling noise. A dice roll is cheaper and no worse,
  and unlike a gate it cannot be wrong in a way that is invisible.
- **The OCR pass**, described above. It worked; it answered the wrong question.
- **The always-on-top overlay window.** Capture lived in its own window because
  Chromium clamps timers in a hidden or occluded renderer, and the main window
  is exactly that while the player is in a fullscreen game. That is already
  solved elsewhere: `backgroundThrottling: false`, and a frame pump built on
  `MediaStreamTrackProcessor` in a worker, which is throttle-immune either way.
  Deleting it removed a second renderer, a duplicated state copy and a push
  fan-out. Sharing is now a **call control**, the way Discord does it: share
  button, source picker, the preview takes over the call window and the avatars
  shrink to a strip.

## Integrating it

These files drop into an Electron app. The authority split is the contract:
the renderer owns pixels and sound (`getDisplayMedia`, the ring, grid
compositing, the clip recorders, local STT, and every wake), and
main owns the session, every model call, the window title, and on macOS the
audio tap. `src/shared/backseatIpc.ts` is the boundary and documents every
channel.

1. Copy `src/` and point the imports at your own chat brain, character store and
   memory store. `backseatService.ts` is the only file that touches them.
2. Register the IPC channels listed at the bottom of `src/shared/backseatIpc.ts`.
3. Add the Chromium switches for system audio in your main process:

   ```js
   app.commandLine.appendSwitch(
     'enable-features',
     'MacSckSystemAudioLoopbackOverride,MacLoopbackAudioForScreenShare',
   );
   ```

4. Set `backgroundThrottling: false` on the window that runs capture.
5. Build the macOS audio helper with `scripts/build-mac-audio-tap.sh` (a
   universal binary into `resources/audio-tap/`) and hook it on predev/predist.
6. Provide a Whisper worker. In Sei it is the same one voice calls use
   (transformers.js, wasm, q8), so the model downloads once and is shared.

Two host-side rules are not enforceable from inside these files and will bite:

- **There must be one conversation, not two.** If the host also runs a voice
  turn loop, anything that gives a companion a turn has to check whether that
  companion is currently sharing, and route the player's utterance into
  `sendUserTick` instead. Without this there are literally two turn loops
  against the same thread and the same call, one with the grid and no
  microphone, one with the microphone and no grid, and the player gets a
  companion who can see their screen and never hears them, talking over one who
  can hear them and cannot see. Nothing is broken in either loop. There are two
  of them and neither knows.
- **Anything the service reads off a tick must be in the host's tick schema.**
  Zod strips undeclared keys, so a field the renderer sends and the schema does
  not name arrives as `undefined` with no error anywhere. The previous-grid
  memory shipped, was documented, and never once reached the model for two days
  in exactly that state, invisible because a null previous grid is a legal state
  on the first tick of every session.

## Verifying it

Not by launching the app. A live session is unrepeatable: thresholds cannot be
swept, the timeline cannot be replayed, and "did the colour arm fire on that room
change" is a question about a moment that has already gone.

```sh
npx tsx scripts/backseat-sim.ts <video.mp4> [--dry]   # the voice-over it would have produced
npx tsx scripts/backseat-transcribe.ts [out-dir]      # offline STT for the review video
npx tsx scripts/backseat-render.ts [out-dir]          # the run, rendered beside the footage
```

The sim imports the real signal kernels, the real offsets and grid geometry, the
real idle distribution and priority ladder, and the real prompts and tool list,
so a threshold tuned there is tuned against shipping code. The persona is a stub
and there is no player to type, so it measures timing and what the grid supports,
not how a particular companion sounds. It also **cannot check prompt caching**:
its stub prefix is ~1.1k tokens and Haiku will not cache below 2048, so cache
hits have to be read off a live session's log.

Unit tests run with `vitest`. The pure kernels (`pcm.ts`, `signals.ts`,
`transcriptRing.ts`, and the grid geometry) are where the coverage is.

## Files

| Path | Role |
| --- | --- |
| `src/shared/backseatIpc.ts` | Constants, the tick type, and the contract between both halves |
| `src/renderer/src/lib/backseat/captureWorker.ts` | Frame ring, grid compositing, duplicate collapse, jolt detection |
| `src/renderer/src/lib/backseat/captureController.ts` | Screen and sound capture, clip recorders, wake timing |
| `src/renderer/src/lib/backseat/signals.ts` | Gain and colour kernels. Pure and tested |
| `src/renderer/src/lib/backseat/pcm.ts` | Downmix, resample, loudness. Pure and tested |
| `src/renderer/src/lib/backseat/transcriptRing.ts` | Transcript segments and window selection. Pure and tested |
| `src/renderer/src/lib/backseat/sttStream.ts` | Streaming Whisper over the PCM feed, bounded flush at tick time |
| `src/renderer/src/lib/stores/useBackseatStore.ts` | Owns the capture session for the app, and the pending-share handshake |
| `src/renderer/src/components/backseat/` | Source picker and the clip card |
| `src/main/backseat/backseatService.ts` | Session state, tick arbitration, the companion turn |
| `src/main/backseat/backseatPrompts.ts` | The session contract, the per-tick note, the tools |
| `src/main/backseat/shareLabel.ts` | What the shared surface is called right now |
| `src/main/backseat/audioTap.ts` | Spawns the mac helper, relays its PCM to the renderer |
| `src/main/backseat/backseatLog.ts` | Per-session diagnostics, to terminal and to the host's log surface |
| `native/mac-audio-tap/main.swift` | ScreenCaptureKit system audio capture |
| `scripts/backseat-sim.ts` | Offline run of the real pipeline over recorded footage |
| `scripts/backseat-render.ts` | That run as a review video, with grids and both signal traces |

## Sources

- Wonkyun Kim et al., *An Image Grid Can Be Worth a Video: Zero-shot Video
  Question Answering Using a VLM*, arXiv:2403.18406.
  https://arxiv.org/abs/2403.18406
- Anthropic, image sizing and token cost.
  https://platform.claude.com/docs/en/build-with-claude/vision
- Electron, `setDisplayMediaRequestHandler`, which documents loopback audio as
  Windows only. https://www.electronjs.org/docs/latest/api/session
- Electron issue 47490, macOS loopback audio.
  https://github.com/electron/electron/issues/47490

## License

AGPL-3.0-or-later, same as [Sei](https://github.com/sei-studio/sei). See
[LICENSE](LICENSE).
