---
project: Octavia
tags: [octavia, conventions]
---

# Conventions & Security Model

## Standing constraints

These are load-bearing. Breaking one is a design smell, not a shortcut.

1. **Anything reflex-speed is local.** Lip sync, level meters, and (later) dancing never call a model. The API bill should track *conversation*, not *liveliness*.
2. **The key never reaches a renderer.** New capabilities land in the host.
3. **Renderer changes leave `OctaviaSession` untouched; sense changes leave the face untouched.**
4. **Native runtimes that link CUDA stay out-of-process where practical.** One process should not host two of them — see [[Choosing a Local Model]].

## Security model

- **The API key** is DPAPI-sealed (`DataProtectionScope.CurrentUser`) to one Windows account on one machine, at `<data>\apikey.dat`. `ANTHROPIC_API_KEY` takes precedence when set. It is written *in* from the face and never sent back out — the `hello` message carries a boolean, never the value.
- **It does not travel.** Copying `apikey.dat` to another machine yields an undecryptable blob. This is intended. She degrades gracefully: the read fails, is logged, and she asks for a key as though new.
- **The face is sandboxed by CSP**: `default-src 'none'; script-src 'self'; style-src 'self'` — and `connect-src` names the loopback face socket, the read-only `https://octavia.avatar` origin the host maps to her avatars folder, and `blob:`. It cannot reach the wider network even if something in it wanted to, and it cannot read a character file the host did not offer it.
  - `blob:` is not a loophole: a `blob:` URL can only address data the page already holds. It is on the list because glTF carries its textures inside the binary and three.js decodes them to a `Blob` — without it **every texture in every model was silently blocked**, which is why she had never once rendered with a face before v0.12.0.
- **A face may ask for a diagnostics bundle but never says where it goes.** The host raises its own file dialog. A face that could name the destination would be a face that could write the log — transcripts and all — anywhere on the machine.
- **Downloading an executable is treated differently from downloading a model.** The neural speech engine is fetched only when the neural voice is asked for, into her data folder rather than anywhere on the PATH, from a URL written plainly in `PiperStore.cs` — see [[The Voice]].
- **The diagnostics bundle is the one thing that leaves the building**, so its contents are a decided list rather than whatever was lying around, and it says on its front page that the log holds transcripts of things said in the room. See [[Diagnostics]].
- **The page is a secure context** via `SetVirtualHostNameToFolderMapping` to `https://octavia.face`, not `file://` — required for camera and microphone permissions in later stages.
- **The camera is off by default, and it is the only sense that is.** A microphone in a room is expected; a camera is not. It opens only when three cheap, *readable* conditions hold — the setting, the words, and whether the brain has eyes — and none of them consults a model, because the decision to open a camera in someone's home must be auditable by reading it. One frame, device released in the same breath, an unmissable marker while live, and nothing stored. See [[Eyes]].
- **Nothing acoustic leaves the machine.** Whisper and Silero both run locally. This is a deliberate improvement on the prototype, whose Web Speech API streamed the microphone to Google.
- **The loopback hears everything the machine plays**, which could be a call as easily as a song. So the analysis is local and keeps nothing: buffers are measured and dropped, and what survives is a tempo and a loudness. **No message in the protocol carries audio**, and the diagnostics bundle contains none. The setting that turns it off *closes the device* rather than ignoring it — a switch that left the capture running would be a worse promise than no switch at all. See [[Music]].
- **Music in the room is heard through the microphone, and that is why it is off by default.** `MusicFromRoom` (v0.16.0) tees the frames the voice detector already reads into a second analyser. It adds no new capture — the microphone is already open — but it does mean the same guarantee has to hold on a second path, and it does: measured and dropped, tempo and loudness only, nothing stored and nothing sent. It is off by default because a feature that listens to the room should be switched on deliberately, not inherited.
- **Serving her face beyond this machine is off by default.** `RemoteAccess` (v0.14.0) binds the socket past loopback and v0.20.0 made that socket serve the page itself, so a phone can load her over the LAN. Both are gated on a key. Sub-resources cannot carry a query string, so the key is echoed back as an `HttpOnly`, `SameSite=Strict` cookie rather than being appended to every asset URL — a design forced by how browsers resolve `<link>` and `import()`, not chosen for elegance. `StaticFiles` refuses by default: traversal is checked on the *resolved* path rather than by hunting for `..`, and an unlisted extension is refused rather than sent as octet-stream. It is not a file share. See [[Architecture]].

> **The key gate did not work at all until v0.23.1, and the posture changed the day it started to.** A length guard no generated key could satisfy meant every read of the key replaced it, so a remote face was always compared against a secret a microsecond old and **nothing could ever get in**. The socket was open and accidentally fail-closed. That is now a real lock — which matters here and now, because on this machine `RemoteAccess` is **on**, she answers on `10.1.1.21:8848`, and the Windows firewall is **off entirely**. The remote key is the only thing between the LAN and her microphone. A scoped inbound rule was always the right shape; turning the whole firewall off to open one port is not, and it is worth undoing. See [[Changelog]] 0.23.1.
>
> Two consequences worth carrying forward. **Nothing in Settings displays the key**, despite a source comment saying it is readable so it can be shown there — so `EarsTest remotekey show` exists, and a Settings row is still owed. And **rolling the key unpairs every device at once**, by design: there is no per-device list, so revocation is all-or-nothing and every phone must be told the new one.

> **v0.26.0 made that firewall line urgent rather than tidy.** While she was a window, `RemoteAccess` was opt-in and the desk worked without it — the socket was a convenience. Now it is the *only* way in, for the desktop client as much as the phone, so the bearer key over plain HTTP is load-bearing on a machine with no firewall. **Nothing got less safe; what changed is that there is no longer a path that avoids the lock.** The scoped inbound rule stopped being housekeeping. See [[A Server, And Clients]].

- **The key gets a face *in*. What that face may then do is a separate question, and until v0.24.0 there was no answer.** Every `set*` message acted on the machine she runs on, and **not one of them looked at where it came from** — so a paired phone could open this machine's microphone, change its capture device, switch its output, open Explorer on it, and write a diagnostics bundle containing her transcripts. The remote key was authentication with no authorisation behind it. There is an authority table now, in `OctaviaSession.OnFaceMessage`, keyed on the *room* the message came from: **host only** (the hardware and the folders), **room** (the conversation), **being** (voice, avatar, key — echoed everywhere). `hello.controls` lets a page hide what it cannot use, and that is a hint rather than the guard: a face can always send the message by hand, so the refusal has to live in the host. Refused rather than ignored, out loud, and logged once. See [[One Being, Many Rooms]].
  - The room is **not** derived from the credential. It would be a neat shortcut and it would be wrong: two handsets would silently share a room, and a laptop on the LAN would be indistinguishable from a phone. A room is a statement of intent, so a face states it.
  - **A renderer may borrow senses from whatever it is embedded in** *(v0.25.0)* — `window.OctaviaEmbedder`, which lets a WebView panel use its host app's microphone and camera without either reaching the wire. Two rules make it safe rather than a hole in the guard above. It must be injected over an **origin-restricted** channel — on Android `WebViewCompat.addWebMessageListener` with an allow-list, never `addJavascriptInterface`, because the page arrives over plain HTTP and anything that could inject script into it could otherwise open the microphone. And a **borrowed camera is never claimed to the host**: `senses` still reports what the page itself can do, so `look` is not routed to a face that cannot answer it. The privacy marker stays in the page, one marker in the place a person already looks. See [[Lending A Renderer The Device's Senses]].
  - **Her voice stopped leaking between rooms in the same release.** It used to reach every face that had opted into audio, which meant a question asked in one room was answered aloud in every other — a privacy fault as much as an annoyance, since what she says back is a reply to something somebody said. It goes to one room now, and this machine's speakers are silenced for a turn she is having elsewhere.

## Versioning

Pre-release `0.x`. **PATCH by default** (`0.x.y`) for fixes, tweaks and copy changes. MINOR (`0.x.0`) only for a new subsystem or notable behaviour change — a completed roadmap stage qualifies by definition; "I added a thing" does not. Bump `src\Octavia.App\Octavia.App.csproj` `<Version>` and `versions.md` together.

## Code and docs

- **American English** in all documentation.
- **Comments stay minimal.** Rationale belongs in the commit message and in this vault, not scattered through the source. Comment the *non-obvious why* — the 64-sample VAD context, the out-of-process decision — never the what.
- **Displayed dates are `MM/DD/YYYY`.** `versions.md` changelog headers use ISO `YYYY-MM-DD`, an internal doc convention only.
- **PowerShell: use `pwsh` (7.6) for interactive and diagnostic work.** It defaults to UTF-8. Windows PowerShell 5.1 reads UTF-8 as ANSI, so `Get-Content` on any vault or repo note reports mojibake that is not there — it produced a false corruption alarm on `Changelog.md` on 08/29/2026. Verify text with `pwsh`, `grep`, or `[IO.File]::ReadAllText`, never 5.1's cmdlets.
- **Scripts that may run from a git hook stay 5.1-compatible** — `tools\sync-vault.ps1` is deliberately so, since a target machine may not have pwsh. That is safe because it uses `[IO.File]::ReadAllText` / `WriteAllText` rather than the cmdlets. Portability, not an encoding compromise.
- **Never hand-edit anything under `Code\`** in this vault — `tools\sync-vault.ps1` overwrites it.

## Vault upkeep

The auto-synced half (`Code\`, `_Code Index`, `Changelog`) creates a false impression that the vault is current. **The hand-written half rots silently.** When a change set adds or changes a feature, update the relevant `Features/` note and add any expensive lesson to [[Lessons Learned]] *in the same change set* — not as an occasional catch-up.

Run `tools\sync-vault.ps1 -PruneOrphans` after a source file is renamed or removed; the plain run reports orphans but never deletes them. It caught `VoiceBox.cs` after the rename to `SapiVoice.cs`.

**Screenshots go in `Screenshots\` named `v<version> - <subject>.png`, and the caption says what was being *checked*** — not what is on screen. A picture with no question attached is a gallery; a picture with a question attached is evidence. Take them with `tools\shoot.ps1` (and `tools\poke.ps1` to reach a state), **after every version bump** — most releases change no pixels and need nothing, so the judgement is only ever "did anything visible move?". `check-vault.ps1` reports when the current version has no shot; it reports rather than fails, because demanding a photograph of a documentation-only release would train everyone to ignore it. See [[Screenshots]].

**Never restate the protocol's message lists in a prose note.** [[Face Protocol]] is generated from `PROTOCOL.md` and cannot drift; a hand-kept copy elsewhere can, and two of them did — [[Architecture]] and [[The Face]] both carried lists that were five messages and eight `hello` fields behind, one of them under the words "that is the entire contract". This stopped being cosmetic the moment a second face existed: an Android client was written against the gap, guessed `value` where `say` wanted `text`, and she accepted it and silently did nothing. Link to the mirror instead. **A contract that is quietly incomplete is worse than one that is honestly small.**

**Three notes are generated and must not be hand-edited:** [[Changelog]], [[Roadmap]] and [[Face Protocol]] are rewritten from `versions.md`, `ROADMAP.md` and `PROTOCOL.md` on every sync. They were hand-copied until v0.6.0 and drifted every single time the repo changed.

## Testing

Every subsystem gets a headless harness before it is trusted. `tools\EarsTest` proves the VAD → Whisper pipeline against synthesized speech, asserts silence transcribes to nothing, exercises the streaming text filters, checks config-profile precedence *and persistence*, the face protocol, every guarantee the diagnostics bundle makes about its own contents, the viseme and mood maps, and the audio-derived mouth — then probes the local brain. Each section found a real bug on its first run.

Two of them are **probes rather than assertions**, because some things can only be judged by looking:

| Probe | Answers |
|---|---|
| `EarsTest -- mic` | Which capture devices exist, which one Windows considers default, and whether Windows itself sees any signal |
| `EarsTest -- mouth <wav>` | The lip-sync shape timeline for a piece of speech, and the distribution the thresholds were set from |
| `EarsTest -- music` | What she makes of what is playing, and — the part that matters — how completely and how *faithfully* the audio path delivered it |
| `attach-face.ps1 -Conformance` | Whether the host actually says everything a renderer must hear, with the fields the protocol promises. Run it after changing the host, and before writing a new face |

And two rules learned the expensive way.

**Test that a setting persists, not that it applies.** The settings menu looked entirely correct — dropdown, log line, face changing — while saving nothing at all.

**Measure the input before doubting the algorithm.** When something that works offline fails against a real device, the device is a suspect. A capture diagnostic should report signal *quality* beside frame count: "we received everything" and "what we received is usable" are different claims, and only the second one was false. See [[Music]].

## Secrets a tool server needs *(v0.43.0)*

`Env` is where tokens go and `Args` never is — an argument is visible in the process list to every account on the machine. That rule is right for an API key and **not sufficient for a password**, because `Env` values live in `config.json` in plain text.

So `McpServer` may declare `Secrets`: environment variable names whose values are filled at spawn from `SecretStore`, DPAPI-sealed to the Windows account that wrote them.

```
Octavia.Server.exe --secret unifi:UNIFI_API_KEY
```

Typed with the echo off. Never in `config.json`, never in her log, never on a command line, and deliberately **not** in her Settings panel — that would put it on the wire.

> [!warning] Reversed in v0.48.0
> This note used to argue that **the UniFi API key stayed in `Env`** because it is a scoped, read-only, revocable key while the password is an account credential — and that treating them identically would be consistency at the expense of the distinction that matters.
>
> That was wrong twice over. The key stopped being read-only in v0.41.0, when it started authorising **cutting power to a switch port**; and "revocable" describes the cleanup, not the exposure. See [[#Anything secret-shaped is sealed (v0.48.0)]].

| | |
|---|---|
| Missing is not empty | A declared secret with nothing stored is left **unset**. A server handed `""` reports a login failure, which sends somebody to check the account when nothing was ever stored |
| Sealed to an account | DPAPI. A secret written by a person and read by a service running as LocalSystem is unopenable, and indistinguishable from absent — same trap as the API key |
| The key's entropy is frozen | `Octavia.ApiKey.v1` and `apikey.dat` must not be tidied. DPAPI will not open a blob sealed with different entropy, so renaming either logs everybody out of their own key |

**A check that a secret arrived must not be a way to print one.** The mock server reports the *length* and whether it was set, never the value — and the test secret is cleared in a `finally`, because a test that leaves a credential behind has changed the machine it ran on.

---

## Anything secret-shaped is sealed (v0.48.0)

The rule that replaced the judgement call above: **a value whose name says it is a credential does not live in `config.json`.** No exceptions carved out for scope, revocability, or how the key is used today — that reasoning is what left an API key in plain text for eighteen versions while its powers grew underneath it.

`SecretStore.SealLoose` runs at startup, before anything reads a tool server's settings. It moves every secret-shaped `Env` value into the DPAPI store, adds the name to `Secrets`, takes it out of the file and saves. It is a no-op on every run after the first, so it costs nothing to leave in.

| | |
|---|---|
| Failure keeps the value | If sealing throws, the value is **left exactly where it was**. Removing it from `Env` after a failed seal loses the credential outright, which is worse than one that is readable |
| Empty is just removed | A blank secret-shaped value is deleted rather than sealed. There is nothing to protect and an empty `Secrets` entry would read as "stored" |
| Silent under LocalSystem | DPAPI seals to an account. A service running as the machine would seal a key the person at the keyboard could never replace, so it logs a warning naming the variable and does nothing |
| The window will not draw one | The settings panel refuses to render a secret-shaped `Env` value as text. It gets a masked box, Store/Clear, and the same badge a password gets |

Sealing is **not** asked about. There is no version of it a person would decline, it needs no input, and it changes nothing except who can read the value.

### What counts as secret-shaped

`Sensitive.Looks` splits a name into words and matches whole words against *key, token, secret, password, credential*, plural-insensitively. One implementation, two callers — the diagnostics redactor and the settings window — because two would eventually be two opinions, and the disagreement only ever surfaces as a secret one of them displayed.

> [!bug] It had been wrong since v0.30.0
> The original split fired before **every** capital, so `UNIFI_API_KEY` became thirteen single letters and never contained the word "key". **The one name it was written to catch was the one it could not see.**
>
> Nothing noticed because its only caller was a redactor, and a redactor's failure is invisible by definition — you do not see the secret it should have removed. Splitting only at a *transition into* a capital fixes it. See [[Lessons Learned]].

`MaxTokens` reads as a secret by name, and a check asserts that on purpose. Nothing about the *name* separates it from `AccessToken` — the **value** does, a budget being a number and a token a string — so the bundle asks both questions and only ever redacts a string. Special-casing the name to make `MaxTokens` look right would take `ApiKeys` and `SessionTokens` down with it.

### Rotate a key that was ever in plain text

Sealing protects it from here on; it does nothing about wherever the file has already been. A key that sat in `config.json` should be **rotated at the source**, not just sealed.

## The credential that was never needed *(v0.49.0)*

The note above spends several paragraphs on how the UniFi **account password** should be stored, and concludes correctly: DPAPI-sealed, filled at spawn, never in `config.json`. All of that was right.

**It should never have been there at all.**

The password existed because the security log lives on UniFi's older `/proxy/network/api`, and the finding written down was *"the API key cannot reach it"*. What was actually measured was that every **integration**-API event route 404s — probed carefully, with a nonsense-path control to prove the 404s were real. The conclusion was then extended one step past the experiment, and nobody tested the older API with the key.

The key authenticates there perfectly well. Reads *and* writes — the v0.49.0 PoE switch writes device configuration through the same path.

So a real UniFi account credential sat sealed in the store for months, guarding nothing, when it is worth considerably more to an attacker than the scoped key beside it. Removed, along with the login, the session, the CSRF token and the 401 retry that existed only to renew them.

> **`UNIFI_USERNAME` and `UNIFI_PASSWORD` are read by nothing.** The stored secret file and the read-only UniFi account can both be deleted.

**What generalises:** an exemption and an extra credential are opposite mistakes with the same cause — a claim about capability written down once and never re-tested. The v0.41.0 note said the key was read-only, and it stopped being read-only two releases later. This note said the key could not reach the legacy API, and it always could. **Write down what was measured, not what follows from it**, because the inference is what gets quoted later. See [[Lessons Learned]].

### The read-only account was still worth having

Not as a mistake to undo entirely. It proved the sealed-secret path end to end, and `Secrets` exists and is correct because of it — the machinery that now protects `UNIFI_API_KEY` was built for a password that turned out to be unnecessary. Worth separating: **the mechanism earned its place; the credential did not.**
