---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\mic.js
---

# src\Octavia.Core\wwwroot\mic.js

```javascript
/* The page's own microphone — Stage 15 item 3, the client half.

   **Until now the face owned no audio**, which is the first line of `bridge.js` and was true
   in both directions: her voice came out of the server's sound card, and the only microphone
   she had was the server's own. That is exactly the arrangement the owner's rule removes —
   *"the server should have no hook on any device; the phone sends its mic to the server, the
   Windows client should be doing the same thing."*

   A handset already does this natively, through `window.OctaviaEmbedder`. This is the same
   thing for a renderer that has no embedder to borrow from: **the page captures, and streams
   it up itself.**

   **Nothing on the wire changed to allow it.** `talking` takes the floor and a binary frame
   from a face is microphone audio, 16 kHz 16-bit mono little-endian, fixed by contract since
   Stage 3. The desktop is not a new kind of client; it is finally the same kind as the phone.

   Loaded on demand, like `camera.js`: a face that never listens never pays for this. */

/// 20 ms at 16 kHz. Small enough that letting go feels immediate, large enough that a
/// WebSocket frame is worth its own header — the worklet delivers 128 samples at a time,
/// which is 8 ms and 256 bytes, and sending each one separately is mostly framing.
const FRAME_SAMPLES = 320;

/// How long after she stops speaking before the microphone is trusted again.
///
/// Her voice leaves the sound card, crosses a room and comes back in; `state: idle` says the
/// *stream* ended, not that the air is quiet. Without this her last syllable is transcribed
/// as the beginning of the next thing somebody said.
const TAIL_MS = 250;

export async function createMicrophone({ send, onLevel, onError }) {
  /* Browser echo cancellation, asked for explicitly and worth naming.

     Her voice plays out of the same machine this microphone is on, so without AEC an open
     microphone hears her and she answers herself. Chromium's canceller is good and it is
     right here — this is the textbook case it was built for, a loudspeaker and a microphone
     on one device. It is not a substitute for the mute below; it is the thing that makes the
     mute's edges forgiving rather than critical. */
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: {
      channelCount: 1,
      echoCancellation: true,
      noiseSuppression: true,
      autoGainControl: true
    }
  });

  /* Asking for 16 kHz rather than resampling to it.

     Chromium honours a requested sample rate and does the conversion in the graph, which is
     better than anything written here would be. The contract is 16 kHz because that is what
     Whisper wants; getting there by asking costs nothing and removes the one piece of signal
     processing this file would otherwise have to be right about. */
  const context = new AudioContext({ sampleRate: 16000 });

  try {
    await context.audioWorklet.addModule('./mic-worklet.js');
  } catch (err) {
    stream.getTracks().forEach(t => t.stop());
    await context.close();
    throw err;
  }

  const source = context.createMediaStreamSource(stream);
  const capture = new AudioWorkletNode(context, 'octavia-capture');

  source.connect(capture);

  /* Connected to nothing.

     An `AudioWorkletNode` with no destination still runs — the graph pulls it because it has
     an input. Connecting it to `context.destination` would work identically *and put the
     microphone through the speakers*, which is a feedback loop and an unpleasant way to find
     out. */

  let streaming = false;
  let muted = false;
  let unmuteAt = 0;
  let pending = new Float32Array(FRAME_SAMPLES);
  let filled = 0;
  let peak = 0;
  let levelSentAt = 0;

  capture.port.onmessage = event => {
    const block = event.data;
    if (!streaming) return;

    // Muting drops samples rather than closing the device. Reopening costs a slow
    // re-acquire and a click, and she is only ever muted for as long as she is talking.
    if (muted || performance.now() < unmuteAt) { filled = 0; peak = 0; return; }

    for (let i = 0; i < block.length; i++) {
      const sample = block[i];
      pending[filled++] = sample;

      const size = sample < 0 ? -sample : sample;
      if (size > peak) peak = size;

      if (filled === FRAME_SAMPLES) {
        flush();
        filled = 0;
      }
    }

    // The face reacts while somebody is still speaking, which is what `level` does when the
    // *host* owns the microphone. Twenty a second, as the protocol says.
    const now = performance.now();
    if (onLevel && now - levelSentAt >= 50) {
      levelSentAt = now;
      onLevel(Math.min(1, peak * 1.8));
      peak = 0;
    }
  };

  function flush() {
    const pcm = new Int16Array(FRAME_SAMPLES);

    for (let i = 0; i < FRAME_SAMPLES; i++) {
      // Clamped before scaling. A sample above 1.0 wraps to a large negative number
      // otherwise, which is heard as a crack rather than as clipping.
      const sample = Math.max(-1, Math.min(1, pending[i]));
      pcm[i] = sample < 0 ? sample * 0x8000 : sample * 0x7fff;
    }

    try {
      send(pcm.buffer);
    } catch (err) {
      if (onError) onError(err);
    }
  }

  return {
    /// Whether the device is open, whatever it is currently doing with the samples.
    get open() { return context.state !== 'closed'; },

    get streaming() { return streaming; },

    async start() {
      // A context created before any gesture starts suspended; the click that got here is
      // that gesture, so this resolves immediately in practice.
      if (context.state === 'suspended') await context.resume();
      filled = 0;
      peak = 0;
      streaming = true;
    },

    stop() {
      streaming = false;
      filled = 0;
    },

    /// Silence while she speaks, with a tail afterwards.
    mute(on) {
      muted = on;
      if (!on) unmuteAt = performance.now() + TAIL_MS;
      else filled = 0;
    },

    async close() {
      streaming = false;
      capture.port.onmessage = null;
      source.disconnect();
      capture.disconnect();
      stream.getTracks().forEach(track => track.stop());
      if (context.state !== 'closed') await context.close();
    }
  };
}
```
