# Backseat

**STILL IN DEVELOPMENT NOT TESTED**

A live AI video pipeline that decides what is interesting enough to comment on,
so a character can watch you play and talk about it without talking constantly.

Extracted from [Sei](https://github.com/sei-studio/sei). This is the real
implementation, but it is not a standalone app: it expects an Electron host with
a chat brain, a character persona, and a memory store.

## Architecture

```math
\begin{array}{ccc}
 & \boxed{\text{screen and sound}} & \\
 & \downarrow & \\
 & \boxed{\text{ring buffer: 9 s, one frame per second}} & \\
 & \downarrow & \\
 & \boxed{\text{image grid: 6 frames into 1 picture}} & \\
\downarrow & \downarrow & \downarrow \\
\boxed{\text{you speak or type}} & \boxed{\text{small VLM, every 6 s}} & \boxed{\text{loudness or colour jump}} \\
\downarrow & \downarrow & \downarrow \\
 & \boxed{\text{tick}} & \\
 & \downarrow & \\
 & \boxed{\text{large model}} & \\
 & \downarrow & \\
 & \boxed{\text{short reply, sometimes a saved clip}} &
\end{array}
```

**Ring buffer.** Capture runs at 60 fps but only one frame per second is kept,
chosen by which frame had the loudest sound. It is a running argmax, so one image
is held and one is compressed per second instead of 60, while still examining
every frame. 9 seconds is one grid plus latency slack.

**Image grid.** Still image models cannot watch video, so 6 frames are stitched
into one picture and the model is told to read them in order. Layout, frame count
and ordering follow the IG-VLM paper: 6 frames beat 4, 9, 12, 16 and 20, and near
square grids beat wide ones. The grid is 1204 by 1008 pixels, which is 1548 image
tokens, the largest that fits Claude Haiku 4.5 without being silently downscaled.

**Three triggers.** All three produce the same thing, a *tick*: one grid plus the
reason it fired.

1. You spoke or typed. Always answered. The grid is captured at the moment you
   start, not when you finish, so the character sees what you reacted to.
2. A small VLM is asked every 6 seconds whether something significant happened.
3. A large jump in loudness or screen colour, with no model in the loop.

Triggers 2 and 3 are dropped, never queued, if the character is already thinking
or spoke less than 8 seconds ago.

**Learned threshold.** A fixed cutoff does not work: small vision models say yes
to almost anything. The gate reads the probability of the answer token and sets
its cutoff at the upper quartile of the last 40 scores in the session, so each
game is judged against its own normal. Measured on a static grid versus a
changing grid, the model emitted "no" for both while the scores separated 0.018
against 0.148, so the continuous score is what makes it work at all.

**Sound.** Windows can capture other apps. macOS cannot through this path, so it
falls back to a virtual audio device such as BlackHole if one is installed, and
otherwise runs video only. Without sound, frame choice falls back to the last
frame of each second and the loudness trigger never fires. The grid and the small
VLM are purely visual and unaffected.

## Files

| Path | Role |
| --- | --- |
| `src/shared/backseatIpc.ts` | Constants and the contract between both halves |
| `src/renderer/lib/backseat/captureWorker.ts` | Ring buffer, frame choice, jolt detection, grid building |
| `src/renderer/lib/backseat/captureController.ts` | Screen and sound capture, clip recorders, trigger timing |
| `src/main/backseat/salienceGate.ts` | Small VLM and the learned threshold |
| `src/main/backseat/backseatService.ts` | Decides which ticks become speech, runs the large model |
| `src/main/backseat/backseatPrompts.ts` | Grid explanation and reply style |
| `src/main/backseatOverlay.ts` | Always on top window, which also owns capture |
| `src/renderer/components/backseat/` | Share picker, overlay, clip card |

Capture lives in the overlay window, not the main app window, because a hidden
window has its timers throttled and the main window is hidden during a full
screen game.

## Quickstart

These files drop into an Electron app. To wire them up:

1. Copy `src/` into your project and point the imports at your own chat brain,
   character store, and memory store. `backseatService.ts` is the only file that
   touches them.
2. Register the IPC channels listed at the bottom of `src/shared/backseatIpc.ts`.
3. Add the Chromium switches for system audio in your main process:

   ```js
   app.commandLine.appendSwitch(
     'enable-features',
     'MacSckSystemAudioLoopbackOverride,MacLoopbackAudioForScreenShare',
   );
   ```

4. Set `backgroundThrottling: false` on any window that runs capture.
5. Export a DeepInfra key for the gate. Without it the gate fails closed and
   only triggers 1 and 3 run.

   ```sh
   export SEI_GATE_DEV_KEY=...
   export SEI_GATE_MODEL=Qwen/Qwen3-VL-30B-A3B-Instruct   # optional
   ```

Model choice was measured, not assumed. On DeepInfra, `Qwen3-VL-30B-A3B` answered
correctly on both test grids at about 520 ms and $0.00019 per call, which is about
$0.11 per hour at the 6 second cadence. `gemma-3-4b-it` said yes to everything and
`gemma-3-12b-it` said no to everything.

Run the tests with `vitest`.

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
