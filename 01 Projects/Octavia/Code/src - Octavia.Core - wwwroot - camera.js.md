---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\camera.js
---

# src\Octavia.Core\wwwroot\camera.js

```javascript
/* One still from the camera, on request.

   The face owns the camera rather than the host, for the same reason the page is served
   from a virtual https origin at all: `getUserMedia` needs a secure context, and it was
   set up in Stage 3 knowing this stage was coming. It also puts the camera where the
   *person* is — a face on a wall tablet has one; the machine under the desk may not.

   Three promises this module keeps, and they are the whole design:

     1. It never watches. The device is opened, one frame is taken, and the track is
        stopped in the same breath. There is no stream to leave running. (Watching is a
        separate mode in watch.js, and only a person pressing the camera button starts
        it — code cannot.)
     2. It is never opened unasked. Only a `look` from the host reaches here, and the
        host only sends one when a question cannot be answered without eyes.
     3. It says so. A visible marker appears while the camera is live, because a camera
        that can turn on quietly is a camera nobody should agree to. */

/** Longest edge of the still. Big enough to read a label, small enough to send. */
const EDGE = 768;

let busy = false;

/* Which camera to open. Empty means whichever one the browser would pick, which is
   right until a machine has two of them. Matched by label rather than deviceId: an id
   is regenerated per origin and per permission grant, so it cannot be stored in a
   config file and still mean anything tomorrow. */
let wanted = '';

export function useCamera(label) { wanted = label || ''; }

/** Labels of the cameras attached, for the settings menu. Empty until permission has
    been granted once — the browser withholds labels from an unpermitted page. */
export async function cameras() {
  try {
    const devices = await navigator.mediaDevices?.enumerateDevices?.() ?? [];
    return devices.filter(d => d.kind === 'videoinput' && d.label).map(d => d.label);
  } catch {
    return [];
  }
}

async function constraints() {
  const video = { width: { ideal: 1280 }, height: { ideal: 720 } };
  if (!wanted) return { video, audio: false };

  // Resolve the stored label back to an id at the moment of use.
  try {
    const devices = await navigator.mediaDevices.enumerateDevices();
    const match = devices.find(d => d.kind === 'videoinput' && d.label &&
      (d.label.toLowerCase().includes(wanted.toLowerCase()) ||
       wanted.toLowerCase().includes(d.label.toLowerCase())));
    if (match) video.deviceId = { exact: match.deviceId };
  } catch { /* fall through to the default camera */ }

  return { video, audio: false };
}

export async function takeStill(onLive) {
  if (busy) throw new Error('already looking');
  busy = true;

  let stream = null;
  try {
    if (!navigator.mediaDevices?.getUserMedia) throw new Error('this renderer has no camera API');

    stream = await navigator.mediaDevices.getUserMedia(await constraints());

    onLive(true);

    const video = document.createElement('video');
    video.srcObject = stream;
    video.muted = true;
    video.playsInline = true;
    await video.play();

    /* Let the camera settle before taking the picture.

       Two animation frames were enough against a synthetic device and far too few
       against a real one: the first still off a redirected USB webcam came back at
       0.15 brightness, because auto-exposure and auto-white-balance had not finished.
       Real sensors need a few hundred milliseconds, and this is the difference between
       a photograph of a room and a photograph of the dark.

       It is also why `Glance` exists on the host side — that number is what made this
       visible at all. */
    await new Promise(resolve => setTimeout(resolve, 450));
    await new Promise(resolve => requestAnimationFrame(resolve));

    const w = video.videoWidth, h = video.videoHeight;
    if (!w || !h) throw new Error('the camera opened but produced no frame');

    const scale = Math.min(1, EDGE / Math.max(w, h));
    const canvas = document.createElement('canvas');
    canvas.width = Math.round(w * scale);
    canvas.height = Math.round(h * scale);
    canvas.getContext('2d').drawImage(video, 0, 0, canvas.width, canvas.height);

    video.pause();
    video.srcObject = null;

    // Only the payload: the host wants base64, not a data URL.
    return canvas.toDataURL('image/jpeg', 0.82).split(',')[1];
  } finally {
    // In `finally` so a throw anywhere above still closes the device. A camera left on
    // by an error path is exactly the failure this module exists to make impossible.
    if (stream) stream.getTracks().forEach(track => track.stop());
    onLive(false);
    busy = false;
  }
}
```
