---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\watch.js
---

# src\Octavia.Core\wwwroot\watch.js

```javascript
/* She looks at you.

   The one place she watches rather than glances — and it is bought with a click on the
   camera button, never started by code. The button is the same contract as the
   microphone's: a person presses it, a visible marker stays up the whole time, and
   pressing it again ends it.

   Everything stays inside this renderer. Frames are read into a canvas the size of a
   postage stamp, reduced to "where did something move", and thrown away — no frame, no
   coordinate, nothing at all leaves the page. The host is not even told; as far as the
   protocol is concerned this mode does not exist.

   The tracking is deliberately NOT a face detector. A motion centroid over a 64x36
   grid is thirty lines a person can read, needs no vendored model, and fails the way a
   person does: when nothing moves, she keeps looking where she last saw you. A real
   detector would be more precise and would bring megabytes of somebody else's code to
   a job that only has to feel attentive. */

const W = 64, H = 36;

/** How different a cell must be between frames to count as movement rather than
    sensor noise, and how much total movement means a person rather than a flicker. */
const CELL_STEP = 0.06;
const ENOUGH = 0.8;

export function createWatcher({ onGaze, onLive }) {
  let stream = null;
  let video = null;
  let timer = 0;
  let previous = null;

  // Smoothed centroid in frame fractions. Starts at "a face-height point in the
  // middle", which is where a seated person almost always is.
  let x = 0.5, y = 0.42;

  async function start() {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { width: { ideal: 320 }, height: { ideal: 180 } },
      audio: false
    });

    video = document.createElement('video');
    video.srcObject = stream;
    video.muted = true;
    video.playsInline = true;
    await video.play();

    const canvas = document.createElement('canvas');
    canvas.width = W;
    canvas.height = H;
    const ctx = canvas.getContext('2d', { willReadFrequently: true });

    onLive(true);

    // ~8 Hz. Eyes only need to keep up with a person shifting in a chair, and the
    // whole pass is a few thousand subtractions.
    timer = setInterval(() => {
      if (!video.videoWidth) return;

      ctx.drawImage(video, 0, 0, W, H);
      const data = ctx.getImageData(0, 0, W, H).data;

      const grey = new Float32Array(W * H);
      for (let i = 0; i < W * H; i++) {
        grey[i] = (data[i * 4] + data[i * 4 + 1] + data[i * 4 + 2]) / 765;
      }

      if (previous) {
        let sum = 0, sx = 0, sy = 0;
        for (let i = 0; i < grey.length; i++) {
          const d = Math.abs(grey[i] - previous[i]);
          if (d > CELL_STEP) {
            sum += d;
            sx += (i % W) * d;
            sy += ((i / W) | 0) * d;
          }
        }

        if (sum > ENOUGH) {
          x += (sx / sum / W - x) * 0.25;
          y += (sy / sum / H - y) * 0.25;

          /* Mirrored, like looking in a mirror: move to your left and her eyes go to
             your left. The vertical span is smaller because standing up should raise
             her gaze, not roll her eyes at the ceiling. */
          onGaze((0.5 - x) * 0.8, (0.42 - y) * 0.55);
        }
      }

      previous = grey;
    }, 120);
  }

  function stop() {
    clearInterval(timer);
    timer = 0;

    if (video) {
      video.pause();
      video.srcObject = null;
      video = null;
    }

    // The same rule as the still: whatever happened above, the device ends up closed.
    if (stream) {
      stream.getTracks().forEach(track => track.stop());
      stream = null;
    }

    previous = null;
    onLive(false);
  }

  return { start, stop };
}
```
