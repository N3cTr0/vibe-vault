---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\bust.js
---

# src\Octavia.App\wwwroot\bust.js

```javascript
/* The plaster bust, as an avatar.

   Every avatar exposes the same surface — setViseme, setExpression, setGaze, setBlink,
   setPose, update — so the face can drive a stylised bust or a photoreal character
   without knowing which it has. That interface is the same idea as the host's
   "the face is a renderer": one level down, the avatar is a puppet. */
import * as THREE from './lib/three.module.js';
import { createHeadphones } from './headphones.js';

const PLASTER = 0xF2EDE2, STONE = 0xB9B2A2, DARK = 0x2A2620;

/* Mouth shapes per VRM viseme, so the shape channel means something even here:
   width and height of the mouth, and how far the jaw drops. */
const SHAPES = {
  aa: { w: 1.00, h: 1.00, jaw: 1.00 },
  oh: { w: 0.62, h: 1.05, jaw: 0.85 },
  ou: { w: 0.48, h: 0.80, jaw: 0.55 },
  ee: { w: 1.22, h: 0.42, jaw: 0.40 },
  ih: { w: 1.05, h: 0.36, jaw: 0.30 }
};

/* Poses per expression. The bust has no blendshapes worth the name, so an expression
   is a small posture: brows, lids, mouth and carriage. The vocabulary is VRM 1.0's,
   which is what makes the mapping an identity when a real avatar lands. */
const POSES = {
  neutral:   { brow: 0.000, browTilt:  0.00, squint: 0.00, mouthW: 1.00, mouthUp: 0.000, tilt:  0.000 },
  happy:     { brow: 0.030, browTilt:  0.10, squint: 0.22, mouthW: 1.30, mouthUp: 0.045, tilt:  0.030 },
  relaxed:   { brow: 0.010, browTilt:  0.04, squint: 0.12, mouthW: 1.12, mouthUp: 0.020, tilt:  0.020 },
  sad:       { brow: 0.020, browTilt: -0.26, squint: 0.10, mouthW: 0.88, mouthUp: -0.030, tilt: -0.055 },
  angry:     { brow: -0.055, browTilt: 0.30, squint: 0.30, mouthW: 0.86, mouthUp: -0.020, tilt: 0.000 },
  surprised: { brow: 0.085, browTilt: -0.04, squint: -0.30, mouthW: 0.92, mouthUp: 0.010, tilt: -0.020 }
};

function headGeometry() {
  const g = new THREE.SphereGeometry(1, 96, 72);
  g.scale(0.90, 1.05, 0.93);
  const src = g.attributes.position.array;
  const jaw = new Float32Array(src.length);
  const pivotY = -0.10, pivotZ = -0.45, maxDrop = 0.44;
  const v = new THREE.Vector3();
  for (let i = 0; i < src.length; i += 3) {
    v.set(src[i], src[i + 1], src[i + 2]);
    const low = THREE.MathUtils.smoothstep(-v.y, 0.16, 0.80);
    const front = THREE.MathUtils.smoothstep(v.z, -0.25, 0.55);
    const w = low * front;
    if (w > 0.0005) {
      const py = v.y - pivotY, pz = v.z - pivotZ;
      const a = maxDrop * w, c = Math.cos(a), s = Math.sin(a);
      jaw[i] = v.x * (1 - 0.07 * w);
      jaw[i + 1] = py * c - pz * s + pivotY;
      jaw[i + 2] = py * s + pz * c + pivotZ;
    } else {
      jaw[i] = v.x; jaw[i + 1] = v.y; jaw[i + 2] = v.z;
    }
  }
  g.morphAttributes.position = [new THREE.Float32BufferAttribute(jaw, 3)];
  return g;
}

export function createBust() {
  // morphTargets is no longer a material flag — three enables it from the geometry.
  const plasterMat = new THREE.MeshStandardMaterial({ color: PLASTER, roughness: 0.88, metalness: 0 });
  const plasterPlain = new THREE.MeshStandardMaterial({ color: PLASTER, roughness: 0.88, metalness: 0 });
  const darkMat = new THREE.MeshStandardMaterial({ color: DARK, roughness: 0.55, metalness: 0 });
  const stoneMat = new THREE.MeshStandardMaterial({ color: STONE, roughness: 0.95, metalness: 0 });

  const root = new THREE.Group();
  const headPivot = new THREE.Group();
  root.add(headPivot);

  const head = new THREE.Mesh(headGeometry(), plasterMat);
  head.castShadow = true; head.receiveShadow = true;
  head.morphTargetInfluences = [0];
  headPivot.add(head);

  function brow(sign) {
    const m = new THREE.Mesh(new THREE.BoxGeometry(0.30, 0.045, 0.09), plasterPlain);
    m.position.set(sign * 0.30, 0.30, 0.78);
    m.rotation.z = sign * -0.10;
    m.castShadow = true;
    const holder = new THREE.Group(); holder.add(m); headPivot.add(holder);
    return { holder, mesh: m, sign };
  }
  const brows = [brow(-1), brow(1)];

  function eye(sign) {
    const g = new THREE.Group();
    /* Forward of where these used to sit. The head's surface at eye height is z ≈ 0.871
       and the iris was reaching only 0.839 — buried, exactly like the old mouth was, so
       she had no visible eyes at all on a pale face under a bright key. It now clears
       by a hair, which is all an iris needs to read. */
    g.position.set(sign * 0.30, 0.115, 0.755);
    const ball = new THREE.Mesh(new THREE.SphereGeometry(0.132, 32, 24),
      new THREE.MeshStandardMaterial({ color: 0xFBF8F1, roughness: 0.42 }));
    const aim = new THREE.Group();
    const iris = new THREE.Mesh(new THREE.SphereGeometry(0.060, 24, 16),
      new THREE.MeshStandardMaterial({ color: 0x1E2740, roughness: 0.28 }));
    iris.position.z = 0.100; iris.scale.set(1, 1, 0.5);
    aim.add(iris);
    const upper = new THREE.Mesh(new THREE.SphereGeometry(0.142, 28, 18, 0, Math.PI * 2, 0, Math.PI / 2), plasterPlain);
    const lower = new THREE.Mesh(new THREE.SphereGeometry(0.142, 28, 18, 0, Math.PI * 2, Math.PI / 2, Math.PI / 2), plasterPlain);
    upper.castShadow = true;
    upper.rotation.x = -0.42;
    lower.rotation.x = 0.30;
    g.add(ball, aim, upper, lower);
    headPivot.add(g);
    return { group: g, aim, upper, lower };
  }
  const eyes = [eye(-1), eye(1)];

  const nose = new THREE.Mesh(new THREE.SphereGeometry(1, 24, 18), plasterPlain);
  nose.scale.set(0.085, 0.135, 0.115);
  nose.position.set(0, -0.055, 0.865);
  nose.castShadow = true;
  headPivot.add(nose);

  /* The mouth, as three pieces rather than one.

     It used to be a single dark ellipsoid that closed to 1.6% of its height and sat
     0.076 *inside* the face — so at rest she had no mouth at all, and one appeared out
     of the plaster only while she spoke. A bust has a mouth when it is silent.

     The head is a sphere scaled (0.90, 1.05, 0.93), so its surface at mouth height is
     z = 0.93 * sqrt(1 - (0.30/1.05)^2), about 0.891. Everything below is placed against
     that number: the lips stand slightly proud of it and the aperture sits just behind,
     which is what makes a seam read as a seam and not a drawn line. */
  /* The numbers here are all answers to one problem: the head is a sphere, so it curves
     away from the mouth. Across a lip's half-width the surface falls back about 0.04,
     and a shallow form laid on it is buried at the corners and pokes out only in the
     middle — which is a beak, not a mouth.

     So the lips are *deep* ellipsoids whose centres sit well inside the head (z 0.80)
     with a long z-radius. Only their front caps emerge, standing about 0.024 proud at
     the centre and tapering to nothing at the corners exactly as the head curves away.
     A mouth that emerges from the face instead of sitting on it. */
  const MOUTH_Y = -0.30, MOUTH_Z = 0.80;
  const LIP_GAP = 0.055;

  const mouth = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 20), darkMat);
  mouth.position.set(0, MOUTH_Y, MOUTH_Z);
  headPivot.add(mouth);

  function lip(sign) {
    const m = new THREE.Mesh(new THREE.SphereGeometry(1, 40, 22), plasterPlain);
    m.scale.set(0.26, sign > 0 ? 0.045 : 0.052, 0.115);
    m.position.set(0, MOUTH_Y + sign * LIP_GAP, MOUTH_Z);
    m.castShadow = true;
    headPivot.add(m);
    return m;
  }
  const lipUpper = lip(1), lipLower = lip(-1);

  const neck = new THREE.Mesh(new THREE.CylinderGeometry(0.32, 0.44, 0.62, 40), plasterPlain);
  neck.position.set(0, -1.06, -0.04);
  neck.castShadow = true; neck.receiveShadow = true;
  root.add(neck);

  const chest = new THREE.Mesh(new THREE.CylinderGeometry(0.52, 0.70, 0.52, 44), plasterPlain);
  chest.position.set(0, -1.52, -0.04);
  chest.castShadow = true; chest.receiveShadow = true;
  root.add(chest);

  const plinth = new THREE.Mesh(new THREE.CylinderGeometry(0.74, 0.80, 0.34, 48), stoneMat);
  plinth.position.set(0, -1.94, -0.04);
  plinth.castShadow = true; plinth.receiveShadow = true;
  root.add(plinth);

  // The head is a unit sphere before scaling, so it is its own radius.
  const headphones = createHeadphones(1.0);
  headPivot.add(headphones.object);

  const arcMat = new THREE.MeshBasicMaterial({ color: 0x2F4CD0, transparent: true, opacity: 0 });
  const arc = new THREE.Mesh(new THREE.TorusGeometry(0.86, 0.012, 8, 90, 0.62), arcMat);
  arc.userData.span = 0.62;
  arc.rotation.x = Math.PI / 2;
  arc.position.set(0, -2.10, -0.04);
  root.add(arc);

  const state = {
    shape: null, weight: 0, open: 0,
    expression: 'neutral', expressionWeight: 0,
    pose: { ...POSES.neutral },
    blink: 0, gazeX: 0, gazeY: 0,
    level: 0, busy: false, spin: 0, spinRate: 0.25, breathe: 1
  };

  const lerp = (a, b, t) => a + (b - a) * t;

  return {
    root,
    /** The plinth arc is the bust's own flourish; the face tells it how lively to be. */
    setActivity(busy, spinRate, span) {
      state.busy = busy;
      state.spinRate = spinRate;
      if (Math.abs(span - arc.userData.span) > 0.07) {
        arc.userData.span = span;
        arc.geometry.dispose();
        arc.geometry = new THREE.TorusGeometry(0.86, 0.012, 8, 90, span);
      }
    },
    setBreathing(on) { state.breathe = on ? 1 : 0; },
    setViseme(shape, weight) { state.shape = shape; state.weight = weight; },
    setExpression(name, weight) {
      state.expression = POSES[name] ? name : 'neutral';
      state.expressionWeight = weight;
    },
    setGaze(x, y) { state.gazeX = x; state.gazeY = y; },
    setBlink(v) { state.blink = v; },
    setLevel(v) { state.level = v; },
    setHeadphones(v) { headphones.set(v); },
    // Modelled for this room's rig, so it takes the light as it comes.
    setLightScale() { },
    setPose(yaw, pitch, roll) { headPivot.rotation.set(pitch, yaw, roll + state.pose.tilt * state.expressionWeight); },

    update(dt, elapsed) {
      // Ease towards the expression's posture so a change of mood is a movement,
      // never a cut.
      const target = POSES[state.expression];
      const k = 1 - Math.pow(0.02, dt);
      for (const key of Object.keys(target)) {
        state.pose[key] = lerp(state.pose[key], target[key] * state.expressionWeight, k);
      }

      const shape = SHAPES[state.shape] || SHAPES.aa;
      const w = state.shape ? state.weight : 0;
      state.open = lerp(state.open, w, 1 - Math.pow(0.004, dt));

      head.morphTargetInfluences[0] = state.open * shape.jaw;

      const width = shape.w * (state.pose.mouthW || 1);
      const drop = state.open * 0.075 * shape.jaw;
      const centre = MOUTH_Y + state.pose.mouthUp;

      /* The aperture sits behind the lips and shows through the gap between them —
         0.09 of depth against the lips' 0.115, so it reads as a shadowed opening
         rather than a dark shape stuck to her face. */
      mouth.scale.set(
        (0.21 + state.open * 0.03) * width,
        0.026 + state.open * 0.15 * shape.h,
        0.09);
      mouth.position.y = centre - drop;
      mouth.position.z = MOUTH_Z - state.open * 0.03;

      /* The lips part around it: the upper barely moves, the lower rides the jaw. That
         asymmetry is most of what makes a mouth look like it is opening rather than a
         hole growing. */
      lipUpper.scale.x = 0.26 * width;
      lipUpper.position.y = centre + LIP_GAP + state.open * 0.022 * shape.h;

      lipLower.scale.x = 0.26 * width;
      lipLower.position.y = centre - LIP_GAP - drop - state.open * 0.09 * shape.h;

      const lid = THREE.MathUtils.clamp(state.blink + state.pose.squint, 0, 1);
      eyes.forEach(e => {
        e.upper.rotation.x = lerp(-0.42, 0.62, lid);
        e.lower.rotation.x = lerp(0.30, -0.16, lid);
        e.aim.rotation.y = -state.gazeX;
        e.aim.rotation.x = state.gazeY;
      });

      const raise = state.pose.brow + state.open * 0.018;
      brows.forEach(b => {
        b.mesh.position.y = 0.30 + raise;
        // Inner ends up is grief; outer ends up is fury. One number, both faces.
        b.holder.rotation.z = raise * b.sign * 0.9 + state.pose.browTilt * b.sign * 0.5;
      });

      root.position.y = Math.sin(elapsed * 0.62) * 0.011 * state.breathe;

      arcMat.opacity = lerp(arcMat.opacity, state.busy ? 0.85 : 0, 1 - Math.pow(0.02, dt));
      state.spin += dt * state.spinRate;
      arc.rotation.z = state.spin;
    }
  };
}
```
