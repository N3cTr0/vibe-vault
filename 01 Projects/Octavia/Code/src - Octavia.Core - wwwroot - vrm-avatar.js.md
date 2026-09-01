---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\vrm-avatar.js
---

# src\Octavia.Core\wwwroot\vrm-avatar.js

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

  /* Dancing needs a body, and a body needs a rest position to move *from*.
     The idle above is the only pose the format does not supply, so it is also the only
     thing a dance can be an offset of — applying rotations absolutely would snap her
     arms back to a T-pose the moment the music started. */
  const DANCE_BONES = ['hips', 'spine', 'chest', 'upperChest',
                       'leftUpperArm', 'rightUpperArm', 'leftLowerArm', 'rightLowerArm'];
  const restPose = new Map();
  for (const name of DANCE_BONES) {
    const node = vrm.humanoid?.getNormalizedBoneNode(name);
    if (node) restPose.set(name, { node, x: node.rotation.x, y: node.rotation.y, z: node.rotation.z });
  }

  const hips = restPose.get('hips')?.node ?? null;
  const hipsRestY = hips ? hips.position.y : 0;

  // Look at a point she can be told to move, rather than posing the eyes by hand.
  const gazeTarget = new THREE.Object3D();
  root.add(gazeTarget);
  if (vrm.lookAt) vrm.lookAt.target = gazeTarget;

  // Frame her face, rather than scaling a person to fit a bust's camera. Whatever
  // height the character is, the head ends up the same size on screen.
  vrm.scene.updateWorldMatrix(true, true);
  const headPoint = new THREE.Vector3(0, 1.4, 0);
  if (head) head.getWorldPosition(headPoint);

  /* Headphones ride on the head bone, so they follow every nod and turn for free.

     Sizing them from the character's *height* — which is what `headPoint.y * 0.115` did
     — is a guess about head width made from an unrelated measurement, and it is wrong by
     a different amount for every model. It is measured now: the head bone's world height
     against the top of the model's own bounding box gives the part of the character that
     is actually head, and everything else is a proportion of that.

     They also have to be *positioned*. A head bone sits at the base of the skull, so a
     band added at the bone's origin hangs around the neck; it needs lifting by most of
     the head's height and easing back, because ears are behind the centre of a face. */
  const bounds = new THREE.Box3().setFromObject(vrm.scene);
  const headTop = bounds.max.y;
  const headHeight = head && headTop > headPoint.y ? headTop - headPoint.y : 0.22;

  /* Where the ears are, from the eyes rather than from a proportion.
     A rig has no ear bone, but it does have eyes, and on every human head — and every
     stylised one drawn from a human — **the ear canal sits at eye height and behind the
     eyes**. VRM 1.0 requires `leftEye` and `rightEye`, so that is two measurements
     instead of two guesses: the eyes give the height directly, and the gap between them
     gives the head's width to hang the cups on. Sizing from head *height* put them too
     high and too far forward, because a head is not as deep as it is tall. */
  const eyeL = vrm.humanoid?.getNormalizedBoneNode('leftEye') ?? null;
  const eyeR = vrm.humanoid?.getNormalizedBoneNode('rightEye') ?? null;

  // Wide enough to sit outside the hair rather than inside it, which is where a real
  // pair would be. Narrower and the cups vanish into a long fringe.
  const halfWidth = headHeight * 0.68;

  // Fallbacks for a rig with no eye bones. The height is the one that was wrong: cups
  // sat around the cheekbone, a good way below where an ear is.
  let earHeight = headHeight * 0.52;
  let earBack = -halfWidth * 0.14;

  if (eyeL && eyeR && head) {
    const left = eyeL.getWorldPosition(new THREE.Vector3());
    const right = eyeR.getWorldPosition(new THREE.Vector3());
    const headWorld = head.getWorldPosition(new THREE.Vector3());
    const eyeMid = left.clone().add(right).multiplyScalar(0.5);

    // Head-local, because that is the space the headphones are parented into. The eye
    // line *is* the ear line — that is the whole measurement.
    earHeight = eyeMid.y - headWorld.y;

    /* And behind it, because an ear canal sits well back from the eyes.

       The sign here is measured, not reasoned. The loader's comment above says VRM 1.0
       faces -Z, which invites the assumption that "behind" is +Z — but after the plugin
       has finished with it the head bone's axes are world-aligned and the eye bones sit
       at *greater* z than the head bone, so the face points **+Z** and behind is
       negative. Proved by shoving the whole assembly to +0.10 and watching it end up in
       front of her nose. The 0.26 is then just where the cup lands on the ear rather
       than the temple or the back of the skull. */
    earBack = -(eyeMid.z - headWorld.z) - halfWidth * 0.26;
  }

  const headphones = createHeadphones(halfWidth);
  headphones.object.position.set(0, earHeight, earBack);

  if (head) head.add(headphones.object);
  else {
    headphones.object.position.y += headPoint.y;
    root.add(headphones.object);
  }

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
    blink: 0, gazeX: 0, gazeY: 0, breathe: 1,
    dance: 0, danceSway: 0, danceBeat: 0
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

    /** How much of her is moving to the music.
        `amount` 0–1 eases the whole thing in and out, `sway` is the bar-length
        oscillation the face already computes, and `beat` is the per-beat impulse. */
    setDance(amount, sway, beat) {
      state.dance = amount;
      state.danceSway = sway;
      state.danceBeat = beat;
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

      /* The dance.

         Head movement alone reads as listening, not dancing — a person moving to music
         moves from the hips, and the shoulders and arms follow a beat late. Every bone
         here is written as rest + offset, and the offsets are small: an anime rig will
         happily bend into shapes no shoulder makes, and the line between dancing and
         convulsing is about fifteen degrees.

         Counter-rotation is what sells it. The chest turns *against* the hips rather
         than with them, which is what a torso actually does and what stops the whole
         body reading as one board being waved. */
      const dance = state.dance ?? 0;
      if (restPose.size > 0) {
        const s = (state.danceSway ?? 0) * dance;
        const b = (state.danceBeat ?? 0) * dance;
        const put = (name, x, y, z) => {
          const rest = restPose.get(name);
          if (rest) rest.node.rotation.set(rest.x + x, rest.y + y, rest.z + z);
        };

        put('hips', b * 0.05, s * 0.13, s * 0.075);
        put('spine', -b * 0.035, s * -0.05, s * -0.055);
        put('chest', -b * 0.02, s * -0.07, s * -0.045);
        put('upperChest', 0, s * -0.04, s * -0.03);

        // Arms swing on the opposite phase to each other, and lag the bar slightly so
        // they trail the body rather than moving with it.
        const swing = Math.sin(elapsed * 2.0) * dance;
        put('leftUpperArm', swing * 0.10, 0, s * 0.16 + b * 0.07);
        put('rightUpperArm', -swing * 0.10, 0, s * 0.16 - b * 0.07);
        put('leftLowerArm', 0, 0, -Math.abs(s) * 0.14 - b * 0.10);
        put('rightLowerArm', 0, 0, Math.abs(s) * 0.14 + b * 0.10);

        // A knee-bend she does not have bones for, faked by dropping the hips on the
        // beat. Small: a centimetre reads as a bounce, five reads as a fault.
        if (hips) hips.position.y = hipsRestY - b * 0.012 * dance;
      }

      // Applies expressions, look-at and the spring bones that make hair move.
      vrm.update(dt);
    }
  };
}
```
