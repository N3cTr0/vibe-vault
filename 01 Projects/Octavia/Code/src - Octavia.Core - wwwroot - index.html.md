---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\wwwroot\index.html
---

# src\Octavia.Core\wwwroot\index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta http-equiv="Content-Security-Policy"
      content="default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data: blob: https://octavia.avatar; media-src 'self' blob:; connect-src 'self' blob: ws://127.0.0.1:* ws://localhost:* https://octavia.avatar">
<title>Octavia</title>
<!-- Since v0.20.0 the socket serves this page, so a phone or a browser tab is a face like
     any other — and a face deserves her mark rather than a globe. `apple-touch-icon` is
     what iOS uses when someone adds her to a home screen; `icon-192` is Android's. -->
<link rel="icon" type="image/png" sizes="32x32" href="favicon-32.png">
<link rel="icon" type="image/png" sizes="192x192" href="icon-192.png">
<link rel="apple-touch-icon" href="apple-touch-icon.png">
<link rel="stylesheet" href="face.css">
</head>
<body data-state="idle" class="loading">

<!-- Held until the scene has built *and* the host has said hello. Before this existed she
     showed a finished-looking console while the renderer, the socket and the voice were
     all still coming up, and the gap between looking ready and being ready is where every
     "she ignored me" report started. It names what it is waiting for rather than
     spinning: a splash that cannot say why it is still there is just a delay. -->
<div id="splash" role="status" aria-live="polite">
  <div class="splash-inner">
    <span class="mark">Octavia<em>.</em></span>
    <ul id="splashSteps">
      <li data-step="renderer">Building the room</li>
      <li data-step="host">Reaching the host</li>
      <li data-step="voice">Finding her voice</li>
    </ul>
    <p id="splashNote"></p>
  </div>
</div>

<div id="app">
  <header>
    <span class="mark">Octavia<em>.</em></span>
    <span class="spacer"></span>
    <span id="watching" hidden>camera</span>

    <span id="state"><span class="dot"></span><span id="stateLabel">Idle</span></span>

    <!-- Last in the row, outboard of the state. The drawer is the way out of the room;
         the state is about the room, so it reads first. -->
    <button id="drawerBtn" title="Transcript, settings and health" aria-label="Open the drawer" aria-expanded="false">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round">
        <path d="M4 7h16M4 12h16M4 17h16"/>
      </svg>
    </button>
  </header>

  <div id="stage">
    <canvas id="scene"></canvas>

    <!-- A readout, not a control surface, and now a *quiet* one: floated over the room
         at the top left on translucent glass rather than filling a strip of chrome.
         It can be switched off entirely — see "Show the status readout" in Settings.
         The missing-key warning that used to live here has gone to Settings, where it
         can nag the person who can act on it instead of the person looking at her. -->
    <div class="meta">
      <span class="pill" id="pillVoice"><span class="d"></span>Voice <b id="voiceLabel">&mdash;</b></span>
      <span class="pill" id="pillEars"><span class="d"></span>Ears <b id="ears">not started</b></span>
      <span class="pill" id="pillBrain"><span class="d"></span>Brain <b id="model">&mdash;</b></span>
      <span class="pill" id="pillMusic"><span class="d"></span>Music <b id="musicLabel">&mdash;</b></span>
      <span class="pill" id="pillProfile"><span class="d"></span>Profile <b id="profile">&mdash;</b></span>
    </div>

    <!-- Inside the stage, over the room, like a subtitle. It used to be a sibling below
         it, which meant hiding it changed the stage's height — and a stage that changes
         height re-frames the camera, so she visibly jumped size every time the caption
         came and went. Overlaid, the room is always full height and nothing moves but
         the text. -->
    <div id="placard">
      <span id="speaker">&nbsp;</span>
      <p id="caption" class="muted">Press the microphone, or say her name.</p>
    </div>

    <!-- What survived the status readout going off by default: the version, and whether
         she is thinking here or somewhere else. Two facts that change, in the corner,
         faint enough to be furniture — rather than five that mostly do not, in a panel
         across the room she is standing in. -->
    <span id="stamp" aria-hidden="true"></span>
  </div>

  <div id="console">
    <div class="row">
      <button id="talk" title="Listen (Ctrl+Alt+O)" aria-label="Toggle listening">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round">
          <rect x="9" y="2.5" width="6" height="11" rx="3"/><path d="M5 11a7 7 0 0 0 14 0"/><path d="M12 18v3.5"/>
        </svg>
      </button>

      <button id="watch" title="Let her see you" aria-label="Let her see you" aria-pressed="false" hidden>
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
          <path d="M2 12s3.6-6.2 10-6.2S22 12 22 12s-3.6 6.2-10 6.2S2 12 2 12z"/><circle cx="12" cy="12" r="2.6"/>
        </svg>
      </button>

      <!-- Typing is the exception, so it costs a click rather than the width of the
           window. Most turns are spoken; a field sized for the full width of a
           full-screen console was announcing the opposite. -->
      <button id="typeBtn" title="Type instead" aria-label="Type instead" aria-expanded="false">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
          <rect x="2.5" y="6" width="19" height="12" rx="2"/><path d="M6 9.5h.01M9.5 9.5h.01M13 9.5h.01M16.5 9.5h.01M8 14h8"/>
        </svg>
      </button>

      <!-- Hush lives out here now: it has to be reachable while she is speaking, and
           she can be speaking with the field shut. -->
      <button id="hush" title="Stop her (Esc)" aria-label="Stop her mid-sentence" hidden>
        <svg viewBox="0 0 24 24" fill="currentColor"><rect x="6" y="6" width="12" height="12" rx="1.5"/></svg>
      </button>

      <div class="field" hidden>
        <input id="text" type="text" autocomplete="off" placeholder="Type instead...">
        <button id="send" aria-label="Send message">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round">
            <path d="M4 12h15M13 6l6 6-6 6"/>
          </svg>
        </button>
      </div>

      <span class="grow"></span>
    </div>
  </div>
</div>

<aside id="drawer" aria-hidden="true">
  <div class="tabs" role="tablist">
    <button class="tab on" data-tab="log" role="tab" aria-selected="true">Transcript</button>
    <button class="tab" data-tab="settings" role="tab" aria-selected="false">Settings</button>
    <button class="tab" data-tab="health" role="tab" aria-selected="false">Health</button>
    <button class="tab" data-tab="dev" role="tab" aria-selected="false" hidden>Dev</button>
    <span class="spacer"></span>
    <button id="drawerClose" aria-label="Close the drawer">Close</button>
  </div>

  <div class="dbody" data-body="log">
    <div id="entries"><div class="empty">Nothing said yet.</div></div>
    <div class="dfoot"><button id="forget" class="ghost">Forget the conversation</button></div>
  </div>

  <div class="dbody" data-body="settings" hidden>
    <div class="dscroll">
      <!-- Whatever the thing showing this page says it owns: the handset's address, room and
           camera; a desktop client's hotkey. Empty and hidden in a browser tab, which owns
           nothing. The page never asks *which* client it is — it renders what it is handed,
           which is the whole point of the embedder seam. -->
      <div id="deviceBox" hidden></div>

      <label class="field-row" for="avatar">
        <span class="label">Appearance</span>
        <select id="avatar"></select>
        <span class="hint">Drop a <code>.vrm</code> in her avatars folder to add one.</span>
      </label>

      <!-- Two rows used to live here: an engine to pick, and a voice within it. Stage 16
           auditioned twenty-two voices and one was chosen, so both became menus over a list
           of one. The Voice pill in the status strip still names her, which is all that is
           left to say - and the same reasoning that hid the camera row in v0.39.2. -->

      <label class="field-row" for="microphone" data-host-only>
        <span class="label">Microphone</span>
        <select id="microphone"></select>
        <span class="hint">Which device she listens through. Changing this reopens her ears.</span>
      </label>

      <!-- The camera is the only sense that is off by default, and until now it was also
           the only one you could not reach without a text editor: the setting existed, the
           protocol carried it, and nothing could set it. -->
      <label class="field-row check" for="camera">
        <span class="label">Let her see you</span>
        <input id="camera" type="checkbox">
        <span class="hint">Off by default &mdash; the only sense that is. She opens it for a
          single frame when a question genuinely needs eyes, shows an unmistakable marker
          while it is live, and keeps nothing.</span>
      </label>

      <label class="field-row" for="cameraDevice">
        <span class="label">Camera</span>
        <select id="cameraDevice"></select>
        <span class="hint" id="cameraHint">Which one she looks through.</span>
      </label>

      <label class="field-row check" for="music" data-host-only>
        <span class="label">Hears what you play</span>
        <input id="music" type="checkbox">
        <span class="hint" id="musicHint">Nothing is recorded: what survives is a tempo and a loudness.</span>
      </label>

      <label class="field-row" for="output" data-host-only>
        <span class="label">Output she listens to</span>
        <select id="output"></select>
        <span class="hint">Pick the real sound card. A virtual endpoint &mdash; streaming software, remote audio &mdash; flattens the sound to full scale and leaves no beat to find.</span>
      </label>

      <label class="field-row check" for="stats">
        <span class="label">Show the status readout</span>
        <input id="stats" type="checkbox">
        <span class="hint">The voice, ears, brain, music and profile panel over her top-left corner. Off is the setting for actually looking at her.</span>
      </label>

      <label class="field-row" for="whisperCompute" data-host-only>
        <span class="label">Speech recognition runs on</span>
        <select id="whisperCompute">
          <option value="auto">Whichever Whisper picks</option>
          <option value="cpu">The processor</option>
          <option value="gpu">The graphics card</option>
        </select>
        <span class="hint">&ldquo;Whichever&rdquo; means the graphics card wherever one will load, which is slower than a good processor on a weak card. Takes effect when she restarts.</span>
      </label>

      <label class="field-row" for="roomHour">
        <span class="label">Room lighting</span>
        <select id="roomHour">
          <option value="-1">Follow the clock</option>
          <option value="6">Dawn</option>
          <option value="9">Morning</option>
          <option value="13">Midday</option>
          <option value="18">Evening</option>
          <option value="21">Dusk</option>
          <option value="0">Night</option>
        </select>
        <span class="hint">The wall, the lights and this window move together through the day.</span>
      </label>

      <!-- Entered once a machine and read never: a setting, not a status. -->
      <label class="field-row" for="key" id="keyrow">
        <span class="label">API key</span>
        <input id="key" type="password" placeholder="sk-ant-..." autocomplete="off">
        <span class="hint" id="keyHint">Sealed to this Windows account with DPAPI. It never returns to this page, and it is not needed while she is on a local brain.</span>
        <button id="saveKey" class="ghost">Store the key</button>
      </label>

      <button id="openData" class="ghost" data-host-only>Open her folder</button>
    </div>
  </div>

  <div class="dbody" data-body="health" hidden>
    <div id="diagBody"><div class="empty">Run the self-test to see how she is doing.</div></div>
    <div class="dfoot">
      <button id="diagRun" class="ghost">Test</button>
      <button id="diagSave" class="ghost" data-host-only>Save a file</button>
      <p id="diagFoot">The saved file contains her log, which holds what she heard and said. Read it before sending it on.</p>
    </div>
  </div>

  <div class="dbody" data-body="dev" hidden>
    <div id="devBody"></div>
    <p class="dfoot" id="devFoot">Everything here drives the renderer directly, except the senses, which reach the host.</p>
  </div>
</aside>

<!-- Whether this face can still reach her. Permanent while it cannot, because the notice
     below fades and a disconnected face would then look like a working one. -->
<div id="link" role="status" aria-live="polite"></div>

<div id="notice"></div>

<!-- Modules, and in this order: they execute in document order, so face.js has set
     window.Face before bridge.js reports whether the scene built. -->
<script type="module" src="face.js"></script>
<script type="module" src="bridge.js"></script>
</body>
</html>
```
