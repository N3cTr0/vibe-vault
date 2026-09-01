---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\environment.js
---

# src\Octavia.Core\wwwroot\environment.js

```javascript
/* The room she is in: backdrop, parallax depth, and the light rig.

   Lighting and backdrop belong together — a warm evening key over a midday wall looks
   wrong immediately — so one module owns both and moves them through the day together.
   Everything here is a shader and three planes: it costs almost nothing, which matters
   because the GPU has a photoreal renderer and a speech model in its future. */
import * as THREE from './lib/three.module.js';

/* Keyframes through the day. `at` is the hour; the two bracketing frames are blended,
   so the room is never the same twice and never jumps. */
/* She stands in a room, not outdoors: the wall stays plaster all day and the hours
   move its temperature and brightness, not its species. A backdrop that turns sky-blue
   at noon reads as a different place rather than a different time. */
/* `chrome`, `chromeAlpha` and `chromeInk` are the page's share of the same palette:
   the tint the header and console wear at that hour, and the ink that stays legible on
   it. They live here rather than in face.css because a room and the window around it
   drifting apart is exactly the fault Stage 10 set out to fix. */
const DAY = [
  { at: 0,  chrome: 0x1A1F30, chromeAlpha: 0.62, chromeInk: 0xDDE2F0, top: 0x141824, bottom: 0x232838, glow: 0x2F4CD0, key: 0x7E93CE, keyLevel: 0.62, sky: 0x24304A, ground: 0x14192A, ambient: 0.34, rim: 0x8FA6E8, rimLevel: 0.55 },
  { at: 6,  chrome: 0xD98A50, chromeAlpha: 0.12, chromeInk: 0xF4EEE4, top: 0x4A4550, bottom: 0x8E7361, glow: 0xD98A50, key: 0xFFC49A, keyLevel: 1.30, sky: 0x9B8790, ground: 0x3B3540, ambient: 0.52, rim: 0xA8B6F0, rimLevel: 0.70 },
  { at: 9,  chrome: 0xFFFCF2, chromeAlpha: 0.34, chromeInk: 0x242219, top: 0xB6B0A2, bottom: 0xE6DFD0, glow: 0x2F4CD0, key: 0xFFF4E2, keyLevel: 2.05, sky: 0xF3EEE2, ground: 0x8B8676, ambient: 0.72, rim: 0xBFD0FF, rimLevel: 0.85 },
  { at: 13, chrome: 0xFFFCF2, chromeAlpha: 0.38, chromeInk: 0x242219, top: 0xC4BEAE, bottom: 0xEFE9DC, glow: 0x3350C4, key: 0xFFFDF6, keyLevel: 2.20, sky: 0xF6F2E8, ground: 0x938E7E, ambient: 0.80, rim: 0xC8D6FF, rimLevel: 0.80 },
  { at: 18, chrome: 0xE8823C, chromeAlpha: 0.14, chromeInk: 0x2A2118, top: 0x6E5C55, bottom: 0xC28D62, glow: 0xE8823C, key: 0xFFB273, keyLevel: 1.55, sky: 0xC0A093, ground: 0x4A3F3E, ambient: 0.56, rim: 0x9AA8E0, rimLevel: 0.72 },
  { at: 21, chrome: 0x3A3648, chromeAlpha: 0.46, chromeInk: 0xEDE7DC, top: 0x33323F, bottom: 0x50484F, glow: 0x6A64C8, key: 0xB3A6D2, keyLevel: 0.92, sky: 0x4A4560, ground: 0x241F30, ambient: 0.44, rim: 0x92A2E4, rimLevel: 0.62 },
  { at: 24, chrome: 0x1A1F30, chromeAlpha: 0.62, chromeInk: 0xDDE2F0, top: 0x141824, bottom: 0x232838, glow: 0x2F4CD0, key: 0x7E93CE, keyLevel: 0.62, sky: 0x24304A, ground: 0x14192A, ambient: 0.34, rim: 0x8FA6E8, rimLevel: 0.55 }
];

const VERT = `
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}`;

const FRAG = `
varying vec2 vUv;
uniform vec3 uTop;
uniform vec3 uBottom;
uniform vec3 uGlow;
uniform float uTime;
uniform float uLevel;
uniform float uDrift;
uniform float uAspect;
uniform float uMusic;
uniform float uBeat;

void main() {
  vec3 col = mix(uBottom, uTop, smoothstep(0.0, 1.0, vUv.y));

  // The halo she stands in. It breathes with the microphone, which is what makes the
  // room feel like it is listening rather than merely lit.
  vec2 p = vUv - vec2(0.5 + uDrift * 0.04, 0.52);
  p.x *= uAspect;
  float d = length(p);
  col += uGlow * exp(-d * d * 11.0) * (0.13 + uLevel * 0.45 + uMusic * 0.30 + uBeat * uMusic * 0.55);

  // A ring that leaves on the beat. It is drawn on top of the halo rather than instead
  // of it, so the room keeps its light between beats instead of strobing.
  if (uMusic > 0.001) {
    float ring = exp(-pow((d - (1.0 - uBeat) * 0.62) * 9.0, 2.0));
    col += uGlow * ring * uBeat * uMusic * 0.22;
  }

  // Two very slow bands: enough gradient movement that the wall is never flat.
  float band = sin(vUv.y * 6.0 + uTime * 0.05) + sin(vUv.x * 3.0 - uTime * 0.031);
  col *= 1.0 + band * 0.010;

  // smoothstep wants its edges in ascending order; reversing them is undefined, which
  // is a silent way to get no vignette at all.
  float vig = 1.0 - smoothstep(0.30, 0.92, length((vUv - 0.5) * vec2(uAspect * 0.72, 1.0)));
  col *= mix(0.58, 1.0, vig);

  // Grain, matching the paper texture the page overlays on everything else.
  float n = fract(sin(dot(vUv * vec2(432.1, 913.7), vec2(12.9898, 78.233)) + uTime * 0.4) * 43758.5453);
  col += (n - 0.5) * 0.014;

  gl_FragColor = vec4(col, 1.0);
}`;

function frames(hour) {
  for (let i = 0; i < DAY.length - 1; i++) {
    if (hour >= DAY[i].at && hour <= DAY[i + 1].at) {
      const span = DAY[i + 1].at - DAY[i].at;
      return { a: DAY[i], b: DAY[i + 1], t: span > 0 ? (hour - DAY[i].at) / span : 0 };
    }
  }
  return { a: DAY[0], b: DAY[0], t: 0 };
}

export function createEnvironment(scene, options = {}) {
  const colour = (hex) => new THREE.Color(hex);

  const uniforms = {
    uTop: { value: colour(0xA9C0DA) },
    uBottom: { value: colour(0xE4DCCB) },
    uGlow: { value: colour(0x2F4CD0) },
    uTime: { value: 0 },
    uLevel: { value: 0 },
    uDrift: { value: 0 },
    uAspect: { value: 1.6 },
    uMusic: { value: 0 },
    uBeat: { value: 0 }
  };

  /* The wall is a fullscreen quad drawn before the room, not a plane standing in it.
     A plane has edges, and at some window shape they always come into frame; this also
     spares the gradient from being warped by perspective. */
  const backdropScene = new THREE.Scene();
  const backdropCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  backdropScene.add(new THREE.Mesh(
    new THREE.PlaneGeometry(2, 2),
    new THREE.ShaderMaterial({ vertexShader: VERT, fragmentShader: FRAG, uniforms, depthTest: false, depthWrite: false })
  ));

  /* Depth. Two soft slabs between the wall and her, drifting at different rates — the
     cheapest convincing parallax there is, and it survives any avatar. They must have
     no edge at all: a rectangle with a visible border reads as a mistake, not a shadow. */
  const slabs = [-7.2, -4.6].map((z, i) => {
    const mesh = new THREE.Mesh(
      new THREE.PlaneGeometry(10 - i * 2.6, 24),
      new THREE.ShaderMaterial({
        transparent: true,
        depthWrite: false,
        uniforms: { uStrength: { value: 0.085 + i * 0.055 } },
        vertexShader: VERT,
        fragmentShader: `
          varying vec2 vUv;
          uniform float uStrength;
          void main() {
            float edge = smoothstep(0.0, 0.42, vUv.x) * smoothstep(1.0, 0.58, vUv.x)
                       * smoothstep(0.0, 0.30, vUv.y) * smoothstep(1.0, 0.70, vUv.y);
            gl_FragColor = vec4(0.03, 0.035, 0.05, edge * uStrength);
          }`
      })
    );
    mesh.position.set(i === 0 ? -4.6 : 5.4, 0, z);
    mesh.renderOrder = -1;
    scene.add(mesh);
    return { mesh, home: mesh.position.x, z, rate: 0.031 + i * 0.019, reach: 0.55 + i * 0.75 };
  });

  /* Dust in the key light.
     Two slabs drifting behind her give depth only when something moves *across* them,
     and until there is a reason for the camera to move there is nothing to parallax
     against. A few hundred motes between her and the lens supply that reason: they are
     nearer than she is, so they slide further per frame, and the eye reads the gap as
     room. One Points draw of additive dots, which matters on a GT 730.

     They drift on a sine rather than being simulated. Nothing here is physics — it is a
     suggestion of moving air, and a suggestion costs one multiply per axis. */
  const MOTES = 260;
  const motePositions = new Float32Array(MOTES * 3);
  const motePhase = new Float32Array(MOTES * 3);

  for (let i = 0; i < MOTES; i++) {
    motePositions[i * 3] = (Math.random() - 0.5) * 9;
    motePositions[i * 3 + 1] = (Math.random() - 0.5) * 6.4;
    motePositions[i * 3 + 2] = -1.4 + Math.random() * 3.4;
    motePhase[i * 3] = Math.random() * Math.PI * 2;
    motePhase[i * 3 + 1] = Math.random() * Math.PI * 2;
    motePhase[i * 3 + 2] = 0.12 + Math.random() * 0.5;   // this mote's own rate
  }

  const moteGeometry = new THREE.BufferGeometry();
  moteGeometry.setAttribute('position', new THREE.BufferAttribute(motePositions, 3));

  const moteMaterial = new THREE.PointsMaterial({
    color: 0xFFF3DE,
    size: 0.028,
    sizeAttenuation: true,
    transparent: true,
    opacity: 0.0,          // raised by applyDay; dust is only visible in a lit room
    depthWrite: false,
    blending: THREE.AdditiveBlending
  });

  const motes = new THREE.Points(moteGeometry, moteMaterial);
  motes.frustumCulled = false;
  motes.renderOrder = 2;
  scene.add(motes);

  const key = new THREE.DirectionalLight(0xFFF4E2, 2.05);
  key.position.set(-2.6, 3.6, 3.2);
  key.castShadow = true;
  key.shadow.mapSize.set(1024, 1024);
  key.shadow.camera.near = 1; key.shadow.camera.far = 14;
  key.shadow.camera.left = -3.4; key.shadow.camera.right = 3.4;
  key.shadow.camera.top = 3.4; key.shadow.camera.bottom = -3.4;
  key.shadow.radius = 3.2; key.shadow.bias = -0.0016;
  scene.add(key);

  const rim = new THREE.DirectionalLight(0xBFD0FF, 0.85);
  rim.position.set(3.4, 1.0, -2.6);
  scene.add(rim);

  const hemi = new THREE.HemisphereLight(0xF3EEE2, 0x8B8676, 0.72);
  scene.add(hemi);

  const floor = new THREE.Mesh(new THREE.PlaneGeometry(16, 16), new THREE.ShadowMaterial({ opacity: 0.30 }));
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -2.11;
  floor.receiveShadow = true;
  scene.add(floor);

  const state = { level: 0, music: 0, beat: 0, hour: options.hour ?? null, forced: options.hour != null };
  const scratch = new THREE.Color();

  /* CSS colours are mixed here in plain sRGB bytes, deliberately not through
     THREE.Color. A three colour is a *rendering* colour and lives in linear space; its
     channels read straight out come back as the wrong colour entirely — the first
     version of this turned a near-white tint into dark red. The lights above want a
     three colour; a stylesheet wants this. */
  const mixHex = (from, to, t) => [16, 8, 0].map(shift =>
    Math.round(((from >> shift) & 255) + (((to >> shift) & 255) - ((from >> shift) & 255)) * t));

  function blend(target, a, b, t) {
    scratch.setHex(b);
    target.setHex(a).lerp(scratch, t);
  }

  function applyDay(hour) {
    const { a, b, t } = frames(hour);
    blend(uniforms.uTop.value, a.top, b.top, t);
    blend(uniforms.uBottom.value, a.bottom, b.bottom, t);
    blend(uniforms.uGlow.value, a.glow, b.glow, t);
    blend(key.color, a.key, b.key, t);
    blend(rim.color, a.rim, b.rim, t);
    blend(hemi.color, a.sky, b.sky, t);
    blend(hemi.groundColor, a.ground, b.ground, t);
    key.intensity = a.keyLevel + (b.keyLevel - a.keyLevel) * t;
    rim.intensity = a.rimLevel + (b.rimLevel - a.rimLevel) * t;
    hemi.intensity = a.ambient + (b.ambient - a.ambient) * t;
    floor.material.opacity = 0.12 + (key.intensity / 2.2) * 0.20;

    // Dust is only visible where there is light to catch, so it follows the key rather
    // than sitting at a fixed strength. At night it all but disappears, which is right:
    // a dark room with motes in it reads as fog.
    moteMaterial.opacity = 0.05 + (key.intensity / 2.2) * 0.16;

    // The window's share of the same hour. Handed out rather than applied here: this
    // module owns a room, and the page is not part of it.
    if (options.onPalette) {
      const chrome = mixHex(a.chrome, b.chrome, t);
      const ink = mixHex(a.chromeInk, b.chromeInk, t);
      const alpha = a.chromeAlpha + (b.chromeAlpha - a.chromeAlpha) * t;

      options.onPalette({
        tint: `rgba(${chrome[0]},${chrome[1]},${chrome[2]},${alpha.toFixed(3)})`,
        ink: `rgb(${ink[0]},${ink[1]},${ink[2]})`,
        // A rule that reads on whatever the tint turned out to be.
        line: `rgba(${ink[0]},${ink[1]},${ink[2]},0.18)`,

        /* How much of this light a character should actually take.
           A VRM is authored for roughly unit lighting; this room runs its key up to
           2.2 at midday, which clips an anime face to a white oval and takes its eyes,
           nose and mouth with it. Normalising to 1 at the bright end keeps her legible
           without touching the room — and because it only ever *reduces*, the dim hours
           still read as dim. The bust ignores it; it was modelled for this rig. */
        avatarScale: Math.min(1, 1.02 / Math.max(key.intensity, 0.001))
      });
    }
  }

  applyDay(state.hour ?? new Date().getHours() + new Date().getMinutes() / 60);

  let sinceDayCheck = 0;

  return {
    /** Microphone amplitude. The halo answers it, so the room reacts before she does. */
    setLevel(v) { state.level = v; },

    /** Energy 0–1 and the decaying beat impulse. The room joins in, quietly. */
    setMusic(energy, beat) { state.music = energy; state.beat = beat; },

    /** Drawn first, behind everything, filling the frame whatever shape it is. */
    render(renderer) {
      renderer.render(backdropScene, backdropCamera);
    },

    setAspect(aspect) { uniforms.uAspect.value = aspect; },

    /** Pin the clock, for looking at midnight in the middle of the morning. */
    setHour(hour) {
      state.forced = hour != null;
      state.hour = hour;
      applyDay(hour ?? new Date().getHours() + new Date().getMinutes() / 60);
    },

    update(dt, elapsed) {
      uniforms.uTime.value = elapsed;
      uniforms.uLevel.value += (state.level - uniforms.uLevel.value) * (1 - Math.pow(0.05, dt));
      uniforms.uDrift.value = Math.sin(elapsed * 0.037);

      // Energy eases; the beat does not. Smoothing the beat would turn every pulse into
      // the same gentle swell and there would be nothing to feel.
      uniforms.uMusic.value += (state.music - uniforms.uMusic.value) * (1 - Math.pow(0.12, dt));
      uniforms.uBeat.value = state.beat;

      slabs.forEach(s => {
        s.mesh.position.x = s.home + Math.sin(elapsed * s.rate) * s.reach;
        // The nearer slab leans on the beat, so the depth belongs to the music too
        // rather than only the wall behind it changing colour.
        s.mesh.position.z = s.z + uniforms.uBeat.value * 0.22;
      });

      /* The motes drift, and lift very slightly, and wrap when they leave the top. The
         positions buffer is rewritten in place — allocating 780 floats a frame on a
         weak GPU is exactly the kind of thing that shows up as stutter later. */
      const drift = moteGeometry.attributes.position;
      for (let i = 0; i < MOTES; i++) {
        const rate = motePhase[i * 3 + 2];
        drift.array[i * 3] += Math.sin(elapsed * rate + motePhase[i * 3]) * dt * 0.06;
        drift.array[i * 3 + 1] += dt * rate * 0.045;
        if (drift.array[i * 3 + 1] > 3.2) drift.array[i * 3 + 1] = -3.2;
      }
      drift.needsUpdate = true;

      // Re-reading the clock every frame would be waste; every ten seconds is plenty
      // for something that takes an hour to change.
      sinceDayCheck += dt;
      if (!state.forced && sinceDayCheck > 10) {
        sinceDayCheck = 0;
        const now = new Date();
        applyDay(now.getHours() + now.getMinutes() / 60);
      }
    }
  };
}
```
