---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\headphones.js
---

# src\Octavia.App\wwwroot\headphones.js

```javascript
/* The headphones.

   A prop rather than a feature, and the reason a stylised character was worth building
   before a photoreal one: putting an object on a head is ordinary geometry here, where a
   neural face would have nothing to hang it from.

   Built once and shared by both avatars, sized to whatever head is asking, because the
   bust is a unit sphere and a VRM head is about fifteen centimetres. */
import * as THREE from './lib/three.module.js';

const SHELL = 0x22242C, PAD = 0x14151A, TRIM = 0xB9BFD4;

export function createHeadphones(radius) {
  const group = new THREE.Group();

  const shellMat = new THREE.MeshStandardMaterial({ color: SHELL, roughness: 0.42, metalness: 0.35 });
  const padMat = new THREE.MeshStandardMaterial({ color: PAD, roughness: 0.95, metalness: 0 });
  const trimMat = new THREE.MeshStandardMaterial({ color: TRIM, roughness: 0.30, metalness: 0.65 });

  /* The band is a partial torus standing upright, wide enough to clear the head. Its
     arc stops short of the ears so the cups can meet it rather than pass through it. */
  const band = new THREE.Mesh(
    new THREE.TorusGeometry(radius * 1.02, radius * 0.055, 10, 48, Math.PI * 0.86),
    shellMat);
  band.rotation.z = Math.PI * 0.07;
  band.castShadow = true;
  group.add(band);

  const padTop = new THREE.Mesh(
    new THREE.TorusGeometry(radius * 0.96, radius * 0.045, 8, 32, Math.PI * 0.5),
    padMat);
  padTop.rotation.z = Math.PI * 0.25;
  group.add(padTop);

  function cup(sign) {
    const holder = new THREE.Group();
    holder.position.set(sign * radius * 1.00, radius * 0.02, -radius * 0.04);
    holder.rotation.z = sign * -0.06;

    const shell = new THREE.Mesh(new THREE.CylinderGeometry(radius * 0.33, radius * 0.30, radius * 0.16, 32), shellMat);
    shell.rotation.z = Math.PI / 2;
    shell.castShadow = true;

    const cushion = new THREE.Mesh(new THREE.CylinderGeometry(radius * 0.30, radius * 0.26, radius * 0.12, 32), padMat);
    cushion.rotation.z = Math.PI / 2;
    cushion.position.x = sign * -radius * 0.09;

    const ring = new THREE.Mesh(new THREE.TorusGeometry(radius * 0.32, radius * 0.012, 8, 32), trimMat);
    ring.rotation.y = Math.PI / 2;
    ring.position.x = sign * radius * 0.085;

    holder.add(shell, cushion, ring);
    return holder;
  }

  group.add(cup(-1), cup(1));

  // Hidden until asked for, and never rendered at zero — an invisible object still
  // costs a draw call, and this one sits in front of her face.
  group.visible = false;

  return {
    object: group,

    /** 0 is off and away, 1 is on her head. Anything between is the movement. */
    set(amount) {
      const t = THREE.MathUtils.clamp(amount, 0, 1);
      group.visible = t > 0.001;
      if (!group.visible) return;

      // They arrive from above and settle, rather than fading in on the spot: a prop
      // that materialises reads as a bug, and one that is put on reads as a choice.
      group.position.y = radius * (1 - t) * 0.85;
      group.rotation.x = (1 - t) * -0.35;
      group.scale.setScalar(0.90 + t * 0.10);

      group.traverse(node => {
        if (!node.material) return;
        node.material.transparent = t < 0.999;
        node.material.opacity = t;
      });
    }
  };
}
```
