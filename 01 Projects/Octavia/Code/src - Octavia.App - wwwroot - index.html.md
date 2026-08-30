---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\index.html
---

# src\Octavia.App\wwwroot\index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta http-equiv="Content-Security-Policy"
      content="default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data: https://octavia.avatar; media-src 'self' blob:; connect-src 'self' ws://127.0.0.1:* ws://localhost:* https://octavia.avatar">
<title>Octavia</title>
<link rel="stylesheet" href="face.css">
</head>
<body data-state="idle">

<div id="app">
  <header>
    <span class="mark">Octavia<em>.</em></span>
    <span class="eyebrow">In residence</span>
    <span class="spacer"></span>
    <span id="watching" hidden>camera</span>
    <span id="state"><span class="dot"></span><span id="stateLabel">Idle</span></span>
  </header>

  <div id="stage"><canvas id="scene"></canvas></div>

  <div id="placard">
    <span id="speaker">&nbsp;</span>
    <p id="caption" class="muted">Press the microphone, or type below.</p>
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

      <div class="field">
        <input id="text" type="text" autocomplete="off" placeholder="Type instead...">
        <!-- Shown only while there is something to stop. Esc does the same thing. -->
        <button id="hush" title="Stop her (Esc)" aria-label="Stop her mid-sentence" hidden>
          <svg viewBox="0 0 24 24" fill="currentColor"><rect x="6" y="6" width="12" height="12" rx="1.5"/></svg>
        </button>
        <button id="send" aria-label="Send message">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round">
            <path d="M4 12h15M13 6l6 6-6 6"/>
          </svg>
        </button>
      </div>

      <button id="drawerBtn" title="Transcript, settings and health" aria-label="Open the drawer" aria-expanded="false">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round">
          <path d="M4 7h16M4 12h16M4 17h16"/>
        </svg>
      </button>
    </div>

    <!-- A readout, not a control surface: every button that used to live here has gone
         to the drawer, and each row now says whether it is well as well as what it is. -->
    <div class="meta">
      <span class="pill" id="pillVoice"><span class="d"></span>Voice <b id="voiceLabel">&mdash;</b></span>
      <span class="pill" id="pillEars"><span class="d"></span>Ears <b id="ears">not started</b></span>
      <span class="pill" id="pillBrain"><span class="d"></span>Brain <b id="model">&mdash;</b></span>
      <span class="pill" id="pillMusic"><span class="d"></span>Music <b id="musicLabel">&mdash;</b></span>
      <span class="pill" id="pillProfile"><span class="d"></span>Profile <b id="profile">&mdash;</b></span>
      <button class="pill act" id="pillKey" hidden><span class="d"></span>Needs an API key &mdash; open settings</button>
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
      <label class="field-row" for="avatar">
        <span class="label">Appearance</span>
        <select id="avatar"></select>
        <span class="hint">Drop a <code>.vrm</code> in her avatars folder to add one.</span>
      </label>

      <label class="field-row" for="voiceEngine">
        <span class="label">Speech</span>
        <select id="voiceEngine">
          <option value="windows">Windows voices</option>
          <option value="neural">Neural voice</option>
        </select>
        <span class="hint">The neural voice sounds far better and downloads about 80&nbsp;MB the first time.</span>
      </label>

      <label class="field-row" for="voice">
        <span class="label">Voice</span>
        <select id="voice"></select>
        <span class="hint" id="voiceHint">Windows speech voices installed on this machine.</span>
      </label>

      <label class="field-row check" for="music">
        <span class="label">Hears what you play</span>
        <input id="music" type="checkbox">
        <span class="hint" id="musicHint">Nothing is recorded: what survives is a tempo and a loudness.</span>
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
        <span class="hint">Sealed to this Windows account with DPAPI. It never returns to this page, and it is not needed while she is on a local brain.</span>
        <button id="saveKey" class="ghost">Store the key</button>
      </label>

      <button id="openData" class="ghost">Open her folder</button>
    </div>
  </div>

  <div class="dbody" data-body="health" hidden>
    <div id="diagBody"><div class="empty">Run the self-test to see how she is doing.</div></div>
    <div class="dfoot">
      <button id="diagRun" class="ghost">Test</button>
      <button id="diagSave" class="ghost">Save a file</button>
      <p id="diagFoot">The saved file contains her log, which holds what she heard and said. Read it before sending it on.</p>
    </div>
  </div>

  <div class="dbody" data-body="dev" hidden>
    <div id="devBody"></div>
    <p class="dfoot" id="devFoot">Everything here drives the renderer directly, except the senses, which reach the host.</p>
  </div>
</aside>

<div id="notice"></div>

<!-- Modules, and in this order: they execute in document order, so face.js has set
     window.Face before bridge.js reports whether the scene built. -->
<script type="module" src="face.js"></script>
<script type="module" src="bridge.js"></script>
</body>
</html>
```
