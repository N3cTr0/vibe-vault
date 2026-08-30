---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\dev.js
---

# src\Octavia.App\wwwroot\dev.js

```javascript
/* The dev panel: every performance the face can give, on a button.

   The face normally does what the host tells it, which makes anything rare — a mood, a
   viseme, the headphones — awkward to look at: you have to *cause* it. This drives
   `window.Face` directly instead, so a shape can be held still and judged.

   It is built here rather than in index.html because it is scaffolding. The module is
   imported only when the panel is asked for, so a published face never loads it.

   Two kinds of control, and the difference matters:
     - most of them talk to `window.Face`, and never leave the renderer
     - the ones marked as senses send to the *host*, because only the host owns a device

   `Hold the face` stops host messages reaching the renderer, so a mood set here is not
   wiped by the next thing she says. Senses still work while it is on. */

const STATES = ['idle', 'listening', 'thinking', 'speaking'];
const EMOTIONS = ['neutral', 'happy', 'relaxed', 'surprised', 'sad', 'angry'];
const VISEMES = ['aa', 'ih', 'ou', 'ee', 'oh'];
const HOURS = [['Clock', -1], ['Dawn', 6], ['Morning', 9], ['Midday', 13], ['Evening', 18], ['Dusk', 21], ['Night', 0]];

export function createDevPanel({ send, look, onHoldChanged }) {
  const Face = window.Face;
  const body = document.getElementById('devBody');
  const state = { weight: 1, openness: 0.7, autoBeat: 0, bpm: 120, energy: 0.8 };

  const el = (tag, className, text) => {
    const node = document.createElement(tag);
    if (className) node.className = className;
    if (text !== undefined) node.textContent = text;
    return node;
  };

  function group(title) {
    const wrap = el('div', 'devgroup');
    wrap.appendChild(el('span', 'devlabel', title));
    const row = el('div', 'devrow');
    wrap.appendChild(row);
    body.appendChild(wrap);
    return row;
  }

  function button(row, label, onClick, className) {
    const b = el('button', className ? `devbtn ${className}` : 'devbtn', label);
    b.type = 'button';
    b.addEventListener('click', () => onClick(b));
    row.appendChild(b);
    return b;
  }

  /** A row of buttons where exactly one is lit. */
  function choice(row, options, onPick, current) {
    const made = options.map(([label, value]) =>
      button(row, label, b => {
        made.forEach(other => other.classList.remove('on'));
        b.classList.add('on');
        onPick(value);
      }));

    const at = options.findIndex(([, value]) => value === current);
    if (at >= 0) made[at].classList.add('on');
    return made;
  }

  function slider(row, label, min, max, step, value, onInput) {
    const wrap = el('label', 'devslider');
    wrap.appendChild(el('span', null, label));

    const input = document.createElement('input');
    input.type = 'range';
    input.min = min; input.max = max; input.step = step; input.value = value;

    const readout = el('span', 'devvalue', Number(value).toFixed(2));
    input.addEventListener('input', () => {
      readout.textContent = Number(input.value).toFixed(2);
      onInput(Number(input.value));
    });

    wrap.append(input, readout);
    row.appendChild(wrap);
    return input;
  }

  // ── hold ────────────────────────────────────────────────
  let held = false;
  const holdRow = group('The host');
  button(holdRow, 'Hold the face', b => {
    held = !held;
    b.classList.toggle('on', held);
    b.textContent = held ? 'Held — host ignored' : 'Hold the face';
    onHoldChanged(held);
  });

  // ── state ───────────────────────────────────────────────
  const current = Face.read();
  choice(group('State'), STATES.map(s => [s, s]), s => Face.setState(s), current.state);

  // ── emotion ─────────────────────────────────────────────
  const moodRow = group('Mood');
  choice(moodRow, EMOTIONS.map(e => [e, e]), e => Face.setEmotion(e, state.weight), current.expression);
  slider(moodRow, 'weight', 0, 1, 0.05, state.weight, v => state.weight = v);

  // ── mouth ───────────────────────────────────────────────
  const mouthRow = group('Mouth');
  choice(mouthRow, VISEMES.map(v => [v, v]).concat([['shut', null]]),
    shape => Face.setViseme(shape ? state.openness : 0, shape));
  slider(mouthRow, 'open', 0, 1, 0.05, state.openness, v => state.openness = v);

  // Speaking is a *sequence*, and a single held shape says nothing about whether the
  // mouth reads as talking. This is the cheapest way to see that without a voice.
  button(mouthRow, 'Say a line', b => {
    if (b.dataset.running) return;
    b.dataset.running = '1';
    b.classList.add('on');

    let i = 0;
    const timer = setInterval(() => {
      const shape = VISEMES[Math.floor(Math.random() * VISEMES.length)];
      Face.setViseme(0.25 + Math.random() * 0.7, shape);
      if (++i > 34) {
        clearInterval(timer);
        Face.setViseme(0, null);
        b.classList.remove('on');
        delete b.dataset.running;
      }
    }, 85);
  });

  // ── eyes ────────────────────────────────────────────────
  const eyeRow = group('Eyes');
  button(eyeRow, 'Blink', () => Face.blink());
  choice(eyeRow, [['left', [-0.30, 0]], ['up', [0, 0.22]], ['at you', [0, 0]],
                  ['down', [0, -0.20]], ['right', [0.30, 0]]],
    ([x, y]) => Face.look(x, y));
  button(eyeRow, 'Let her look', b => {
    Face.look(null);
    eyeRow.querySelectorAll('.devbtn.on').forEach(other => other.classList.remove('on'));
    b.classList.remove('on');
  });

  // ── level ───────────────────────────────────────────────
  // The host stops sending level when it stops metering and the face decays it, so this
  // has to be re-sent rather than set once.
  const levelRow = group('Microphone level');
  let levelHold = 0;
  const levelInput = slider(levelRow, 'level', 0, 1, 0.05, 0, v => {
    clearInterval(levelHold);
    if (v > 0) levelHold = setInterval(() => Face.setLevel(v), 100);
    Face.setLevel(v);
  });

  // ── music ───────────────────────────────────────────────
  const musicRow = group('Music');
  const playing = button(musicRow, 'Playing', b => {
    const on = !b.classList.contains('on');
    b.classList.toggle('on', on);
    Face.setMusic({ playing: on, bpm: state.bpm, energy: state.energy });
    if (!on) stopAutoBeat();
  });

  button(musicRow, 'Beat', () => Face.setMusic({ beat: true }));

  const auto = button(musicRow, 'Beat at tempo', b => {
    if (state.autoBeat) { stopAutoBeat(); return; }
    if (!playing.classList.contains('on')) playing.click();
    b.classList.add('on');
    state.autoBeat = setInterval(() => Face.setMusic({ beat: true }), 60000 / state.bpm);
  });

  function stopAutoBeat() {
    clearInterval(state.autoBeat);
    state.autoBeat = 0;
    auto.classList.remove('on');
  }

  slider(musicRow, 'bpm', 60, 190, 1, state.bpm, v => {
    state.bpm = v;
    if (playing.classList.contains('on')) Face.setMusic({ playing: true, bpm: v, energy: state.energy });
    if (state.autoBeat) { stopAutoBeat(); auto.click(); }
  });

  slider(musicRow, 'energy', 0, 1, 0.05, state.energy, v => {
    state.energy = v;
    if (playing.classList.contains('on')) Face.setMusic({ playing: true, bpm: state.bpm, energy: v });
  });

  // ── props ───────────────────────────────────────────────
  choice(group('Props'),
    [['auto', null], ['headphones on', true], ['headphones off', false]],
    value => Face.setProp('headphones', value),
    current.headphones === null ? null : !!current.headphones);

  // ── room ────────────────────────────────────────────────
  choice(group('Room'), HOURS, hour => Face.setHour(hour >= 0 ? hour : null), -1);

  // ── senses ──────────────────────────────────────────────
  // These are the only controls here that leave the renderer: a microphone and a
  // loopback are devices, and the face does not own one.
  const senseRow = group('Senses (these reach the host)');
  button(senseRow, 'Toggle listening', () => send({ type: 'listen' }));
  button(senseRow, 'Hush', () => send({ type: 'hush' }));

  // The same path the host's `look` takes, without needing a question that earns one.
  // Costs nothing and exercises the part that cannot be tested any other way: whether
  // this renderer is actually allowed to open a camera.
  button(senseRow, 'Take a still', b => {
    b.classList.add('on');
    look().finally(() => b.classList.remove('on'));
  });

  button(senseRow, 'Music sense on', () => send({ type: 'setMusic', value: true }));
  button(senseRow, 'Music sense off', () => send({ type: 'setMusic', value: false }));

  if (current.reduced) {
    const note = el('p', 'devnote',
      'This machine asks for reduced motion, so breathing and dancing are damped. ' +
      'Shapes still hold, but movement will look smaller than it is meant to.');
    body.appendChild(note);
  }

  return {
    /** Everything the panel started is stopped, so closing it leaves nothing running. */
    release() {
      clearInterval(levelHold);
      stopAutoBeat();
      levelInput.value = 0;
      Face.setLevel(0);
    }
  };
}
```
