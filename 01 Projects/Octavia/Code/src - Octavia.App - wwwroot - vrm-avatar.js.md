---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\vrm-avatar.js
---

# src\Octavia.App\wwwroot\vrm-avatar.js

```javascript
/* A VRM character, as an avatar.

   Same surface as the bust — setViseme, setExpression, setGaze, setBlink, setPose,
   update — so the face drives either one without knowing which it has. The whole point
   of Stage 3's protocol was that photorealism becomes a swap; this is the same bet one
   level down, and it is why the expression vocabulary *is* VRM 1.0's: the mapping from
   protocol to character is an identity, with nothing in between to get wrong. */
import * as THREE from './lib/three.module.js';
import { GLTFLoader } from './lib/GLTFLoader.js';
import { VRMLoaderPlugin, VRMUtils } from './lib/three-vrm.module.js';
import { createHeadphones } from './headphones.js';

const VISEMES = ['aa', 'ih', 'ou', 'ee', 'oh'];
const EMOTIONS = ['happy', 'angry', 'sad', 'relaxed', 'surprised'];

export async function loadVrmAvatar(url) {
  const loader = new GLTFLoader();
  loader.register(parser => new VRMLoaderPlugin(parser));

  const gltf = await loader.loadAsync(url);
  const vrm = gltf.userData.vrm;
  if (!vrm) throw new Error('that file loaded, but it is not a VRM');

  VRMUtils.removeUnnecessaryVertices(gltf.scene);
  VRMUtils.combineSkeletons(gltf.scene);

  // VRM 0.x characters face +Z; 1.0 faces -Z. Rotating the old ones is the one
  // difference between the two versions that reaches this far.
  if (vrm.meta?.metaVersion === '0') VRMUtils.rotateVRM0(vrm);

  // Heads and hands leave the bounding box the moment she moves; culling them mid-turn
  // is the classic VRM flicker.
  vrm.scene.traverse(node => { node.frustumCulled = false; });

  const root = new THREE.Group();
  root.add(vrm.scene);

  const head = vrm.humanoid?.getNormalizedBoneNode('head') ?? null;

  /* Every VRM is authored in a T-pose, arms straight out, because that is the rig's
     rest position rather than anybody's idea of standing still. Nothing in the format
     supplies an idle, so she gets one here: arms down, elbows just off straight. */
  function rest(bone, x, y, z) {
    const node = vrm.humanoid?.getNormalizedBoneNode(bone);
    if (node) node.rotation.set(x, y, z);
  }
  rest('leftUpperArm', 0.06, 0, -1.24);
  rest('rightUpperArm', 0.06, 0, 1.24);
  rest('leftLowerArm', 0, -0.16, -0.12);
  rest('rightLowerArm', 0, 0.16, 0.12);
  rest('leftHand', 0, 0, -0.08);
  rest('rightHand', 0, 0, 0.08);
  rest('leftShoulder', 0, 0, -0.06);
  rest('rightShoulder', 0, 0, 0.06);

  // Look at a point she can be told to move, rather than posing the eyes by hand.
  const gazeTarget = new THREE.Object3D();
  root.add(gazeTarget);
  if (vrm.lookAt) vrm.lookAt.target = gazeTarget;

  // Frame her face, rather than scaling a person to fit a bust's camera. Whatever
  // height the character is, the head ends up the same size on screen.
  vrm.scene.updateWorldMatrix(true, true);
  const headPoint = new THREE.Vector3(0, 1.4, 0);
  if (head) head.getWorldPosition(headPoint);

  /* Headphones ride on the head bone, so they follow every nod and turn for free. The
     size is taken from the character rather than assumed: VRM does not standardise how
     big a head is, and a fixed radius fits one model and swallows the next. */
  const headSize = headPoint.y > 0 ? headPoint.y * 0.115 : 0.16;
  const headphones = createHeadphones(headSize);
  if (head) head.add(headphones.object);
  else root.add(headphones.object);

  /* Making her face legible.

     A VRM is MToon — a toon shader carrying line art for eyes, brows and mouth as
     textures on the face. Two things were erasing all of it in this room, both measured
     rather than guessed:

       - `rimLightingMixFactor` is 1 on every material in the sample models, which lays
         a bright rim wash over the whole head.
       - The room's key runs to 2.2 at midday against the roughly unit lighting these
         models assume, so the lit side clips and takes the dark line art with it. At
         42% she had eyes, a nose and a mouth again; at 100% she was a white oval.

     The rim is switched off outright. The over-lighting is handled by scaling what she
     responds to rather than dimming the room — see `setLightScale` below. The outline
     is widened because it is the one thing in MToon that draws contour regardless of
     how bright the light gets. */
  const toon = [];
  vrm.scene.traverse(node => {
    const materials = Array.isArray(node.material) ? node.material : node.material ? [node.material] : [];
    for (const material of materials) {
      if (!('rimLightingMixFactor' in material)) continue;

      material.rimLightingMixFactor = 0;

      if (material.outlineWidthMode && material.outlineWidthMode !== 'none') {
        material.outlineWidthFactor = Math.max(material.outlineWidthFactor ?? 0, 0.0022);
      }

      // The base factors are kept so the scale below is applied to them rather than
      // compounding on whatever the last frame left behind.
      toon.push({
        material,
        lit: material.color?.clone(),
        shade: material.shadeColorFactor?.clone()
      });
    }
  });

  const expressions = vrm.expressionManager;
  const has = name => !!expressions?.getExpression?.(name);

  const state = {
    shape: null, weight: 0, smoothed: 0,
    expression: 'neutral', expressionWeight: 0, applied: {},
    blink: 0, gazeX: 0, gazeY: 0, breathe: 1
  };

  function set(name, value) {
    if (!expressions || !has(name)) return;
    expressions.setValue(name, value);
  }

  return {
    root,
    vrm,
    /** Where the camera should sit to make a portrait of this particular character. */
    frame: { target: headPoint.clone(), distance: 1.30, height: -0.10 },

    setActivity() { /* the bust's plinth arc has no counterpart here */ },
    setBreathing(on) { state.breathe = on ? 1 : 0; },
    setViseme(shape, weight) { state.shape = shape; state.weight = weight; },
    setExpression(name, weight) { state.expression = name; state.expressionWeight = weight; },
    setGaze(x, y) { state.gazeX = x; state.gazeY = y; },
    setBlink(v) { state.blink = v; },
    setLevel() { },
    setHeadphones(v) { headphones.set(v); },

    /** Scales her response to the room's light so the lit side never clips. Multiplying
        the material's colour factors is the same arithmetic as dimming the lights, but
        it lands on this character only — the room and the bust keep the light they were
        built for, and she still darkens through the evening because the scale only ever
        reduces. */
    setLightScale(scale) {
      const k = Math.min(1, Math.max(0.05, scale ?? 1));
      for (const entry of toon) {
        if (entry.lit) entry.material.color.copy(entry.lit).multiplyScalar(k);
        if (entry.shade) entry.material.shadeColorFactor.copy(entry.shade).multiplyScalar(k);
      }
    },
    setPose(yaw, pitch, roll) {
      if (head) head.rotation.set(pitch, yaw, roll);
    },

    update(dt, elapsed) {
      state.smoothed += (state.weight - state.smoothed) * (1 - Math.pow(0.004, dt));

      for (const name of VISEMES) {
        set(name, name === state.shape ? state.smoothed : 0);
      }

      for (const name of EMOTIONS) {
        const target = name === state.expression ? state.expressionWeight : 0;
        const current = state.applied[name] ?? 0;
        state.applied[name] = current + (target - current) * (1 - Math.pow(0.02, dt));
        set(name, state.applied[name]);
      }

      set('blink', state.blink);

      // The gaze target lives a metre in front of her face; moving it is how the eyes
      // and the head-follow both learn where she is looking.
      gazeTarget.position.set(
        headPoint.x + state.gazeX * 0.9,
        headPoint.y + state.gazeY * 0.7,
        headPoint.z + 1.0);

      root.position.y = Math.sin(elapsed * 0.62) * 0.006 * state.breathe;

      // Applies expressions, look-at and the spring bones that make hair move.
      vrm.update(dt);
    }
  };
}
```
