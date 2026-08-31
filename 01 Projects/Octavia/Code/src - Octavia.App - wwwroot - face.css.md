---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\wwwroot\face.css
---

# src\Octavia.App\wwwroot\face.css

```css
/* ── tokens ────────────────────────────────────────────────
   Stage 10's first move. Every colour, size and rhythm in this file comes from here,
   so a change is made once. The room's palette reaches this set at runtime: face.js
   writes --room-tint and --room-ink from the same day-cycle keyframes that light her,
   which is what stops the window floating in fixed grey above a room that changes. */
:root{
  /* her plaster, unchanged since the first bust */
  --wall-hi:#D4D0C4;
  --wall-lo:#A29D8F;

  --ink:#242219;
  --ink-soft:#6B675B;
  --ink-faint:#8D897C;

  --cobalt:#2F4CD0;
  --cobalt-soft:rgba(47,76,208,.14);

  /* semantic, and deliberately separate from the accent */
  --ok:#2E7D5B;
  --warn:#B8791F;
  --alert:#C4362F;

  --rule:rgba(36,34,25,.16);
  --panel:rgba(255,253,247,.62);
  --sunk:rgba(255,253,247,.72);

  /* Set by face.js each time the day is applied; these are the midday values so the
     page is correct for the frame before the first update lands. */
  --room-tint:rgba(255,252,242,.30);
  --room-ink:#242219;
  --room-line:rgba(36,34,25,.16);

  /* type */
  --display:"Segoe UI Variable Display","Segoe UI",system-ui,sans-serif;
  --body:"Segoe UI Variable Text","Segoe UI",system-ui,sans-serif;
  --mono:"Cascadia Mono",Consolas,ui-monospace,monospace;

  --t-label:9.5px;      /* uppercase mono labels */
  --t-meta:10.5px;      /* status pills, tabs */
  --t-body:14px;
  --t-input:15px;
  --t-mark:19px;
  /* The distance size. 10-foot-interface guidance puts the floor near 28px at 1080p;
     the old ceiling was 25px, which is why she could not be read from the sofa. */
  --t-caption:clamp(21px,3.1vw,34px);
  --t-caption-quiet:clamp(15px,1.9vw,19px);

  --track:.2em;         /* letter-spacing for uppercase labels */

  /* rhythm */
  --s1:6px; --s2:10px; --s3:14px; --s4:20px; --s5:26px; --s6:36px;
  --radius:2px;
  --radius-pill:100px;
  --ease:cubic-bezier(.32,.72,0,1);
}

*{box-sizing:border-box}
html,body{height:100%}
body{
  margin:0;
  font-family:var(--body);
  color:var(--ink);
  background:
    radial-gradient(120% 80% at 50% 8%, #E0DCD1 0%, transparent 60%),
    linear-gradient(178deg, var(--wall-hi) 0%, var(--wall-hi) 46%, var(--wall-lo) 100%);
  overflow:hidden;
  user-select:none;
  -webkit-font-smoothing:antialiased;
}
body::after{
  content:"";position:fixed;inset:0;pointer-events:none;z-index:5;opacity:.5;
  background-image:url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='140' height='140'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='3'/><feColorMatrix values='0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .06 0'/></filter><rect width='140' height='140' filter='url(%23n)'/></svg>");
}
@media (prefers-reduced-motion: reduce){
  *{transition-duration:.01ms!important;animation-duration:.01ms!important}
}

#app{position:fixed;inset:0;display:flex;flex-direction:column;z-index:2}

/* ── header ─────────────────────────────────────────────── */
header{
  display:flex;align-items:center;gap:var(--s3);padding:var(--s4) var(--s5);flex:0 0 auto;
  background:var(--room-tint);color:var(--room-ink);
  transition:background 1s linear,color 1s linear;
}
.mark{font-family:var(--display);font-weight:700;font-size:var(--t-mark);letter-spacing:-.02em}
.mark em{font-style:normal;color:var(--cobalt)}
/* The "In residence" eyebrow is gone. It was a label nobody could read a meaning off —
   it meant "she is here rather than in the cloud", which is true, unremarkable, and not
   worth a permanent line in the room. */
.spacer{flex:1}

#watching{
  font-family:var(--mono);font-size:10px;letter-spacing:var(--track);text-transform:uppercase;
  color:#fff;background:var(--alert);border-radius:var(--radius-pill);padding:5px 12px;white-space:nowrap;
}

#state{
  font-family:var(--mono);font-size:var(--t-meta);letter-spacing:.16em;text-transform:uppercase;
  display:flex;align-items:center;gap:8px;
  border:1px solid var(--room-line);border-radius:var(--radius-pill);padding:5px 12px 5px 10px;
  background:var(--panel);white-space:nowrap;color:var(--ink-soft);
  transition:border-color 1s linear,color .25s;
}
#state .dot{width:6px;height:6px;border-radius:50%;background:var(--ink-faint);transition:background .25s}
body[data-state="listening"] #state .dot{background:var(--cobalt);animation:pulse 1.1s infinite}
body[data-state="thinking"] #state .dot{background:var(--warn);animation:pulse .7s infinite}
body[data-state="speaking"] #state .dot{background:var(--ok)}
/* Failure is a designed state, not a missing one. */
body.trouble #state{border-color:var(--alert);color:var(--alert)}
body.trouble #state .dot{background:var(--alert)}
@keyframes pulse{50%{opacity:.25}}

#stage{flex:1 1 auto;position:relative;min-height:0}
/* `inset:0` is not enough on its own. A canvas is a *replaced* element, so with width and
   height auto it lays out at its intrinsic size — the drawing buffer — and `inset` cannot
   stretch it. The renderer is called as `setSize(w, h, false)`, which deliberately leaves
   the style alone, so on any display with `devicePixelRatio > 1` the buffer is 2x the
   stage and the canvas overflows it by 2x, anchored top left. She then renders enormous
   and off to the bottom right, because the middle of the picture is no longer the middle
   of the box.
   Invisible on this machine — WebView2 runs at dpr 1 here — and waiting for the first
   4K monitor or a laptop with display scaling. */
#scene{position:absolute;inset:0;display:block;width:100%;height:100%}

/* ── placard ────────────────────────────────────────────── */
/* Overlaid on the room rather than stacked under it.
   Collapsing it used to change the stage's height, which fires the ResizeObserver, which
   re-frames the camera — so she rescaled every time the caption came or went, and the
   transition made that a visible lurch rather than a move. Nothing about the stage
   changes now; only the text fades. The scrim keeps it legible over whatever she happens
   to be standing in front of. */
#placard{
  position:absolute;left:0;right:0;bottom:0;z-index:2;
  padding:var(--s6) var(--s5) var(--s4);text-align:center;
  display:flex;flex-direction:column;justify-content:flex-end;align-items:center;
  pointer-events:none;
  background:linear-gradient(to top, var(--room-tint) 35%, transparent);
  transition:opacity .45s ease;
}

/* Nine seconds after the last word, the room has itself back. */
body.quiet #placard{opacity:0}
#caption{
  font-family:var(--display);font-weight:400;font-size:var(--t-caption);
  line-height:1.25;letter-spacing:-.015em;max-width:38ch;margin:0;
  color:var(--room-ink);transition:color 1s linear,opacity .3s;text-wrap:balance;
}
#caption.muted{opacity:.55;font-size:var(--t-caption-quiet)}
#caption.tentative{opacity:.72;font-style:italic}
/* Overheard: present, legible, clearly not conversation. */
#caption.overheard{opacity:.5;font-style:italic;font-size:var(--t-caption-quiet)}
#caption.err{color:var(--alert)}
#speaker{
  font-family:var(--mono);font-size:10px;letter-spacing:var(--track);text-transform:uppercase;
  color:var(--room-ink);opacity:.45;margin-bottom:8px;transition:color 1s linear;
}

/* ── console ────────────────────────────────────────────── */
#console{
  flex:0 0 auto;border-top:1px solid var(--room-line);
  background:var(--room-tint);
  backdrop-filter:blur(8px);
  padding:var(--s3) var(--s5) var(--s2);
  transition:background 1s linear,border-color 1s linear;
}
/* Full width rather than a centred 860px column. The controls belong at the left edge
   and the readout at the right one; a centred column pulled both into the middle and
   left the corners empty. */
.row{display:flex;gap:var(--s3);align-items:center}

/* One primary action. Everything else steps down a full size. */
#talk{
  flex:0 0 auto;width:58px;height:58px;border-radius:50%;
  border:1.5px solid var(--room-ink);background:transparent;color:var(--room-ink);
  cursor:pointer;display:grid;place-items:center;position:relative;
  transition:background .18s,color .18s,border-color .18s,transform .12s;
}
#talk:hover{background:rgba(36,34,25,.06)}
#talk:active{transform:scale(.96)}
#talk svg{width:21px;height:21px}
body[data-state="listening"] #talk{background:var(--cobalt);border-color:var(--cobalt);color:#fff}
#talk::after{content:"";position:absolute;inset:-7px;border-radius:50%;border:1px solid var(--cobalt);opacity:0;transform:scale(.9)}
body[data-state="listening"] #talk::after{animation:halo 1.4s infinite}
@keyframes halo{0%{opacity:.6;transform:scale(.92)}100%{opacity:0;transform:scale(1.28)}}

/* The senses sit together: ear and eye, one cluster, the same contract. */
#watch{
  flex:0 0 auto;width:42px;height:42px;border-radius:50%;
  border:1px solid var(--room-line);background:transparent;color:var(--room-ink);opacity:.75;
  cursor:pointer;display:grid;place-items:center;
  transition:background .18s,color .18s,border-color .18s,opacity .18s,transform .12s;
}
#watch:hover{opacity:1}
#watch:active{transform:scale(.96)}
#watch svg{width:17px;height:17px}
#watch[aria-pressed="true"]{background:var(--alert);border-color:var(--alert);color:#fff;opacity:1}

/* The field is opened by #typeBtn and takes the room it needs only then. `.grow`
   holds the space open when it is shut, so the drawer button stays where it was and
   the row does not jump as the field appears. */
.grow{flex:1 1 auto}
.field[hidden]{display:none}
.field{flex:1 1 auto;position:relative;display:flex;align-items:center;max-width:560px}

#typeBtn,#hush{
  flex:0 0 auto;width:42px;height:42px;border-radius:50%;
  border:1px solid var(--room-line);background:transparent;color:var(--room-ink);opacity:.75;
  cursor:pointer;display:grid;place-items:center;
  transition:background .18s,color .18s,border-color .18s,opacity .18s,transform .12s;
}
/* `display:grid` above beats the `hidden` attribute's own `display:none`, which is how
   Hush ended up permanently visible the moment it moved out of the field. */
#typeBtn[hidden],#hush[hidden],#watch[hidden]{display:none}
#typeBtn:hover,#hush:hover{opacity:1}
#typeBtn:active,#hush:active{transform:scale(.96)}
#typeBtn svg{width:18px;height:18px}
#typeBtn[aria-expanded="true"]{border-color:var(--cobalt);color:var(--cobalt);opacity:1}
#text{
  width:100%;font-family:var(--body);font-size:var(--t-input);color:var(--ink);
  background:var(--sunk);border:1px solid var(--rule);border-radius:var(--radius);
  padding:15px 58px 15px 15px;outline:none;transition:border-color .18s,box-shadow .18s,padding .18s;
  user-select:text;
}
body.busy #text{padding-right:98px}
#text::placeholder{color:var(--ink-faint)}
#text:focus{border-color:var(--cobalt);box-shadow:0 0 0 3px var(--cobalt-soft)}
#send{
  position:absolute;right:6px;width:40px;height:38px;border:0;background:transparent;
  color:var(--ink-soft);cursor:pointer;border-radius:var(--radius);display:grid;place-items:center;
}
#send:hover{color:var(--cobalt);background:var(--cobalt-soft)}
#send svg{width:17px;height:17px}
#hush{border-color:var(--alert);color:var(--alert);opacity:1}
#hush:hover{background:rgba(196,54,47,.12)}
#hush svg{width:15px;height:15px}

/* In the header now, beside the state. Smaller than it was in the console, because up
   here it is a way out rather than a control you reach for mid-sentence. */
#drawerBtn{
  flex:0 0 auto;width:32px;height:32px;border-radius:var(--radius);margin-right:var(--s3);
  border:1px solid var(--room-line);background:transparent;color:var(--room-ink);opacity:.75;
  cursor:pointer;display:grid;place-items:center;transition:opacity .18s,border-color .18s;
}
#drawerBtn:hover{opacity:1}
#drawerBtn[aria-expanded="true"]{border-color:var(--cobalt);color:var(--cobalt);opacity:1}
#drawerBtn svg{width:15px;height:15px}

/* ── status: a readout, with health ─────────────────────── */
/* Stacked at the left rather than spread across the window. Five readings on one line
   read as a sentence you have to parse; five short lines read as a list you can scan,
   and from a sofa scanning is all that happens. */
/* Stacked and right-aligned, sharing the control row rather than sitting under it. The
   left of the bar is where you act; the right is where you look. */
/* Floated over her top-left corner on translucent glass, rather than filling a strip of
   chrome. It is reference material — glanced at, not read — so it should sit *in* the
   room quietly instead of taking a band of the window away from her.
   `pointer-events:none` because it is a readout: nothing here is clickable, and a panel
   that swallows clicks over the scene is a trap. */
.meta{
  position:absolute;top:var(--s4);left:var(--s4);z-index:3;
  display:flex;flex-direction:column;align-items:flex-start;gap:5px;
  padding:var(--s2) var(--s3);border-radius:var(--radius);
  background:rgba(255,253,247,.34);
  border:1px solid var(--room-line);
  backdrop-filter:blur(10px);
  pointer-events:none;
  transition:opacity .3s,background 1s linear,border-color 1s linear;
}
/* Off entirely, for when you just want to look at her. */
body.no-stats .meta{opacity:0;visibility:hidden}
.pill{
  display:flex;gap:7px;align-items:center;
  font-family:var(--mono);font-size:var(--t-meta);letter-spacing:.1em;text-transform:uppercase;
  color:var(--room-ink);opacity:.58;transition:color 1s linear,opacity .2s;
}
.pill b{font-weight:400;text-transform:none;letter-spacing:.03em;opacity:.92}
.pill .d{width:6px;height:6px;border-radius:50%;background:var(--ok);flex:0 0 auto;transition:background .3s}
.pill.warn .d{background:var(--warn)}
.pill.dead{opacity:.4}
.pill.dead .d{background:var(--ink-faint)}
.pill.bad .d{background:var(--alert)}
/* The missing-key pill lived here. It has gone to Settings — a permanent amber warning
   across the bottom of the room is exactly the kind of thing you stop seeing, and it was
   nagging the person looking at her rather than the person who can fix it. */

/* ── splash ─────────────────────────────────────────────── */
#splash{
  position:fixed;inset:0;z-index:40;display:grid;place-items:center;
  background:linear-gradient(160deg,var(--wall-hi),var(--wall-lo));
  transition:opacity .5s ease,visibility .5s;
}
body:not(.loading) #splash{opacity:0;visibility:hidden;pointer-events:none}

.splash-inner{display:flex;flex-direction:column;align-items:center;gap:var(--s4);text-align:center}
.splash-inner .mark{font-size:30px}

#splashSteps{list-style:none;display:flex;flex-direction:column;gap:9px;align-items:flex-start}
#splashSteps li{
  display:flex;align-items:center;gap:10px;
  font-family:var(--mono);font-size:var(--t-meta);letter-spacing:.1em;text-transform:uppercase;
  color:var(--ink);opacity:.4;transition:opacity .3s;
}
#splashSteps li::before{
  content:"";width:7px;height:7px;border-radius:50%;
  border:1px solid var(--ink-faint);background:transparent;transition:background .3s,border-color .3s;
}
/* The step being waited on pulses; the ones behind it are simply filled. Anonymous
   spinners are what this screen exists to replace. */
#splashSteps li.now{opacity:.95}
#splashSteps li.now::before{background:var(--cobalt);border-color:var(--cobalt);animation:pulse 1.1s ease-in-out infinite}
#splashSteps li.done{opacity:.62}
#splashSteps li.done::before{background:var(--ok);border-color:var(--ok)}
@keyframes pulse{0%,100%{transform:scale(.8);opacity:.55}50%{transform:scale(1.15);opacity:1}}

#splashNote{
  font-family:var(--body);font-size:12.5px;color:var(--ink-soft);
  max-width:340px;line-height:1.5;min-height:1.2em;
}

:focus-visible{outline:2px solid var(--cobalt);outline-offset:2px}

/* ── the drawer: one component, four tabs ───────────────── */
#drawer{
  position:fixed;top:0;right:0;bottom:0;width:min(420px,90vw);z-index:6;
  background:rgba(250,248,242,.96);backdrop-filter:blur(14px);
  border-left:1px solid var(--rule);transform:translateX(100%);
  transition:transform .34s var(--ease);
  display:flex;flex-direction:column;
}
#drawer.on{transform:none}

.tabs{display:flex;align-items:stretch;border-bottom:1px solid var(--rule);padding:0 var(--s2);gap:2px}
.tab,#drawerClose{
  font-family:var(--mono);font-size:var(--t-meta);letter-spacing:.14em;text-transform:uppercase;
  background:none;border:0;color:var(--ink-faint);cursor:pointer;
  padding:var(--s4) var(--s2) var(--s3);border-bottom:2px solid transparent;
  transition:color .18s,border-color .18s;
}
.tab:hover{color:var(--ink-soft)}
.tab.on{color:var(--cobalt);border-bottom-color:var(--cobalt)}
#drawerClose{letter-spacing:var(--track);color:var(--ink-soft)}
#drawerClose:hover{color:var(--cobalt)}
.tabs .spacer{flex:1}

.dbody{flex:1;min-height:0;display:flex;flex-direction:column}
.dbody[hidden]{display:none}
.dscroll{overflow-y:auto;padding:var(--s4) 22px var(--s5);display:flex;flex-direction:column;gap:22px;flex:1}
.dfoot{
  flex:0 0 auto;border-top:1px solid var(--rule);padding:var(--s3) 22px var(--s4);
  display:flex;gap:var(--s2);align-items:center;flex-wrap:wrap;
}
.dfoot p{margin:0}

/* transcript */
#entries{overflow-y:auto;padding:var(--s1) 22px var(--s4);flex:1;user-select:text}
.turn{padding:var(--s3) 0;border-bottom:1px solid var(--rule)}
.turn .who{font-family:var(--mono);font-size:var(--t-label);letter-spacing:var(--track);text-transform:uppercase;color:var(--ink-faint);display:block;margin-bottom:6px}
.turn.me .who{color:var(--cobalt)}
.turn p{margin:0;font-size:var(--t-body);line-height:1.5;color:var(--ink)}
.turn.overheard{opacity:.55}
.turn.overheard .who{text-transform:none;letter-spacing:.04em;font-style:italic}
.turn.overheard p{font-style:italic}
.empty{font-family:var(--mono);font-size:11px;color:var(--ink-faint);line-height:1.7;padding:var(--s4) 22px;letter-spacing:.04em}
#entries .empty{padding-left:0;padding-right:0}

/* settings */
.field-row{display:flex;flex-direction:column;gap:7px}
.field-row .label{
  font-family:var(--mono);font-size:var(--t-label);letter-spacing:var(--track);text-transform:uppercase;color:var(--ink-faint);
}
.field-row select,.field-row input[type=password]{
  font-family:var(--body);font-size:var(--t-body);color:var(--ink);
  background:var(--sunk);border:1px solid var(--rule);border-radius:var(--radius);
  padding:9px 10px;outline:none;width:100%;transition:border-color .18s,box-shadow .18s;
}
.field-row input[type=password]{user-select:text;font-family:var(--mono);font-size:12.5px}
.field-row select:focus,.field-row input:focus{border-color:var(--cobalt);box-shadow:0 0 0 3px var(--cobalt-soft)}
.field-row .hint{font-size:12px;line-height:1.5;color:var(--ink-faint)}
.field-row code{font-family:var(--mono);font-size:11px}
.field-row.check{display:grid;grid-template-columns:1fr auto;align-items:center;gap:7px 12px}
.field-row.check .hint{grid-column:1/-1}
.field-row.check input{
  appearance:none;width:38px;height:21px;border-radius:11px;cursor:pointer;
  background:rgba(120,116,108,.28);border:1px solid var(--rule);
  position:relative;transition:background .18s;outline:none;
}
.field-row.check input::after{
  content:"";position:absolute;top:2px;left:2px;width:15px;height:15px;border-radius:50%;
  background:#FFFDF7;box-shadow:0 1px 2px rgba(0,0,0,.28);transition:transform .18s;
}
.field-row.check input:checked{background:var(--cobalt)}
.field-row.check input:checked::after{transform:translateX(17px)}
/* Louder now that this is the *only* place the missing key is mentioned. It has to be
   findable by someone who came to Settings for something else. */
#keyrow.wanted .label{color:var(--warn)}
#keyrow.wanted .label::after{content:" — needed";font-weight:600}
#keyrow.wanted input[type=password]{border-color:var(--warn)}
#keyrow.wanted{
  border-left:2px solid var(--warn);
  padding-left:var(--s3);margin-left:calc(var(--s3) * -1 - 2px);
}

.ghost{
  font-family:var(--mono);font-size:var(--t-meta);letter-spacing:.12em;text-transform:uppercase;
  color:var(--ink-soft);background:transparent;border:1px solid var(--rule);
  border-radius:var(--radius);padding:10px 12px;cursor:pointer;white-space:nowrap;
  transition:border-color .18s,color .18s;
}
.ghost:hover{border-color:var(--ink-soft);color:var(--ink)}
.dscroll .ghost{align-self:flex-start}

/* health */
#diagBody{overflow-y:auto;padding:var(--s1) 22px var(--s2);flex:1;user-select:text}
#diagFoot{
  flex:1 1 100%;font-family:var(--mono);font-size:10px;line-height:1.65;color:var(--ink-faint);letter-spacing:.03em;
}
.check{display:flex;gap:11px;padding:12px 0;border-bottom:1px solid var(--rule);align-items:flex-start}
.check .mark{
  flex:0 0 auto;width:7px;height:7px;border-radius:50%;margin-top:6px;
  background:var(--ok);font-size:0;letter-spacing:0;
}
.check.bad .mark{background:var(--alert)}
.check .body{flex:1}
.check .name{font-family:var(--mono);font-size:var(--t-label);letter-spacing:var(--track);text-transform:uppercase;color:var(--ink-faint);display:block;margin-bottom:4px}
.check .detail{margin:0;font-size:13.5px;line-height:1.5;color:var(--ink)}
.check .fix{margin:5px 0 0;font-size:12.5px;line-height:1.55;color:var(--ink-soft);border-left:2px solid var(--rule);padding-left:9px}
#diagBody h3{
  font-family:var(--mono);font-size:var(--t-label);letter-spacing:var(--track);text-transform:uppercase;
  color:var(--ink-faint);margin:var(--s4) 0 var(--s1);font-weight:400;
}
table.facts{width:100%;border-collapse:collapse;font-size:12.5px}
table.facts th{
  text-align:left;font-family:var(--mono);font-size:10px;letter-spacing:.1em;text-transform:uppercase;
  color:var(--ink-faint);font-weight:400;padding:4px 12px 4px 0;vertical-align:top;white-space:nowrap;
}
table.facts td{padding:4px 0;color:var(--ink);vertical-align:top}
#recent{
  font-family:var(--mono);font-size:10.5px;line-height:1.6;color:var(--ink-soft);
  background:rgba(36,34,25,.05);border-radius:var(--radius);padding:10px;
  white-space:pre-wrap;word-break:break-word;margin:0;max-height:220px;overflow-y:auto;
}

/* dev */
#devBody{overflow-y:auto;padding:var(--s3) 22px var(--s4);flex:1;display:flex;flex-direction:column;gap:var(--s3)}
#devFoot{
  flex:0 0 auto;border-top:1px solid var(--rule);margin:0;padding:var(--s3) 22px var(--s4);
  font-family:var(--mono);font-size:10px;line-height:1.65;color:var(--ink-faint);letter-spacing:.03em;
}
.devgroup{display:flex;flex-direction:column;gap:7px}
.devlabel{font-family:var(--mono);font-size:var(--t-label);letter-spacing:var(--track);text-transform:uppercase;color:var(--ink-faint)}
.devrow{display:flex;flex-wrap:wrap;gap:6px;align-items:center}
.devbtn{
  font-family:var(--mono);font-size:var(--t-meta);letter-spacing:.08em;
  color:var(--ink-soft);background:var(--sunk);
  border:1px solid var(--rule);border-radius:var(--radius);padding:6px 9px;cursor:pointer;
  transition:border-color .16s,color .16s,background .16s;
}
.devbtn:hover{border-color:var(--ink-soft);color:var(--ink)}
.devbtn.on{border-color:var(--cobalt);color:var(--cobalt);background:var(--cobalt-soft)}
.devslider{display:flex;align-items:center;gap:8px;flex:1 1 190px;font-family:var(--mono);font-size:10px;color:var(--ink-faint)}
.devslider input{flex:1;accent-color:var(--cobalt);min-width:80px}
.devvalue{color:var(--ink-soft);min-width:30px;text-align:right;font-variant-numeric:tabular-nums}
.devnote{margin:0;font-family:var(--mono);font-size:10px;line-height:1.6;color:var(--ink-faint);letter-spacing:.03em}

/* ── camera-live marker ─────────────────────────────────── */
body.looking::before{
  content:"camera on";
  position:fixed;top:0;left:0;right:0;z-index:9;
  font-family:var(--mono);font-size:10px;letter-spacing:.22em;text-transform:uppercase;
  text-align:center;padding:5px;color:#FFF;background:var(--alert);
}

/* ── notice ─────────────────────────────────────────────── */
#notice{
  position:fixed;left:50%;bottom:130px;transform:translate(-50%,10px);z-index:8;
  background:rgba(36,34,25,.92);color:#F6F3EA;font-size:13.5px;line-height:1.45;
  padding:11px 18px;border-radius:3px;max-width:min(60ch,80vw);text-align:center;
  opacity:0;pointer-events:none;transition:opacity .3s,transform .3s;
}
#notice.on{opacity:1;transform:translate(-50%,0)}

@media (max-width:760px){
  .meta{gap:var(--s3)}
  .pill:nth-child(n+4){display:none}
  #drawer{width:100vw}
}
```
