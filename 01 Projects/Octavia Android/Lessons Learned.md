---
project: Octavia Android
tags: [octavia, octavia-android, lessons]
---

# Lessons Learned

> The expensive ones, so we never pay twice. Her equivalent is [[Lessons Learned|Octavia's]].

## A roadmap step can be recorded as done and be half done

Her Stage 13 step 1 — "a transport that can leave the machine" — was written up as complete in v0.14.0. The socket really did bind the LAN. But **`wwwroot` was never served by anything**: it reaches the built-in face through a WebView2 virtual host mapping, which is a feature of that control and not a server. Nothing in the host had ever answered a GET.

A phone could therefore open the socket and have no page to run inside it, and nobody noticed for a fortnight because every existing face either *was* the WebView2 page or ran on a machine that had the same trick.

**The lesson:** "the transport works" and "the client can actually start" are different claims. A step is done when something that was not there before can now do the thing end to end — not when the piece you were thinking about is finished.

## The alternative that looks symmetrical usually isn't

Vendoring `wwwroot` into the APK looked like a fair trade against serving it over HTTP: no host change, a real HTTPS origin, at the cost of some duplication.

It wasn't a fair trade, and the fact that settled it was findable in ten minutes: **there are two virtual host mappings, not one.** The second serves her avatars folder, and a VRM is *user data* — in a git-ignored folder, chosen at runtime, in no repository. It can never be baked into a client, so the host had to serve files over HTTP either way.

**The lesson:** before weighing two options, check whether one of them actually avoids the cost it claims to avoid. Route B paid route A's price *and* added drift.

## Read the handler, not the table

`say` carries `text`. The face→host table in `PROTOCOL.md` said so, but the first implementation here used `value` — because `setKey`, `setVoice`, `setAvatar` and every other `set*` message *do* use `value`, and the hand reaches for the pattern.

She accepts the wrong field and silently does nothing with it. There is no error, no log line, and no clue: it presents as a dead Send button.

**The lesson:** for any message you are implementing for the first time, read the `case` in `OctaviaSession` rather than the row in the table. The table is a summary and this one had also fallen five messages behind the code, which is how that gets found.

## Do not echo what the host will echo back

The obvious way to make a chat feel responsive is to append the typed line locally and let the reply arrive later. Here that is wrong twice over: she sends `turn who:"you"` back *before* she starts thinking — so it is not even slow — and a local echo would double every line.

It is also wrong in a way that only shows up later. Waiting for her `turn` means the transcript shows **what was said at the desktop too**, which is the entire point of a second face.

**The lesson:** in a multi-client system, render what the server says happened, not what this client did.

## Assumed elevation, unverified

The plan for a whole session was "install Android Studio in an elevated run", and the toolchain was treated as blocked until then. It never needed elevation: `adb`, a JDK, scrcpy and Google's `android` CLI all install with `winget --scope user`, and that CLI does everything Studio was wanted for.

**The lesson:** try the unprivileged path before scheduling the privileged one. See also [[Moving To The New Machine]] — elevation on this box varies per session and must be checked, never assumed, in either direction.

## A cancel flag does not reach a thread that is asleep

`FaceSocket` retried with a sleeping thread that held the host, port and credential it was created with, and `disconnect()` cleared a single `wanted` boolean. That looks sufficient and is not: the thread wakes *after* `connect()` has set the flag back to true, sees a live client, and reconnects with the **old** credential. Every re-pair added another lineage.

The symptom was her log filling with `face socket refused a connection with a bad or missing token` every one to five seconds, from several loops at once, while entering the correct token changed nothing.

The fix is a **generation counter** bumped on both `connect` and `disconnect`, checked when the thread *wakes* rather than only before it sleeps. Anything belonging to a superseded attempt returns instead of acting.

**The lesson:** a boolean cannot express "stop" when the thing being stopped can outlive the decision and the flag can flip back. Use a monotonic token, and re-check it on the far side of every wait.

**And the reason it was findable at all:** her host logs every refusal. A client that failed quietly would have looked merely slow to pair. Noisy rejection on the server is what made a client-side lifetime bug visible in seconds.

## Decide whether a failure is worth retrying, not just how often

The backoff was careful — exponential, capped at fifteen seconds so a phone rejoining a network does not sit in a five-minute wait. All of that is right and none of it is the question that mattered.

A **401 is not a transient fault.** Her per-run token is regenerated every time she starts, so restarting her silently un-pairs the phone — and this client then retried a credential that *could never work*, every fifteen seconds, for twenty minutes, holding the radio up each time and writing a refusal into her log.

**The lesson:** a retry policy needs two decisions, and the tempting one is the second. First *should* this be retried, then how often. Anything that only a person can fix belongs in its own state — `Refused`, not `Retrying` — because "retrying…" on a screen is a promise that something will change.

## Removing a forward does not close an open stream

`adb reverse --remove tcp:8848` looked like a clean way to simulate losing the network. It is not: it stops *new* connections being forwarded and leaves established streams alone, so the socket stayed up and the test reported success while proving nothing.

`adb kill-server` genuinely breaks the transport — established connections dropped 2 → 1 and recovered unaided once it was restored.

**The lesson:** when a negative test passes immediately, check that it actually removed the thing it claimed to. A test that cannot fail is worse than no test, because it is quoted later.

## Driving a text field over `adb` needs one call, not a loop

Clearing a field with `1..40 | ForEach-Object { adb shell input keyevent KEYCODE_DEL }` silently lost most of the presses — 15 of 40 landed, leaving the tail of the old value glued to the new one and a 49-character "token" that was refused with no clue why.

`input keyevent` accepts **several keycodes in a single invocation**: `adb shell "input keyevent 67 67 67 …"`. One call, no race.

**The lesson:** each `adb shell` is a fresh round trip against a device that is also busy rendering; a loop of them is not a sequence of keystrokes. When the result must be exact, check what actually landed — `adb shell run-as <pkg> cat /data/data/<pkg>/shared_prefs/*.xml` reads it straight back.

## The best test device is not the one that runs the app best

The J7 Pro cannot render a VRM and never will — an Exynos 7870 with no big cores, a single-core Mali-T830, **`armeabi-v7a` only** (32-bit userspace on ARMv8 silicon) and a **192 MB** per-app heap cap.

That makes it a bad judge of the renderer decision and an *excellent* tester for everything else. A `subscribe`/`skip` policy that survives it survives anything, and push-to-talk stopped being a preference and became a consequence the moment the hardware was real.

**The lesson:** name which questions a test device can answer and which it cannot, before its verdict gets quoted for something it was never able to measure.

## Links

- [[Octavia Android]] · [[Architecture]] · [[Build & Release]] · [[Roadmap]]
