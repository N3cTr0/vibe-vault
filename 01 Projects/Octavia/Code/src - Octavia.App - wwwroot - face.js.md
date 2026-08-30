---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\face.js
---

# src\Octavia.App\wwwroot\face.js

```javascript
/* The face: the scene, the timing, and the puppet strings.

   It knows nothing about the host — everything reaches it through window.Face — and
   almost nothing about the avatar. Blink schedules, saccades, head carriage and mood
   live here; how a jaw or a cheek actually moves lives in the avatar. That seam is what
   lets a plaster bust and a photoreal character take the same performance. */
import * as THREE from './lib/three.module.js';
import { createBust } from './bust.js';
import { createEnvironment } from './environment.js';

const canvas = document.getElementById('scene');
const stage = document.getElementById('stage');

const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.06;
// The backdrop clears the frame itself, so the main render must not clear over it.
renderer.autoClear = false;

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(32, 1, 0.1, 60);

/* The room hands the page its share of the hour, and the page wears it through three
   custom properties. Stage 10's "the chrome sits in her light": before this the window
   was fixed grey above a room that moved through a whole day. */
const chromeStyle = document.documentElement.style;
let lightScale = 1;

/* Declared before the environment, because creating it applies the day immediately and
   that calls straight back into the palette handler below. Left as `let avatar = ...`
   after the call, the handler reaches a binding that does not exist yet and the whole
   scene fails to build. */
let avatar;
const environment = createEnvironment(scene, {
  onPalette: ({ tint, ink, line, avatarScale }) => {
    chromeStyle.setProperty('--room-tint', tint);
    chromeStyle.setProperty('--room-ink', ink);
    chromeStyle.setProperty('--room-line', line);

    // How much of the room's light this character should take. See environment.js.
    lightScale = avatarScale;
    avatar?.setLightScale(avatarScale);
  }
});

avatar = createBust();
scene.add(avatar.root);
avatar.setLightScale(lightScale);

/* Where the camera sits for the bust. A VRM replaces this with a framing computed
   from its own head bone, so a tall character and a short one look the same size. */
const BUST_FRAME = { target: new THREE.Vector3(0, -0.35, 0), distance: 6.4, height: 0.10, offset: 0.55 };
let frame = BUST_FRAME;

function resize() {
  const w = stage.clientWidth, h = stage.clientHeight;
  if (!w || !h) return;
  renderer.setSize(w, h, false);
  camera.aspect = w / h;

  const fit = Math.max(1, 1.42 / Math.min(camera.aspect, 1.3));
  const offset = (frame.offset ?? 0) / fit;
  camera.position.set(
    frame.target.x + offset,
    frame.target.y + (frame.height ?? 0),
    frame.target.z + frame.distance * fit);
  camera.updateProjectionMatrix();
  camera.lookAt(frame.target);
  environment.setAspect(Math.max(camera.aspect, 0.6));
}
new ResizeObserver(resize).observe(stage);
resize();

const reduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
avatar.setBreathing(!reduced);

const rig = {
  state: 'idle',
  expression: 'neutral', expressionWeight: 0,
  lid: 0, lidTarget: 0, nextBlink: 2 + Math.random() * 3,
  gaze: new THREE.Vector2(), gazeTarget: new THREE.Vector2(), nextSaccade: 1.5,
  yaw: 0, pitch: 0, roll: 0,
  level: 0, sinceLevel: 0,
  open: 0,

  /* Music. `beat` is an impulse the host sets to 1 and this decays, so the movement is
     driven by beats actually detected rather than by a clock running here — the host is
     the one listening, and a face that guessed would drift out of time. */
  music: false, bpm: 0, energy: 0,
  beat: 0, beats: 0, sway: 0, headphones: 0,

  /* Set by the dev panel to drive something the host would normally own. Null means
     "nobody is holding this", which is the state everything ships in. */
  holdGaze: false, propHeadphones: null
};

const lerp = (a, b, t) => a + (b - a) * t;
const clock = new THREE.Clock();

function tick() {
  requestAnimationFrame(tick);
  const dt = Math.min(clock.getDelta(), 0.05);
  const t = clock.elapsedTime;

  /* blink */
  rig.nextBlink -= dt;
  if (rig.nextBlink <= 0) {
    rig.lidTarget = 1;
    setTimeout(() => { rig.lidTarget = 0; }, 92);
    rig.nextBlink = 2.2 + Math.random() * 4.2;
  }
  rig.lid = lerp(rig.lid, rig.lidTarget, 1 - Math.pow(0.00002, dt));
  avatar.setBlink(THREE.MathUtils.clamp(rig.lid, 0, 1));

  /* gaze: where she looks says as much about her state as her face does */
  rig.nextSaccade -= dt;
  if (rig.nextSaccade <= 0 && !rig.holdGaze) {
    if (rig.state === 'thinking') {
      rig.gazeTarget.set((Math.random() - 0.4) * 0.34, 0.13 + Math.random() * 0.16);
      rig.nextSaccade = 0.5 + Math.random() * 0.8;
    } else if (rig.state === 'listening') {
      rig.gazeTarget.set((Math.random() - 0.5) * 0.06, (Math.random() - 0.5) * 0.05);
      rig.nextSaccade = 1.4 + Math.random() * 1.6;
    } else {
      rig.gazeTarget.set((Math.random() - 0.5) * 0.24, (Math.random() - 0.5) * 0.14);
      rig.nextSaccade = 1.0 + Math.random() * 2.4;
    }
  }
  rig.gaze.lerp(rig.gazeTarget, 1 - Math.pow(0.0001, dt));
  avatar.setGaze(rig.gaze.x, rig.gaze.y);

  /* music: headphones on, and a beat to move to */
  rig.beat *= Math.pow(0.0009, dt);
  const wearing = rig.propHeadphones ?? (rig.music ? 1 : 0);
  rig.headphones = lerp(rig.headphones, wearing, 1 - Math.pow(0.06, dt));
  avatar.setHeadphones(reduced ? wearing : rig.headphones);

  // A sway every four beats rather than every beat: moving to the bar is what reading
  // as dancing looks like, and per-beat sway reads as a twitch.
  const swayTarget = rig.music ? Math.sin(rig.beats * Math.PI / 2) : 0;
  rig.sway = lerp(rig.sway, swayTarget, 1 - Math.pow(0.02, dt));

  /* head carriage */
  const breathe = reduced ? 0 : 1;
  let yawT = 0.115 + rig.gaze.x * 0.30;
  let pitT = -rig.gaze.y * 0.18;
  let rolT = 0;
  if (rig.state === 'listening') { rolT = 0.055; pitT += 0.035; }
  if (rig.state === 'thinking') { yawT += 0.10; rolT = -0.045; pitT -= 0.05; }
  if (rig.state === 'speaking') { pitT += Math.sin(t * 2.3) * 0.018 * breathe; }
  yawT += Math.sin(t * 0.41) * 0.030 * breathe;
  pitT += Math.sin(t * 0.33 + 1.7) * 0.020 * breathe;

  // The nod lands on the beat and the sway carries the bar. Both scale with energy, so
  // a quiet passage is a smaller movement rather than the same one turned off.
  if (rig.music && !reduced) {
    const force = 0.35 + rig.energy * 0.65;
    pitT += rig.beat * 0.085 * force;
    rolT += rig.sway * 0.075 * force;
    yawT += rig.sway * 0.10 * force;
  }

  rig.yaw = lerp(rig.yaw, yawT, 1 - Math.pow(0.02, dt));
  rig.pitch = lerp(rig.pitch, pitT, 1 - Math.pow(0.02, dt));
  rig.roll = lerp(rig.roll, rolT, 1 - Math.pow(0.02, dt));
  avatar.setPose(rig.yaw, rig.pitch, rig.roll);

  /* the host only sends level while it is metering; decay in between so a dropped
     stream relaxes the room instead of freezing it */
  rig.sinceLevel += dt;
  if (rig.sinceLevel > 0.25) rig.level *= Math.pow(0.02, dt);
  avatar.setLevel(rig.level);
  environment.setLevel(rig.level);

  const busy = rig.state !== 'idle';
  const spin = rig.state === 'thinking' ? 2.6
    : rig.state === 'listening' ? 0.7 + rig.level * 3.2
    : rig.state === 'speaking' ? 0.5 + rig.open * 1.6 : 0.25;
  const span = THREE.MathUtils.clamp(
    rig.state === 'listening' ? 0.22 + rig.level * 1.9 : 0.62, 0.12, 2.6);
  avatar.setActivity(busy, spin, span);

  environment.setMusic(rig.music ? rig.energy : 0, rig.beat);
  environment.update(dt, t);
  avatar.update(dt, t);
  renderer.clear();
  environment.render(renderer);
  renderer.render(scene, camera);
}
tick();

/* Swapping the avatar is deliberately a small, ordinary function: it is the whole
   reason the interface exists, and Stage 8's photoreal renderer arrives the same way. */
function adopt(next, nextFrame) {
  scene.remove(avatar.root);
  avatar = next;
  frame = nextFrame ?? BUST_FRAME;
  scene.add(avatar.root);
  avatar.setBreathing(!reduced);
  avatar.setExpression(rig.expression, rig.expressionWeight);
  avatar.setHeadphones(rig.headphones);
  avatar.setLightScale(lightScale);
  resize();
}

window.Face = {
  setState(state) {
    rig.state = state;
    document.body.dataset.state = state;
    if (state !== 'speaking') {
      rig.open = 0;
      avatar.setViseme(null, 0);
    }
    // A state is not a mood, but an unstated mood should follow the state rather than
    // leaving her blank through a whole conversation.
    if (rig.expressionWeight === 0) {
      avatar.setExpression(state === 'thinking' ? 'relaxed' : 'neutral', state === 'thinking' ? 0.3 : 0);
    }
  },

  setLevel(value) {
    rig.level = THREE.MathUtils.clamp(value, 0, 1);
    rig.sinceLevel = 0;
  },

  /** `shape` is a VRM viseme name; older hosts send openness alone, so it may be absent. */
  setViseme(value, shape) {
    rig.open = THREE.MathUtils.clamp(value, 0, 1);
    avatar.setViseme(rig.open > 0.001 ? (shape || 'aa') : null, rig.open);
  },

  setEmotion(name, weight) {
    rig.expression = name || 'neutral';
    rig.expressionWeight = THREE.MathUtils.clamp(weight ?? 1, 0, 1);
    avatar.setExpression(rig.expression, rig.expressionWeight);
  },

  /** Pin the room's clock: Face.setHour(21) to see the evening at breakfast. */
  setHour(hour) { environment.setHour(hour); },

  /**
   * What the machine is playing. `beat` messages carry nothing but the beat itself and
   * arrive on their own, so they are the one field that may appear without the others.
   */
  setMusic(music) {
    if (music.beat) {
      rig.beat = 1;
      rig.beats++;
      return;
    }

    rig.music = !!music.playing;
    rig.bpm = music.bpm || 0;
    rig.energy = THREE.MathUtils.clamp(music.energy ?? 0, 0, 1);
    if (!rig.music) rig.beats = 0;
  },

  /** Swap in a VRM character. Falls back to the bust, loudly, rather than to nothing. */
  async loadAvatar(url) {
    const { loadVrmAvatar } = await import('./vrm-avatar.js');
    const next = await loadVrmAvatar(url);
    adopt(next, next.frame);
    return next.vrm?.meta ?? null;
  },

  useBust() { adopt(createBust(), BUST_FRAME); },

  /* ── the handles the dev panel needs ──────────────────────
     Everything above is driven by the host. These three are things the face schedules
     for itself, so there is otherwise no way to ask for one on demand. */

  /** Blink now, and restart the ordinary schedule afterwards. */
  blink() {
    rig.lidTarget = 1;
    setTimeout(() => { rig.lidTarget = 0; }, 92);
    rig.nextBlink = 2.2 + Math.random() * 4.2;
  },

  /** Aim her gaze and hold it. `Face.look(null)` returns her to her own saccades. */
  look(x, y) {
    rig.holdGaze = x !== null && x !== undefined;
    if (rig.holdGaze) rig.gazeTarget.set(x, y);
    else rig.nextSaccade = 0;
  },

  /** Wear a prop regardless of what the room is doing. `null` gives it back to the music. */
  setProp(name, on) {
    if (name !== 'headphones') return false;
    rig.propHeadphones = on === null || on === undefined ? null : (on ? 1 : 0);
    return true;
  },

  /** What the panel should show as current, so opening it does not reset her. */
  read() {
    return {
      state: rig.state,
      expression: rig.expression, expressionWeight: rig.expressionWeight,
      level: rig.level, music: rig.music, bpm: rig.bpm, energy: rig.energy,
      headphones: rig.propHeadphones, holdGaze: rig.holdGaze, reduced
    };
  }
};
```
