---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\bridge.js
---

# src\Octavia.App\wwwroot\bridge.js

```javascript
/* Everything that crosses between the face and the host process.
   The face holds no key, makes no network calls, and owns no audio. */
(function () {
"use strict";

const el = id => document.getElementById(id);
const embedded = window.chrome && window.chrome.webview;

/* Transport. The socket is preferred even for the built-in page, so there is one code
   path and the embedded face is not a special case. postMessage is the fallback for
   when the socket could not bind. See PROTOCOL.md. */
const params = new URLSearchParams(location.search);
const port = params.get('port');
const token = params.get('token');

/* A face from another machine presents the durable remote key instead of the per-run
   token, which is scoped to a process on the host's own box. See PROTOCOL.md. */
const key = params.get('key');

/* Which room this face is in. Absent means the host's, so the built-in page and every
   renderer written before rooms existed keep behaving exactly as they did.

   **Deliberately not derived from the credential.** Token-means-loopback-means-host is
   tempting and wrong: two handsets would silently share one room, and a laptop on the LAN
   would be indistinguishable from a phone. A room is a statement of intent, so it is
   stated — and an Android client puts its native connection and its WebView panel in the
   same room by passing the same `?room=` to the page it loads. */
const room = params.get('room');

/* What this face can actually do, so `look` reaches a face that has a camera rather than
   whichever one spoke last.

   `isSecureContext` is the honest test and not a guess: `getUserMedia` does not exist
   outside one. The built-in page is served from the virtual `https://octavia.face` origin
   and qualifies; the same file served over plain `http://<lan-ip>` does not, and claiming
   a camera there would have the host asking for a frame that can never arrive. */
const canOpenACamera = window.isSecureContext;

/* Whatever this renderer is running inside, if it is running inside anything.

   A browser tab has no embedder and never will; an Android WebView, an iOS one or an
   Unreal shell can each provide one. **Absent is the normal case and stays the quiet one**
   — with no embedder every line below is inert and the page behaves exactly as it did.

   It exists because a face can be *less capable than the device it is standing in for*. A
   handset has a microphone and a camera; the page inside it, served over plain HTTP on the
   LAN, has neither — no secure context for `getUserMedia`, and no way to take the floor,
   because the floor is a FaceId and the native connection is a different face. Neither is
   fixable on the wire. So the renderer borrows.

   Deliberately **not** an Android interface. The page special-casing one client is how a
   renderer stops being a renderer. See `Stage 14 - Lending A Renderer The Device's Senses`.

   Read once. An embedder is injected at document start, before any page script, so there
   is nothing to wait for; one that appeared later would be describing a device this page
   had already told the host it did not have. */
const embedder = window.OctaviaEmbedder ?? null;

const lent = new Set(
  embedder && Array.isArray(embedder.senses) ? embedder.senses : []);

/* What this face tells the *host* it can do, and it deliberately does **not** include what
   an embedder lends.

   That looks like an oversight and is the opposite. `senses` routes `look`, which is
   answered by `camera.js` taking a still — and the embedder interface lends *watching*, not
   stills. A panel that claimed a camera here would be asked for a frame it cannot produce,
   and on Android it would take that frame away from the native client, which can. Borrowed
   senses are for this renderer's own controls; they are not a claim made to the host. */
const senses = canOpenACamera ? ['camera'] : [];

/* Where the socket is.

   The built-in page is served by WebView2 from a virtual host and is *not* served by the
   socket, so it cannot infer the port and is told it with `?port=`. Anything else — a
   browser on a tablet, the Android client — loaded this page over HTTP *from the socket
   itself*, so the socket is simply where the page came from. Hardcoding 127.0.0.1 was
   right while every face was on this machine and is exactly wrong for a phone: it would
   have the handset dial itself. */
function socketAddress() {
  const credential = token ? `token=${encodeURIComponent(token)}`
                   : key ? `key=${encodeURIComponent(key)}`
                   : null;
  if (!credential) return null;

  if (port) return `ws://127.0.0.1:${port}/?${credential}`;

  if (location.protocol === 'http:' || location.protocol === 'https:')
    return `${location.protocol === 'https:' ? 'wss:' : 'ws:'}//${location.host}/?${credential}`;

  return null;
}

let socket = null;
let socketReady = false;
let queued = [];

const captionEl = el('caption');
const speakerEl = el('speaker');
const stateLabel = el('stateLabel');
const entries = el('entries');
const voiceSel = el('voice');
const avatarSel = el('avatar');
const engineSel = el('voiceEngine');
const micSel = el('microphone');
const outSel = el('output');
const computeSel = el('whisperCompute');

/// Which camera she should open. Held here rather than in camera.js because that
/// module is imported lazily, and the setting arrives long before the first look.
let wantedCamera = '';
const hourSel = el('roomHour');
const musicChk = el('music');
const cameraChk = el('camera');
const cameraSel = el('cameraDevice');
const keyIn = el('key');
const keyRow = el('keyrow');
const statsChk = el('stats');
const textIn = el('text');
const hushBtn = el('hush');

const LABELS = { idle: 'Idle', listening: 'Listening', thinking: 'Thinking', speaking: 'Speaking' };

function send(message) {
  if (socketReady) { socket.send(JSON.stringify(message)); return; }
  if (socket && socket.readyState === WebSocket.CONNECTING) { queued.push(message); return; }
  if (embedded) { embedded.postMessage(message); return; }
  console.info('[no transport]', message);
}

function connectSocket() {
  const address = socketAddress();
  if (!address) return false;

  try {
    socket = new WebSocket(address);
  } catch (err) {
    console.warn('socket construction failed', err);
    return false;
  }

  socket.addEventListener('open', () => {
    socketReady = true;
    queued.forEach(m => socket.send(JSON.stringify(m)));
    queued = [];
  });

  socket.addEventListener('message', e => {
    try { receive(JSON.parse(e.data)); }
    catch (err) { console.error('bad host message', err, e.data); }
  });

  socket.addEventListener('close', () => {
    const wasReady = socketReady;
    socketReady = false;

    /* A held microphone must not survive this page losing its connection. The audio itself
       goes out over the *embedder's* socket rather than this one, so it would keep streaming
       from a panel whose button can no longer be trusted to report the release — and the
       host would hold the floor until its sixty-second timeout. */
    holdToTalk(false);

    // Only meaningful for a face outside the app; the built-in one dies with the host.
    if (wasReady && !embedded) notify('Lost the connection to Octavia.');
  });

  socket.addEventListener('error', () => {
    // If the socket never opened, fall back rather than leaving the face inert.
    if (!socketReady && embedded) {
      socket = null;
      queued.forEach(m => embedded.postMessage(m));
      queued = [];
    }
  });

  return true;
}

/* ── host → face ─────────────────────────────────────────── */

/* While the dev panel holds the face, host messages that would move it are dropped.
   Otherwise a mood set by hand is wiped by the next thing she says, which makes holding
   a shape still long enough to judge it impossible. Everything that is *information* —
   captions, the transcript, notices, settings — still arrives. */
let heldByDev = false;
const MOVES_HER = new Set(['state', 'level', 'viseme', 'emotion', 'music']);

function receive(msg) {
  if (heldByDev && MOVES_HER.has(msg.type)) return;

  switch (msg.type) {
    case 'state':
      applyState(msg.value);
      break;

    case 'level':
      window.Face.setLevel(msg.value);
      break;

    case 'viseme':
      window.Face.setViseme(msg.value, msg.shape);
      break;

    case 'emotion':
      window.Face.setEmotion(msg.value, msg.weight);
      break;

    case 'music':
      window.Face.setMusic(msg);
      if (!msg.beat) showMusic(msg);
      break;

    case 'caption':
      caption(msg.text, msg.who, msg.tentative);
      break;

    case 'turn':
      addTurn(msg.who, msg.text);
      break;

    case 'overheard':
      // She heard it and decided it was not for her. Shown rather than swallowed:
      // "she ignored me" has to be a question with a visible answer.
      caption(msg.text, 'Overheard', false, 'overheard');
      addOverheard(msg.text, msg.why);
      break;

    case 'notice':
      // While the splash is up it is the only surface there is, and a notice during
      // startup is almost always the long thing: a voice or a speech model downloading.
      if (document.body.classList.contains('loading')) splashNote.textContent = msg.text;
      notify(msg.text);
      break;

    case 'needKey':
      wantKey(true);
      break;

    case 'look':
      look();
      break;

    case 'cleared':
      entries.innerHTML = '<div class="empty">Nothing said yet.</div>';
      caption('', '');
      break;

    case 'hello':
      applyHello(msg);
      break;

    case 'diagnostics':
      renderDiagnostics(msg);
      break;
  }
}

/* State drives three things now, so it is worth one function: the avatar, the pill,
   and whether the console is showing a control for stopping her. */
function applyState(value) {
  window.Face.setState(value);
  document.body.dataset.state = value;
  stateLabel.textContent = LABELS[value] || value;

  // Hush appears only when there is something to hush. A permanent button that is dead
  // most of the time teaches people to ignore that corner of the screen.
  const busy = value === 'thinking' || value === 'speaking';
  hushBtn.hidden = !busy;
  document.body.classList.toggle('busy', busy);

  // Any state at all means she is working, so whatever went wrong before has passed.
  if (value !== 'idle') document.body.classList.remove('trouble');

  // Anything but idle brings the placard back immediately and restarts its clock.
  stayAwhile();
}

/* The character the host is offering. Compared before loading, because a VRM is
   megabytes and every `hello` after the first would otherwise refetch it — but a
   *different* one, or none, must still take effect immediately. */
let avatarShowing = null;

/* A URL that failed is not retried this session. `hello` arrives on every settings
   change, and a face that cannot reach the avatar origin at all would otherwise refetch
   megabytes and log an error every time. */
const avatarFailed = new Set();

/* The host names a character with the virtual-host URL *its own* WebView2 page can reach.
   Every other face loaded this page from the socket over HTTP, where `octavia.avatar` does
   not resolve at all — so point it back at the origin the page came from, which serves her
   avatars folder at `/avatars/`.

   Rewritten in the renderer rather than in `hello` because one `hello` is serialised once
   and broadcast to every attached face at once: the host cannot say something different to
   each of them, and the face is the only party that knows which of the two it is. */
const AVATAR_ORIGIN = 'https://octavia.avatar/';

function reachableAvatar(url) {
  if (!url || !url.startsWith(AVATAR_ORIGIN)) return url;
  if (port) return url;   // the built-in page, where that virtual host is real
  return '/avatars/' + url.slice(AVATAR_ORIGIN.length);
}

function adoptAvatar(url) {
  const wanted = reachableAvatar(url) || null;
  if (wanted === avatarShowing || (wanted && avatarFailed.has(wanted))) return;
  avatarShowing = wanted;

  if (!wanted) {
    window.Face.useBust();
    return;
  }

  window.Face.loadAvatar(wanted).then(meta => {
    notify(meta?.name ? `${meta.name} is here.` : 'Avatar loaded.');
  }).catch(err => {
    // The bust is still on screen, so this is a message rather than a catastrophe —
    // but it must reach the log, or "she looks wrong" is the whole bug report.
    avatarShowing = null;
    avatarFailed.add(wanted);
    window.Face.useBust();
    send({ type: 'faceError', text: `avatar ${wanted} failed to load: ${err && err.message}` });
    notify('That avatar could not be loaded; keeping the bust.');
  });
}

/* Options are rebuilt only when the set of them changes: replacing them on every
   `hello` would close the dropdown under the pointer mid-choice. */
function fill(select, options, current) {
  const signature = options.map(o => o.value).join(' ');
  if (select.dataset.signature !== signature) {
    select.dataset.signature = signature;
    select.innerHTML = '';
    options.forEach(option => {
      const el = document.createElement('option');
      el.value = option.value;
      el.textContent = option.label;
      select.appendChild(el);
    });
  }
  if (current !== undefined && current !== null) select.value = String(current);
}

/* A status pill says two things: what she is running, and whether it is well. The dot
   is the second half, and it is the reason the strip can be read rather than studied. */
function pill(id, health, text) {
  const node = el(id);
  if (!node) return;
  node.classList.remove('warn', 'dead', 'bad');
  if (health !== 'ok') node.classList.add(health);
  if (text !== undefined) node.querySelector('b').textContent = text;
}

/* What this face may drive. `host` is everything, as it always was; anything else hides the
   controls that act on the machine she runs on — her microphone, her speakers, her music
   listening, Whisper's compute, her data folder, the diagnostics file.

   **This is a hint, not the enforcement.** The host refuses those messages by the room they
   came from, which is the half that matters: a face that can send `listen` by hand could
   otherwise still open a microphone in an empty house. Both are needed — without the guard
   a remote face drives the hardware anyway, and without the hint a phone shows a microphone
   button that silently does nothing, which is its own kind of broken.

   Default `host`, so a face talking to a host that predates this field loses nothing. */
let controls = 'host';

function applyControls(value) {
  controls = value === 'room' ? 'room' : 'host';
  const hidden = controls !== 'host';

  // `style.display` rather than `hidden`, because these rows are laid out with `display`
  // and a class rule would win against the attribute.
  document.querySelectorAll('[data-host-only]')
          .forEach(node => { node.style.display = hidden ? 'none' : ''; });

  /* The microphone is not one of them, and it is the only control with three answers
     rather than two. In the host room it toggles `listen` as it always has. In any other
     room it is hidden — *unless* an embedder lends a microphone, in which case it comes
     back as press-and-hold. See the note on `holdToTalk`. */
  talkBtn.hidden = hidden && !lent.has('mic');

  const holding = hidden && lent.has('mic');
  talkBtn.title = holding ? 'Hold to talk' : 'Listen (Ctrl+Alt+O)';
  talkBtn.setAttribute('aria-label', holding ? 'Hold to talk' : 'Toggle listening');
}

function applyHello(msg) {
  // The host answering *is* the second splash step; the voice is the third.
  splashStep('host', true);
  splashStep('voice', !!msg.voice);

  pill('pillBrain', msg.hasKey ? 'ok' : 'warn', msg.model || '—');
  pill('pillEars', msg.ears && msg.ears !== 'not started' ? 'ok' : 'dead', msg.ears || 'not started');

  /* The room is shown beside the profile when it is not the host's. That is the whole
     reason `hello` echoes it back: `?room=phne` would otherwise put this face in a room of
     its own, silently, and look exactly like her ignoring it. */
  pill('pillProfile', 'ok',
    msg.room && msg.room !== 'host' ? `${msg.profile || '—'} · ${msg.room}` : (msg.profile || '—'));

  applyControls(msg.controls);

  wantKey(!msg.hasKey);

  // The host labels its own voices now: a Piper file name and a SAPI name need very
  // different tidying, and only the host knows which engine it is running.
  const voices = (msg.voices || []).map(v =>
    typeof v === 'string' ? { value: v, label: v.replace(/^Microsoft\s+/i, '') } : v);

  fill(voiceSel, voices, msg.voice);
  const chosen = voices.find(v => v.value === msg.voice);
  pill('pillVoice', msg.voice ? 'ok' : 'dead', chosen ? chosen.label : (msg.voice || '—'));

  if (msg.voiceEngine) engineSel.value = msg.voiceEngine;
  el('voiceHint').textContent = msg.voiceEngine === 'neural'
    ? 'Downloaded on first use. Choosing one she does not have yet fetches it.'
    : 'Windows speech voices installed on this machine.';

  fill(avatarSel,
    [{ value: '', label: 'Plaster bust' }]
      .concat((msg.avatars || []).map(file => ({ value: file, label: prettyAvatar(file) }))),
    msg.avatarFile ?? '');

  /* Devices. "Follow the Windows default" is an option rather than an absence, so the
     menu can say which one that currently is instead of leaving it blank. */
  if (msg.microphones) {
    fill(micSel,
      [{ value: '', label: 'Windows default' }]
        .concat(msg.microphones.map(d => ({ value: d.value, label: d.label }))),
      msg.microphone ?? '');
  }

  if (msg.outputs) {
    fill(outSel,
      [{ value: '', label: 'Windows default' }]
        .concat(msg.outputs.map(d => ({ value: d.value, label: d.label }))),
      msg.output ?? '');
  }

  if (msg.whisperCompute) computeSel.value = msg.whisperCompute;

  if (msg.stats !== undefined) {
    statsChk.checked = !!msg.stats;
    document.body.classList.toggle("no-stats", !msg.stats);
  }

  if (msg.cameraDevice !== undefined) wantedCamera = msg.cameraDevice;

  if (msg.camera !== undefined) {
    cameraChk.checked = !!msg.camera;
    // The picker is useless while the sense is off, and leaving it live would imply
    // choosing a camera does something on its own.
    cameraSel.disabled = !msg.camera;

    // Enumerating means importing camera.js, and a face that is never asked to look should
    // never load any camera code — so while the sense is off the menu is filled with the
    // one option that is true without asking anything. A disabled *empty* select reads as
    // broken rather than as switched off.
    if (msg.camera) listCameras();
    else fill(cameraSel, [{ value: '', label: 'Whichever the browser picks' }], wantedCamera);
  }

  if (msg.music !== undefined) {
    musicChk.checked = !!msg.music;
    /* Naming the endpoint matters more than the reassurance about privacy. She taps one
       output; a machine has several, and music playing through any of the others is
       silence to her — which looks exactly like the beat detection being broken. */
    const heard = (msg.outputs || []).find(d => d.value === (msg.output || ''))
                ?? (msg.outputs || []).find(d => /default/i.test(d.label));

    el('musicHint').textContent = !msg.music
      ? 'She is not listening to the speakers at all.'
      : msg.musicAvailable === false
        ? 'On, but this machine has no output she can listen to.'
        : heard
          ? `Listening to ${heard.label.replace(/\s*\(Windows default\)$/, '')}. Play music through that one — she hears a single output, not the machine. Nothing is recorded: a tempo and a loudness.`
          : 'Nothing is recorded: what survives is a tempo and a loudness.';
    if (!msg.music) showMusic({ playing: false });
  }

  if (msg.roomHour !== undefined) {
    hourSel.value = String(msg.roomHour);
    window.Face.setHour(msg.roomHour >= 0 ? msg.roomHour : null);
  }

  // Only in the host room. Elsewhere this button is held rather than toggled, and its
  // pressed state belongs to the finger on it rather than to her microphone.
  if (controls === 'host') talkBtn.setAttribute('aria-pressed', String(!!msg.listening));

  // A face that attached mid-session has missed whatever she is currently doing and
  // wearing, and neither is re-sent until it next changes.
  if (msg.state) applyState(msg.state);
  if (msg.emotion) window.Face.setEmotion(msg.emotion, msg.emotionWeight ?? 0);

  /* Two questions, and it took a handset to notice they were different.

     `msg.camera` is the **room's** answer to "may she look at all". `canOpenACamera` is
     *this renderer's* answer to "could I, physically". Watching is renderer-local — the
     motion centroid never leaves the page — so it needs both, and it used to ask only the
     first. On a plain `http://<lan-ip>` origin `navigator.mediaDevices` is undefined, so a
     remote face was offered a button whose only possible outcome was
     `Cannot read properties of undefined (reading 'getUserMedia')`.

     The setting itself stays visible on such a face on purpose: it belongs to the room, and
     in that room the native client is the one that answers `look`. */
  const couldWatch = canOpenACamera || lent.has('camera');
  watchBtn.hidden = !msg.camera || !couldWatch;
  if ((!msg.camera || !couldWatch) && watcher) toggleWatch();

  offerDev(!!msg.dev);
  adoptAvatar(msg.avatar);
}

/* The missing key marks the field that wants it, in Settings, and nowhere else.
   It used to light an amber pill across the bottom of the room as well — which nagged
   whoever was looking at her rather than whoever could act on it, and a permanent
   warning is one you stop seeing. */
function wantKey(wanted) {
  keyRow.classList.toggle('wanted', wanted);
}

/* The one place the tempo is written down. Without it "she is not dancing" and "there
   is nothing to dance to" look exactly the same from the sofa. */
function showMusic(msg) {
  pill('pillMusic', msg.playing ? 'ok' : 'dead',
    msg.playing ? (msg.bpm ? `${Math.round(msg.bpm)} bpm` : 'playing') : '—');
}

/** "octavia-v2.vrm" reads better as "Octavia v2" in a menu. */
function prettyAvatar(file) {
  const name = file.replace(/\.vrm$/i, '').replace(/[-_]+/g, ' ').trim();
  return name.charAt(0).toUpperCase() + name.slice(1);
}

if (embedded) embedded.addEventListener('message', e => {
  // Ignored while the socket is carrying traffic, so a message is never acted on twice.
  if (socketReady) return;
  try { receive(e.data); }
  catch (err) { console.error('bad host message', err, e.data); }
});

connectSocket();

/* ── watching ────────────────────────────────────────────── */

/* Continuous, and therefore different in kind from a still: only a person's press
   starts it, and nothing about it crosses the protocol — the gaze is computed here and
   dies here. The host learns nothing, which is the point. */
let watcher = null;
const talkBtn = el('talk');
const watchBtn = el('watch');

async function toggleWatch() {
  if (watcher) {
    watcher.stop();
    watcher = null;
    window.Face.look(null);
    watchBtn.setAttribute('aria-pressed', 'false');
    return;
  }

  const live = on => {
    document.body.classList.toggle('watching', on);
    el('watching').hidden = !on;
  };

  try {
    /* A borrowed camera wins over this page's own. On a handset the page has no camera at
       all — plain HTTP, no secure context — and where both exist the embedder is the one
       holding the device the person is looking at.

       The embedder drives `window.Face.look(x, y)` itself and stops with `look(null)`; it
       computes the gaze on its side and **nothing crosses the socket either way**, which is
       the same promise `watch.js` makes and the reason the host is never told this mode
       exists at all.

       The marker stays here on purpose. One privacy marker, in the page, where a person
       already looks — an embedder drawing its own instead would be two things to trust
       rather than one. */
    const next = lent.has('camera')
      ? {
          // Awaited, so an embedder that opens its camera asynchronously can fail loudly
          // rather than leaving the marker up over a camera that never started.
          start: async () => { await embedder.watch(true); live(true); },
          stop: () => { embedder.watch(false); live(false); }
        }
      : (await import('./watch.js')).createWatcher({
          onGaze: (x, y) => window.Face.look(x, y),
          onLive: live
        });

    await next.start();
    watcher = next;
    watchBtn.setAttribute('aria-pressed', 'true');
  } catch (err) {
    const why = (err && err.name === 'NotAllowedError') ? 'permission refused'
              : (err && err.name === 'NotFoundError') ? 'no camera on this machine'
              : String(err && err.message || err);
    notify(`She could not watch: ${why}.`);
    send({ type: 'faceError', text: `watch failed: ${why}` });
  }
}

watchBtn.addEventListener('click', () => { toggleWatch(); });

/* ── eyes ────────────────────────────────────────────────── */

/* The host asks; the face answers once, or explains why it could not. It always answers:
   silence would leave her standing there for twenty seconds waiting for a frame that is
   never coming. The module is imported on first use, so a face that is never asked to
   look never loads any camera code at all. */
async function look() {
  try {
    const { takeStill, useCamera } = await import('./camera.js');
    useCamera(wantedCamera);
    const image = await takeStill(live => document.body.classList.toggle('looking', live));
    send({ type: 'sight', image });

    // The red bar is only up for a moment. Saying so afterwards is the difference
    // between a camera you trust and one you merely tolerate.
    notify('She took one still.');
  } catch (err) {
    const why = (err && err.name === 'NotAllowedError') ? 'permission refused'
              : (err && err.name === 'NotFoundError') ? 'no camera on this machine'
              : String(err && err.message || err);

    document.body.classList.remove('looking');
    send({ type: 'sight', error: why });
    notify(`She could not look: ${why}.`);
  }
}

/* ── face → host ─────────────────────────────────────────── */

function submitTyped() {
  const value = textIn.value.trim();
  textIn.value = '';
  if (!value) return;
  send({ type: 'say', text: value });
  // The field stays. Somebody who typed once is usually about to type again, and
  // closing it under them meant re-opening it for every single line.
  keepFieldAwhile();
  textIn.focus();
}

el('send').addEventListener('click', submitTyped);
/* Typing is opt-in. Opening focuses the field, because someone who clicked the keyboard
   wants to type, not to look at a box. */
const typeBtn = el('typeBtn');
const fieldWrap = document.querySelector('.field');

/* Closing it is the keyboard button's job, or a minute of not using it — whichever
   comes first. The timer only ever fires on an *empty* field: a half-written line is
   somebody still thinking, and reclaiming a strip of chrome is not worth throwing
   their sentence away. */
const FIELD_IDLE = 60000;
let fieldIdle = null;

function keepFieldAwhile() {
  clearTimeout(fieldIdle);
  fieldIdle = setTimeout(() => {
    if (!fieldWrap.hidden && !textIn.value.trim()) showField(false);
  }, FIELD_IDLE);
}

function showField(on) {
  fieldWrap.hidden = !on;
  typeBtn.setAttribute('aria-expanded', String(on));
  clearTimeout(fieldIdle);
  if (on) { textIn.focus(); keepFieldAwhile(); }
  else textIn.value = '';
}

typeBtn.addEventListener('click', () => showField(fieldWrap.hidden));

// Any keystroke counts as using it, including the ones that leave it empty again.
textIn.addEventListener('input', keepFieldAwhile);

textIn.addEventListener('keydown', e => {
  if (e.key === 'Enter') submitTyped();
  // Esc closes the field; if she is talking, the global handler stops her instead.
  if (e.key === 'Escape' && !document.body.classList.contains('busy')) showField(false);
});

// `listen` toggles the **host machine's** microphone, which is why it is guarded here as
// well as hidden: a phone at the gym pressing this would open a microphone in an empty
// house. The host refuses it too; this stops the request being made at all.
//
// On a room face the click does nothing and the pointer handlers below take over.
talkBtn.addEventListener('click', () => { if (controls === 'host') send({ type: 'listen' }); });

/* Press-and-hold, for a room face with a borrowed microphone.

   **The one place the interface cannot match the host, and it is worth saying rather than
   hiding.** The desktop's button is a *toggle*: `listen` opens her microphone and leaves it
   open, and the attention gate decides what was addressed to her. A remote room cannot have
   that yet — an open microphone beside a speaker playing her voice, across a network with
   latency each way, is the echo problem Stage 14 item 6 deferred. So: same button, same
   place, same look, held rather than toggled. Making it a toggle needs real echo
   cancellation, not a smarter button.

   The embedder owns the socket that streams, so the page never sees a sample — it only says
   when. That is also why this cannot be done on the wire: the floor is a FaceId, and the
   native connection is a different face from this panel. */
let holdingFloor = false;

function holdToTalk(on) {
  if (!lent.has('mic') || controls === 'host') return;
  if (holdingFloor === on) return;

  holdingFloor = on;
  talkBtn.setAttribute('aria-pressed', String(on));

  try {
    embedder.talking(on);
  } catch (err) {
    holdingFloor = false;
    talkBtn.setAttribute('aria-pressed', 'false');
    notify('The microphone could not be reached.');
    send({ type: 'faceError', text: `talking failed: ${err && err.message || err}` });
  }
}

/* Every way a press can end, and they all release.

   A held button that never releases holds her ears until the host's sixty-second timeout,
   which is the failure this cannot be allowed to have. `pointerup` is the ordinary one;
   `pointerleave` makes dragging off cancel, which is what a person expects; `pointercancel`
   covers the system taking the gesture away, which on a phone is a scroll or a call
   arriving. `blur` and `visibilitychange` cover the app going to the background mid-press.
   `holdToTalk` is idempotent, so the overlap costs nothing. */
talkBtn.addEventListener('pointerdown', e => { e.preventDefault(); holdToTalk(true); });
talkBtn.addEventListener('pointerup', () => holdToTalk(false));
talkBtn.addEventListener('pointerleave', () => holdToTalk(false));
talkBtn.addEventListener('pointercancel', () => holdToTalk(false));
window.addEventListener('blur', () => holdToTalk(false));
document.addEventListener('visibilitychange', () => { if (document.hidden) holdToTalk(false); });
hushBtn.addEventListener('click', () => send({ type: 'hush' }));
el('forget').addEventListener('click', () => send({ type: 'forget' }));

el('saveKey').addEventListener('click', () => {
  const value = keyIn.value.trim();
  if (!value) return;
  keyIn.value = '';
  send({ type: 'setKey', value });
});

keyIn.addEventListener('keydown', e => { if (e.key === 'Enter') el('saveKey').click(); });

/* The camera list comes from the **face**, not the host — unlike the microphone and the
   output, which are the host's devices. `camera.js` enumerates them and matches by *label*
   rather than deviceId, because an id is regenerated per origin and per permission grant,
   so it cannot be stored in a config file and still mean anything tomorrow.

   A browser withholds device labels from a page that has never been granted the permission,
   so this list is empty until she has looked once. Saying so is the difference between a
   menu that is waiting and a menu that looks broken. */
async function listCameras() {
  let found = [];

  try {
    const { cameras } = await import('./camera.js');
    found = await cameras();
  } catch (err) {
    console.warn('could not list the cameras', err);
  }

  fill(cameraSel,
    [{ value: '', label: found.length ? 'Whichever the browser picks' : 'Not known yet' }]
      .concat(found.map(label => ({ value: label, label }))),
    wantedCamera);

  el('cameraHint').textContent = found.length
    ? 'Which one she looks through.'
    : canOpenACamera
      ? 'A browser will not name the cameras until she has been allowed to look once. Ask her something that needs eyes, allow it, then come back.'
      // The same distinction as the watch button. An empty list here has two very different
      // causes, and telling somebody on a phone to "allow it once" sends them looking for a
      // permission prompt that can never appear.
      : 'This face cannot open a camera at all — it was loaded over plain HTTP, and a browser only offers cameras to a secure origin. The setting still applies to this room; whichever face here owns a camera answers.';
}

cameraChk.addEventListener('change', () => send({ type: 'setCamera', value: cameraChk.checked }));
cameraSel.addEventListener('change', () => {
  // Only the choice is recorded here. `look()` hands it to the module on the next look,
  // which is what keeps the promise in camera.js that a face never asked to look loads
  // no camera code at all — and this listener has no `useCamera` in scope anyway.
  wantedCamera = cameraSel.value;
  send({ type: 'setCameraDevice', value: cameraSel.value });
});

voiceSel.addEventListener('change', () => send({ type: 'setVoice', value: voiceSel.value }));
avatarSel.addEventListener('change', () => send({ type: 'setAvatar', value: avatarSel.value }));
engineSel.addEventListener('change', () => send({ type: 'setVoiceEngine', value: engineSel.value }));
micSel.addEventListener('change', () => send({ type: 'setMicrophone', value: micSel.value }));
outSel.addEventListener('change', () => send({ type: 'setOutput', value: outSel.value }));
computeSel.addEventListener('change', () => send({ type: 'setWhisperCompute', value: computeSel.value }));
musicChk.addEventListener('change', () => send({ type: 'setMusic', value: musicChk.checked }));
statsChk.addEventListener('change', () => send({ type: 'setStats', value: statsChk.checked }));
hourSel.addEventListener('change', () => {
  const hour = Number(hourSel.value);
  // Applied here as well as sent, so the room changes while the click is still warm
  // rather than waiting for the settings to come back around.
  window.Face.setHour(hour >= 0 ? hour : null);
  send({ type: 'setRoomHour', value: hour });
});
el('openData').addEventListener('click', () => send({ type: 'openDataFolder' }));

document.addEventListener('keydown', e => {
  const typing = document.activeElement === textIn || document.activeElement === keyIn;
  if (e.code === 'Space' && !typing && !e.repeat && controls === 'host') {
    e.preventDefault();
    send({ type: 'listen' });
  }
  if (e.key === 'Escape') {
    if (drawer.classList.contains('on')) { closeDrawer(); return; }
    send({ type: 'hush' });
  }
});

/* ── placard, transcript, notices ────────────────────────── */

function caption(text, who, tentative, extra) {
  captionEl.textContent = text || '…';
  captionEl.className = !text ? 'muted' : (extra || '');
  if (tentative) captionEl.classList.add('tentative');
  speakerEl.textContent = who || ' ';
  stayAwhile();
}

/* How long the last thing said stays on screen once everything has gone quiet.
   Not zero: a reply that vanishes the instant she stops speaking is unreadable, and the
   caption is the only record outside the transcript. Long enough to finish reading a
   couple of sentences, then the room takes the space back. */
const LINGER = 9000;
let lingering = null;

function stayAwhile() {
  clearTimeout(lingering);
  document.body.classList.remove('quiet');

  // Only idle earns the collapse. While she is listening, thinking or speaking there is
  // something about to appear there, and collapsing between every turn would be a
  // twitch rather than a room.
  lingering = setTimeout(() => {
    if (document.body.dataset.state === 'idle') document.body.classList.add('quiet');
  }, LINGER);
}

// The opening prompt is subject to the same rule: read it, then have the room back.
stayAwhile();

function addTurn(who, text) {
  const empty = entries.querySelector('.empty');
  if (empty) entries.innerHTML = '';

  const wrap = document.createElement('div');
  wrap.className = 'turn' + (who === 'you' ? ' me' : '');

  const label = document.createElement('span');
  label.className = 'who';
  label.textContent = who === 'you' ? 'You' : 'Octavia';

  const body = document.createElement('p');
  body.textContent = text;

  wrap.append(label, body);
  entries.appendChild(wrap);
  entries.scrollTop = entries.scrollHeight;
}

/* Kept in the transcript rather than only on the placard, because the interesting
   question is usually "what has she been ignoring?" — which is a list, not a moment. */
function addOverheard(text, why) {
  const empty = entries.querySelector('.empty');
  if (empty) entries.innerHTML = '';

  const wrap = document.createElement('div');
  wrap.className = 'turn overheard';

  const label = document.createElement('span');
  label.className = 'who';
  label.textContent = why ? `overheard — ${why}` : 'overheard';

  const body = document.createElement('p');
  body.textContent = text;

  wrap.append(label, body);
  entries.appendChild(wrap);
  entries.scrollTop = entries.scrollHeight;
}

/* ── the drawer ──────────────────────────────────────────── */

/* One component, four tabs. It replaces three hand-written drawers that each had their
   own header, close button and slide — so a change to drawer behaviour was three edits
   and a chance to miss one. Stage 8's second renderer needs this built once, not four
   times. */
const drawer = el('drawer');
const drawerBtn = el('drawerBtn');
const tabs = [...document.querySelectorAll('.tab')];
let devReady = null;
let tested = false;

function openDrawer(tab) {
  drawer.classList.add('on');
  drawer.setAttribute('aria-hidden', 'false');
  drawerBtn.setAttribute('aria-expanded', 'true');
  if (tab) selectTab(tab);
}

function closeDrawer() {
  drawer.classList.remove('on');
  drawer.setAttribute('aria-hidden', 'true');
  drawerBtn.setAttribute('aria-expanded', 'false');
}

function selectTab(name) {
  tabs.forEach(tab => {
    const on = tab.dataset.tab === name;
    tab.classList.toggle('on', on);
    tab.setAttribute('aria-selected', String(on));
  });
  document.querySelectorAll('.dbody').forEach(body => {
    body.hidden = body.dataset.body !== name;
  });

  // Opening Health is the question; asking for the answer as well saves a click at the
  // exact moment someone is already frustrated.
  if (name === 'health' && !tested) { tested = true; send({ type: 'selfTest' }); }

  if (name === 'dev' && !devReady) {
    import('./dev.js').then(({ createDevPanel }) => {
      devReady = createDevPanel({
        send,
        look,
        onHoldChanged: held => { heldByDev = held; }
      });
    }).catch(err => send({ type: 'faceError', text: `dev panel failed: ${err && err.message}` }));
  }
}

drawerBtn.addEventListener('click', () =>
  drawer.classList.contains('on') ? closeDrawer() : openDrawer());
el('drawerClose').addEventListener('click', closeDrawer);
tabs.forEach(tab => tab.addEventListener('click', () => selectTab(tab.dataset.tab)));

/* Offered when the host says it is running the dev profile, and when there is no host at
   all — a face served on its own is being worked on by definition. A published Octavia
   on the live profile never shows the tab and never loads the module. */
function offerDev(on) {
  const tab = tabs.find(t => t.dataset.tab === 'dev');
  if (tab) tab.hidden = !on;
}

/* ── health ──────────────────────────────────────────────── */

const diagBody = el('diagBody');

el('diagRun').addEventListener('click', () => send({ type: 'selfTest' }));
el('diagSave').addEventListener('click', () => send({ type: 'saveDiagnostics' }));

function placeholder(text) {
  diagBody.innerHTML = '';
  const div = document.createElement('div');
  div.className = 'empty';
  div.textContent = text;
  diagBody.appendChild(div);
}

function renderDiagnostics(msg) {
  if (msg.running) { placeholder('Testing. The microphone check listens for a second...'); return; }
  if (!msg.checks) { placeholder('The self-test could not run. Her log has the detail.'); return; }

  diagBody.innerHTML = '';
  msg.checks.forEach(check => diagBody.appendChild(checkRow(check)));

  if (msg.facts) diagBody.append(heading('This machine'), factTable(msg.facts));
  if (msg.log && msg.log.length) diagBody.append(heading('Recent log'), recentLog(msg.log));

  diagBody.scrollTop = 0;
}

function checkRow(check) {
  const row = document.createElement('div');
  row.className = 'check' + (check.ok ? '' : ' bad');

  const mark = document.createElement('span');
  mark.className = 'mark';
  mark.textContent = check.ok ? 'ok' : 'problem';

  const body = document.createElement('div');
  body.className = 'body';

  const name = document.createElement('span');
  name.className = 'name';
  name.textContent = check.name;

  const detail = document.createElement('p');
  detail.className = 'detail';
  detail.textContent = check.detail;

  body.append(name, detail);

  // A red line that does not say what to try is barely better than no line.
  if (!check.ok && check.fix) {
    const fix = document.createElement('p');
    fix.className = 'fix';
    fix.textContent = check.fix;
    body.appendChild(fix);
  }

  row.append(mark, body);
  return row;
}

function heading(text) {
  const h = document.createElement('h3');
  h.textContent = text;
  return h;
}

function factTable(facts) {
  const table = document.createElement('table');
  table.className = 'facts';
  facts.forEach(fact => {
    const row = document.createElement('tr');
    const name = document.createElement('th');
    name.textContent = fact.name;
    const value = document.createElement('td');
    value.textContent = fact.value;
    row.append(name, value);
    table.appendChild(row);
  });
  return table;
}

function recentLog(lines) {
  const pre = document.createElement('pre');
  pre.id = 'recent';
  pre.textContent = lines.join('\n');
  return pre;
}

let noticeTimer = 0;
function notify(text) {
  const n = el('notice');
  n.textContent = text;
  n.classList.add('on');
  clearTimeout(noticeTimer);
  noticeTimer = setTimeout(() => n.classList.remove('on'), 7000);

  // Trouble is a designed state rather than a missing one: the state pill turns red
  // and stays red until she does something else.
  if (/could not|failed|went wrong|no route|cannot/i.test(text)) {
    document.body.classList.add('trouble');
  }
}

/* The host has no console. Anything the face throws goes into octavia.log instead. */
window.addEventListener('error', e => {
  send({ type: 'faceError', text: `${e.message} (${e.filename}:${e.lineno})` });
});
window.addEventListener('unhandledrejection', e => {
  send({ type: 'faceError', text: String(e.reason) });
});

/* ── splash ──────────────────────────────────────────────────
   Held until the scene has built and the host has answered. It reports which step it is
   on because a splash that cannot say why it is still there is indistinguishable from a
   hang — and the step that usually takes the time, fetching a voice or a speech model,
   is exactly the one worth naming. */
const splashSteps = el('splashSteps');
const splashNote = el('splashNote');
const splashDone = new Set();

function splashStep(name, done) {
  if (done) splashDone.add(name);

  const order = ['renderer', 'host', 'voice'];
  let waiting = null;

  for (const step of order) {
    const li = splashSteps.querySelector(`[data-step="${step}"]`);
    if (!li) continue;

    if (splashDone.has(step)) { li.classList.add('done'); li.classList.remove('now'); }
    else if (!waiting) { waiting = step; li.classList.add('now'); }
  }

  // The voice arrives late on a first run and is not worth holding the room hostage
  // for; the renderer and the host are.
  if (splashDone.has('renderer') && splashDone.has('host')) finishSplash();
}

let splashClosed = false;
function finishSplash() {
  if (splashClosed) return;
  splashClosed = true;
  // A beat, so the last step is legibly ticked rather than flashing past.
  setTimeout(() => document.body.classList.remove('loading'), 420);
}

// Never leave someone looking at a splash because a step we were waiting on never
// arrived. Whatever is wrong is better seen through the console than behind it.
setTimeout(() => {
  if (!splashClosed) {
    splashNote.textContent = 'Taking longer than expected — opening anyway.';
    setTimeout(finishSplash, 1200);
  }
}, 15000);

/* `ready` must reach the host however the transport settled, so it waits for the socket
   to resolve one way or the other rather than firing into a connecting socket. */
function announceReady() {
  const built = typeof window.Face === 'object';
  splashStep('renderer', built);
  send({ type: 'ready', faceBuilt: built, room: room || undefined, senses });
}

if (socket) {
  socket.addEventListener('open', announceReady);
  socket.addEventListener('error', () => setTimeout(announceReady, 0));
} else {
  announceReady();
  if (!embedded) {
    offerDev(true);
    notify('No host to connect to. The face renders; use the Dev tab to drive it.');
  }
}

})();
```
