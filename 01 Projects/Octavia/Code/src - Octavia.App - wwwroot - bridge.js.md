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
const hourSel = el('roomHour');
const musicChk = el('music');
const keyIn = el('key');
const keyRow = el('keyrow');
const keyPill = el('pillKey');
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
  if (!port || !token) return false;

  try {
    socket = new WebSocket(`ws://127.0.0.1:${port}/?token=${encodeURIComponent(token)}`);
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
}

/* The character the host is offering. Compared before loading, because a VRM is
   megabytes and every `hello` after the first would otherwise refetch it — but a
   *different* one, or none, must still take effect immediately. */
let avatarShowing = null;

/* A URL that failed is not retried this session. `hello` arrives on every settings
   change, and a face that cannot reach the avatar origin at all would otherwise refetch
   megabytes and log an error every time. */
const avatarFailed = new Set();

function adoptAvatar(url) {
  const wanted = url || null;
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

function applyHello(msg) {
  pill('pillBrain', msg.hasKey ? 'ok' : 'warn', msg.model || '—');
  pill('pillEars', msg.ears && msg.ears !== 'not started' ? 'ok' : 'dead', msg.ears || 'not started');
  pill('pillProfile', 'ok', msg.profile || '—');

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

  if (msg.music !== undefined) {
    musicChk.checked = !!msg.music;
    el('musicHint').textContent = !msg.music
      ? 'She is not listening to the speakers at all.'
      : msg.musicAvailable === false
        ? 'On, but this machine has no output she can listen to.'
        : 'Nothing is recorded: what survives is a tempo and a loudness.';
    if (!msg.music) showMusic({ playing: false });
  }

  if (msg.roomHour !== undefined) {
    hourSel.value = String(msg.roomHour);
    window.Face.setHour(msg.roomHour >= 0 ? msg.roomHour : null);
  }

  el('talk').setAttribute('aria-pressed', String(!!msg.listening));

  // A face that attached mid-session has missed whatever she is currently doing and
  // wearing, and neither is re-sent until it next changes.
  if (msg.state) applyState(msg.state);
  if (msg.emotion) window.Face.setEmotion(msg.emotion, msg.emotionWeight ?? 0);

  // The button only exists where the host would grant the permission behind it. A
  // button that always shows and usually fails would teach people it is broken.
  watchBtn.hidden = !msg.camera;
  if (!msg.camera && watcher) toggleWatch();

  offerDev(!!msg.dev);
  adoptAvatar(msg.avatar);
}

/* The missing key is the one status that has to lead somewhere. It lights a pill in
   the strip and marks the field it wants, rather than parking an input in the console
   where it is read every day and needed once. */
function wantKey(wanted) {
  keyPill.hidden = !wanted;
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
const watchBtn = el('watch');

async function toggleWatch() {
  if (watcher) {
    watcher.stop();
    watcher = null;
    window.Face.look(null);
    watchBtn.setAttribute('aria-pressed', 'false');
    return;
  }

  try {
    const { createWatcher } = await import('./watch.js');
    const next = createWatcher({
      onGaze: (x, y) => window.Face.look(x, y),
      onLive: live => {
        document.body.classList.toggle('watching', live);
        el('watching').hidden = !live;
      }
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
    const { takeStill } = await import('./camera.js');
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
  if (value) send({ type: 'say', text: value });
}

el('send').addEventListener('click', submitTyped);
textIn.addEventListener('keydown', e => { if (e.key === 'Enter') submitTyped(); });

el('talk').addEventListener('click', () => send({ type: 'listen' }));
hushBtn.addEventListener('click', () => send({ type: 'hush' }));
el('forget').addEventListener('click', () => send({ type: 'forget' }));

el('saveKey').addEventListener('click', () => {
  const value = keyIn.value.trim();
  if (!value) return;
  keyIn.value = '';
  send({ type: 'setKey', value });
});

keyIn.addEventListener('keydown', e => { if (e.key === 'Enter') el('saveKey').click(); });

// The pill is the signpost; Settings is where the thing actually is.
keyPill.addEventListener('click', () => {
  openDrawer('settings');
  keyIn.focus();
});

voiceSel.addEventListener('change', () => send({ type: 'setVoice', value: voiceSel.value }));
avatarSel.addEventListener('change', () => send({ type: 'setAvatar', value: avatarSel.value }));
engineSel.addEventListener('change', () => send({ type: 'setVoiceEngine', value: engineSel.value }));
musicChk.addEventListener('change', () => send({ type: 'setMusic', value: musicChk.checked }));
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
  if (e.code === 'Space' && !typing && !e.repeat) {
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
}

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

/* `ready` must reach the host however the transport settled, so it waits for the socket
   to resolve one way or the other rather than firing into a connecting socket. */
function announceReady() {
  send({ type: 'ready', faceBuilt: typeof window.Face === 'object' });
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
