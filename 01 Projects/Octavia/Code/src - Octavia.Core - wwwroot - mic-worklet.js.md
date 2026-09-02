---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\mic-worklet.js
---

# src\Octavia.Core\wwwroot\mic-worklet.js

```javascript
/* The capture end of the page's microphone, on the audio thread.

   Deliberately the smallest thing that can exist. An `AudioWorkletProcessor` runs on a
   real-time thread where a slow frame is a click in the recording, so this does nothing but
   hand the samples over — every decision about them belongs to `mic.js`, which runs where
   being slow is merely being slow.

   `ScriptProcessorNode` would have been fewer files and is deprecated for exactly the reason
   that matters here: it runs on the main thread, so a busy renderer drops audio she was
   supposed to hear. */
class Capture extends AudioWorkletProcessor {
  process(inputs) {
    const channel = inputs[0]?.[0];

    // A copy, because the block is reused by the engine the moment this returns. Sending
    // the original would deliver whatever the next 128 samples happen to be.
    if (channel && channel.length) this.port.postMessage(channel.slice(0));

    // Never false: returning false ends the processor for good, and a microphone that is
    // muted is still a microphone somebody is about to unmute.
    return true;
  }
}

registerProcessor('octavia-capture', Capture);
```
