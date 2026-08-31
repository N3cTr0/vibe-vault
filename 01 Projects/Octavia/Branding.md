---
project: Octavia
tags: [octavia, branding]
---

# Branding

What she has looked like, and when. Source art lives in `Branding\`, dated
`YYYY-MM-DD - <what it is>.png`, so a later sheet sits beside the earlier one rather than
replacing it — the point is to be able to see the drift.

The **source files are kept, not just the generated icons.** An `.ico` is a dead end: it
cannot be re-cropped, re-masked or re-exported at a size nobody asked for yet. Whatever the
icons were cut from is the thing worth keeping.

---

## 2026-08-31 — the first mark

![[2026-08-31 - Octavia mock logo sheet.png]]

A mock sheet, 1536×1024, carrying the wordmark and three icon treatments: an app icon as a
rounded square, a circular tray icon, and a monochrome version. Generated art, supplied as
a single sheet on a black background — there is no transparent master, and no vector.

**Shipped in v0.21.1** as the exe, window and tray icons, and as the favicon on the page the
socket serves. Before this she had **no icon at all**: the csproj had no `<ApplicationIcon>`
and the tray fell back to `SystemIcons.Application`, the generic Windows box.

### What was done to it

`tools\make-icons.ps1` cuts the panels out and writes the icons. It is committed rather than
scratch, because the moment a transparent or higher-resolution master arrives the whole set
should be regenerated in one command rather than re-derived by hand.

Two decisions worth keeping:

- **The mask is a shape, not a colour key.** Keying black to transparent would punch holes
  straight through artwork that is mostly dark. Clipping to the rounded square and the circle
  the design already draws removes the surround and leaves the art untouched.
- **Nothing is upscaled.** Every generated size is a downscale from its panel. A 512px PWA
  icon was generated and then deleted: the app panel is 315px, so 512 would have been
  invented detail presented as resolution.

### The tray icon does not really work, and that is a property of the art

Measured rather than guessed — rendered at a true 16×16, which is what Windows draws, then
magnified:

- The **colour circle** becomes an unreadable purple blob. No face, no ring, no structure.
- The **monochrome** version is worse: a dark grey smudge, near-invisible on a dark taskbar.

The circular mask rescued it as far as it can be rescued — it is now a clean purple *disc*
rather than a black square, and it reads as a distinct coloured dot in the tray. That is
acceptable and it is what shipped, but it is not the design working.

**If it is ever worth fixing:** the strongest small mark already in this sheet is the
concentric ring at her temple. A circle-in-circle survives 16 pixels; a circuit-brain, a face
profile and a HUD ring do not. That would be a tray-only variant, not a change to the logo.

### What is missing

- **No transparent master.** The black surround is masked off by shape, which works for the
  square and the circle and would not survive a different crop.
- **No vector.** Every size is a downscale of a 315px raster panel, so 256 is the practical
  ceiling for the app icon and the sheet cannot be re-laid out.
- **No monochrome tray variant that reads small** — see above.

### A note on the tagline

The sheet reads *"AI Home Security Assistant — Smart. Silent. Always Watching."* Her actual
behaviour is close to the opposite: the camera is **off by default**, it is the only sense
that is, it opens for one frame when a question needs eyes, and it shows an unmissable marker
while live. That is a deliberate promise in `camera.js` and in
[[Conventions & Security Model]].

Fine on a logo. Worth not letting it drift into the README or the app, where it would
describe something she is not.

## Links

- [[Screenshots]] — what the app looked like, release by release
- [[Conventions & Security Model]] — the camera promise the tagline sits against
- [[Changelog]] — 0.21.1
