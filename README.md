# Backseat

**STILL IN DEVELOPMENT NOT TESTED**

Backseat lets an AI character watch you play a game, or watch a video with you,
and talk about it while it happens. You share a window. It sees the screen,
reacts to what just happened, and says what it wants to see you try next. It can
also save the last 15 seconds as a clip and send it to you in chat.

This is the screen watching part of [Sei](https://github.com/sei-studio/sei),
pulled out on its own so the design is easy to read. The code here is the real
implementation, not a sketch, but it is not a standalone app. It expects an
Electron host with a chat brain, a character persona, and a memory store.

## The problem

An AI that watches your screen has to answer one question well: **when should it
say something?**

Get it wrong in one direction and it talks over everything. Get it wrong in the
other and it sits there silent while you do the best play of your life. Calling
a large model on every frame would answer the question, but it costs far too
much and it is far too slow.

So the work is mostly about spending model calls only when they are worth it.

## How it works

There are three parts. A cheap part that always runs, a small model that decides
if something happened, and a large model that actually talks.

```
screen + sound
      |
      v
 [ ring buffer ]  keeps the last 9 seconds, one chosen frame per second
      |
      v
 [ image grid ]   6 frames stitched into 1 picture
      |
      +--> [ small VLM ]  "did something significant happen?"   every 6 seconds
      |          |
      |          v
      +----> [ tick ] <---- loud sound or big colour change (no model)
      |          ^
      |          |
      +----- you say something (always answered)
                 |
                 v
          [ large model ]  writes one or two short lines, in character
                 |
                 v
          speech, or chat, and sometimes a saved clip
```

### The ring buffer, and picking one frame per second

The screen is captured at 60 frames per second. Keeping all of them would be
gigabytes, so the buffer keeps **one frame per second**, and picks that frame by
**which one had the loudest sound**. In a game the loudest moment in a second is
usually the moment that mattered: the shot, the hit, the explosion.

The naive way to do this is to store all 60 frames and pick at the end of the
second. Instead the code keeps a running best. Each new frame either beats the
current best and replaces it, or is thrown away immediately. That means **one
image is held at a time and one image is compressed per second**, instead of 60,
while still looking at every frame.

The buffer holds 9 seconds: 6 for the grid, plus 3 spare so a slow model call
still finds the moment it was asked about.

### The image grid

A model that reads still pictures cannot watch video. The trick is to stitch
several frames into **one picture**, laid out in a grid, and tell the model to
read them in order. It then sees change over time inside a single image.

This follows the IG-VLM paper, which tested this idea and found the best setup:

- **6 frames**, which beat 4, 9, 12, 16 and 20
- a **near square grid**, which beat wide ones
- read **left to right, then down**

Since each frame is widescreen, 3 rows of 2 gives a nearly square picture, so
that is the layout used here. The prompt describes the layout to the model,
because without that it just describes six unrelated pictures.

The grid is sized to the exact limit of the model reading it. Claude Haiku 4.5
accepts up to 1568 pixels on the long edge and 1568 image tokens, where an image
costs `ceil(width / 28) * ceil(height / 28)` tokens. The biggest grid that fits
is **1204 by 1008 pixels, which is 1548 tokens**, with each cell 602 by 336. Going
bigger is not an error, the image is just silently shrunk, so there is a test
that keeps these numbers honest.

### Three ways a comment gets triggered

Every trigger produces the same thing, called a **tick**: one image grid plus the
reason it fired. Only main process code decides if a tick becomes speech.

**1. You said something.** Always answered. When you start talking or start
typing, the grid is captured **at that moment**, not when you finish, so the
character looks at what made you react rather than what is on screen ten seconds
later. It is sent with your finished sentence.

**2. The small VLM says something happened.** Every 6 seconds a fresh grid goes
to a small vision model with one question: was there a significant change, like a
kill, a revive, a discovery, or a big change of location. If yes, that is a tick.

**3. A jolt.** A very large jump in loudness, or a near total change of screen
colour, with no model involved at all. This catches the thing that happens right
after a check, so you do not wait a full 6 seconds. The thresholds are set very
high on purpose and there is a 20 second cooldown, so this stays rare.

Ticks 2 and 3 are dropped, never queued, if the character is already thinking or
spoke less than 8 seconds ago. A late reaction describing a moment that has
already passed reads as confusion, not as being slow.

### Why the small model's threshold is learned, not fixed

The goal is that roughly one grid in four is worth commenting on. A fixed cutoff
cannot do this. Small vision models say yes to almost anything, and their stated
confidence sits in a narrow band whether they are right or wrong.

So instead of reading the yes or no answer, the gate reads the **probability of
the yes token**, and sets its cutoff at the **upper quartile of the last 40
scores in this session**. A fast shooter and a slow strategy game then each get
judged against their own normal, rather than against a number picked in advance.

If the gate errors or has no key, it returns no. A broken gate makes the
character quiet, never noisy.

### The large model

Only a tick reaches it. It gets the grid, the character persona, memory,
knowledge files, and recent conversation, all shared with every other surface, so
it is the same character you talk to elsewhere. It is told to reply with one or
two short lines, to stay in character, and that **saying nothing is normal and
expected**. Most ticks end in silence.

The part that gives the feature its name is in the prompt: it should have
opinions about what you do next, ask for the thing it wants to see, and take the
win when it was right. It is meant to be a friend on the couch, not a
commentator.

### Clips

The character can call a tool to save the last 15 seconds. The clip lands in your
normal chat as a playable card.

A recorded video segment can only be decoded from its own start, so you cannot
just keep the tail of a recording. Two recorders run offset from each other by
half a period, so stopping the older one always gives a complete, playable file
covering the moment asked for. The honest cost is that a clip runs 15 to 30
seconds rather than exactly 15.

Clip recording is the most expensive thing in the whole pipeline, because it
encodes video continuously for the whole session. It is behind a single switch
(`CLIPS_ENABLED`) and turning it off removes it completely. Nothing else depends
on it.

## Sound

Sound matters twice: it picks which frame each second, and it drives the jolt
trigger.

- **Windows** can capture what other apps are playing.
- **macOS cannot**, as of macOS 26.4 with Electron 42 and Chromium 148. This was
  measured, not assumed. With the documented feature flags enabled and confirmed
  applied, asking for system audio gives a track labelled "System audio" that
  contains pure silence. This happens whether audio is asked for together with
  video, in a separate request, or on its own with no video at all. Electron
  documents this feature as Windows only, and that matches what the machine does.

On macOS the code looks for a virtual audio device instead, such as BlackHole or
Loopback. If you have one installed and selected as your output, sound works.

Without sound, nothing breaks. Frame choice falls back to the last frame of each
second, the loudness trigger never fires, and the colour trigger and the small
model carry on as normal, because both are purely visual by design.

## Where things live

| Path | What it does |
| --- | --- |
| `src/shared/backseatIpc.ts` | All the numbers and the contract between the two halves |
| `src/renderer/lib/backseat/captureWorker.ts` | Ring buffer, frame choice, jolt detection, grid building |
| `src/renderer/lib/backseat/captureController.ts` | Screen and sound capture, clip recorders, trigger timing |
| `src/main/backseat/salienceGate.ts` | The small model and its learned threshold |
| `src/main/backseat/backseatService.ts` | Decides which ticks become speech, runs the large model |
| `src/main/backseat/backseatPrompts.ts` | How the grid is explained, and how the character should talk |
| `src/main/backseatOverlay.ts` | The always on top window, which also owns capture |
| `src/renderer/components/backseat/` | Share picker, overlay, clip card |

The capture lives in the floating overlay window rather than the main app window
on purpose. While you are in a full screen game the main window is hidden, and a
hidden window gets its timers slowed down. The overlay is always visible, so it
keeps running.

## Sources

- Wonkyun Kim et al., *An Image Grid Can Be Worth a Video: Zero-shot Video
  Question Answering Using a VLM*, arXiv:2403.18406.
  https://arxiv.org/abs/2403.18406
- Anthropic, image sizing and token cost.
  https://platform.claude.com/docs/en/build-with-claude/vision
- Electron, `setDisplayMediaRequestHandler` and the note that loopback audio is
  Windows only.
  https://www.electronjs.org/docs/latest/api/session
- Electron issue 47490, on macOS loopback audio.
  https://github.com/electron/electron/issues/47490

## License

AGPL-3.0-or-later, the same as [Sei](https://github.com/sei-studio/sei). See
[LICENSE](LICENSE).
