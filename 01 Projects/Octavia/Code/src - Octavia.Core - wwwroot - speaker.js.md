---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\speaker.js
---

# src\Octavia.Core\wwwroot\speaker.js

```javascript
/* Her voice, played by the page — Stage 15 item 3, the other half.

   The microphone went up from the renderer in v0.34.0; this is the same journey in the other
   direction. Until now her voice came out of the **server's** sound card, which is the second
   of the two device hooks the owner's rule removes — *"the server should have no hook on any
   device."*

   A handset has done this since item 9, natively. This is the same thing for a renderer with
   no shell to do it for them, and it needs nothing new on the wire: **a binary frame from the
   host is her voice**, raw PCM in the format `hello` advertises, no header and no type tag.

   **Why raw PCM and not a codec:** that decision is Stage 3's and is written down in
   `PROTOCOL.md`. What matters here is the consequence — there is nothing to decode, so a
   frame can be scheduled the moment it lands, and the only real work is not letting the
   scheduling drift. */

/// How far ahead of the clock the first frame is placed.
///
/// A frame scheduled at exactly `currentTime` is already late by the time the graph runs, and
/// arrives as a click. This is the smallest cushion that reliably survives a garbage
/// collection between one frame and the next; it is also the whole of the latency this adds.
const LEAD_S = 0.06;

/// When the queue is considered to have run dry.
///
/// Sentences arrive with gaps — she is generating them. A gap is not a fault and must not
/// reset the cushion, or every sentence after the first starts with a click. This says how
/// far behind the clock the cursor has to fall before it is treated as a new utterance
/// rather than as a late frame.
const RESTART_AFTER_S = 0.25;

export async function createSpeaker({ sampleRate, channels, bits }) {
  if (bits !== 16) throw new Error(`only 16-bit audio is understood, not ${bits}-bit`);

  /* The context is created at *her* sample rate, so nothing resamples.

     Asking for a rate the device cannot do makes the browser resample instead, which is
     fine — it is the same conversion this file would otherwise have to do, done by
     somebody who has already got it right. */
  const context = new AudioContext({ sampleRate });

  // A page that has not been clicked yet gets a suspended context. Resuming without a
  // gesture is refused in a browser and allowed in her own client, which passes the flag
  // for it — so this is a request, not an assertion.
  if (context.state === 'suspended') {
    try { await context.resume(); } catch { /* stays suspended; `ready` says so */ }
  }

  let cursor = 0;

  /* Every frame still scheduled or playing.

     Kept because **hush has to be immediate** and there is no other way to reach a source
     that has already been handed to the graph. Sources remove themselves on `ended`, so this
     is a few hundred milliseconds of them at most, not a leak. */
  let scheduled = new Set();

  return {
    /* **The honest test, and the reason it is separate from "did the page try".**

       An `AudioContext` that will not resume plays nothing, silently and for ever. If the
       page subscribed to her voice on the strength of having *asked* for a speaker, the
       server would stop using its own sound card and she would simply go quiet — which is
       the same fault the microphone fallback was rebuilt to avoid, in the other direction.
       So the subscription waits on this. */
    get ready() { return context.state === 'running'; },

    /// One binary frame. Interleaved, little-endian, and already in her format.
    play(buffer) {
      if (context.state !== 'running') return;

      const samples = new Int16Array(buffer);
      const frames = Math.floor(samples.length / channels);
      if (frames === 0) return;

      const block = context.createBuffer(channels, frames, sampleRate);

      for (let channel = 0; channel < channels; channel++) {
        const out = block.getChannelData(channel);
        for (let i = 0; i < frames; i++) {
          const sample = samples[i * channels + channel];
          // Asymmetric, because two's complement is: -32768 is a real sample and dividing
          // it by 32767 would push it past -1 and clip on the way out.
          out[i] = sample < 0 ? sample / 0x8000 : sample / 0x7fff;
        }
      }

      const source = context.createBufferSource();
      source.buffer = block;
      source.connect(context.destination);

      const now = context.currentTime;
      if (cursor < now - RESTART_AFTER_S) cursor = now + LEAD_S;
      else if (cursor < now) cursor = now;

      scheduled.add(source);
      source.addEventListener('ended', () => scheduled.delete(source));

      source.start(cursor);
      cursor += block.duration;
    },

    /// Everything queued, dropped. **Hush has to be immediate**: frames already scheduled
    /// would otherwise go on playing a sentence she has been told to stop, which is the one
    /// thing that makes an interruption feel broken rather than obeyed.
    silence() {
      cursor = 0;

      for (const source of scheduled) {
        // Stopping a source that has already ended throws in some engines and is a no-op in
        // others; either way it is not worth interrupting the rest of the queue for.
        try { source.stop(); } catch { /* already finished */ }
      }

      scheduled.clear();
    },

    async close() {
      if (context.state !== 'closed') await context.close();
    }
  };
}
```
