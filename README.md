---
name: vep-extractor
description: Inspect and extract AVS Video Editor (AVS4YOU) .vep project files — plain-XML timeline, source/effect/menu structure, and embedded base64 PNG thumbnails, entirely client-side. Trigger whenever the user has a .vep file or asks to open/inspect/extract an AVS Video Editor project or its thumbnails. Clip timeline position and time units are inferred, not confirmed — preserve those caveats rather than presenting them as settled.
---

# .vep — AVS Video Editor Project Format

**Scope:** `.vep` project files written by AVS Video Editor (AVS4YOU). Confirmed against one real file (`Cheers_.vep`, 273,958 bytes). Everything below marked **confirmed** was checked against that file's bytes; **inferred** means a reasonable reading that nothing in the file proves.

Separate from the `.veg` skill on purpose: different container family, different tooling, different pitfalls.

---

## 1. Container

- **Plain UTF-8 text, CRLF line endings, no XML declaration** (`<?xml ... ?>` absent), very long lines (15,002 chars on one). Confirmed via `file`: "Unicode text, UTF-8 text, with very long lines, with CRLF line terminators".
- Not binary, not compressed, not a container. Parse with `DOMParser` as `application/xml`; no parser error on the sample.
- Root: `<TimeLine version="1" timebinding="1" resizetype="1">` → `<Project>` → … → `<VideoMenu>`. File ends `</VideoMenu></Project></TimeLine>`.
- Top-level groups seen: `SourceGroup`, `DeletionGroup`, `VideoEffectGroup`, `TextGroup`, `AudioEffectGroup`, `PipGroup`, `MixGroup`.

## 2. Confirming the app

No product name or version string appears anywhere in the file (checked head, tail, and a list of other editors' names). The app is confirmed only through an absolute install path inside a menu-style reference:

```
C:\Program Files (x86)\AVS4YOU\AVSVideoEditor\MenuStyles\...
```

Do not claim an app *version* — none is recorded. `TimeLine version="1"` is a schema version, not the app's.

## 3. Sources

```xml
<Source sourcetype="video" filename="C:\Users\...\clip.mkv"
        sourcebegin="26137" sourceend="89574" audiotrack="0" videotrack="0" .../>
<Source sourcetype="audio" filename="C:\...\fireworks.wav"
        sourcebegin="0" sourceend="4500" audiotrack="0"/>
```

- Sample has exactly 4 `<Source>` elements: 1 video, 3 audio (two are the same `cheers2.wav`). All on video track 0 / audio track 0.
- `sourcebegin` / `sourceend` are the **trim in/out points inside that source's own media** (**confirmed by role**: the video's 26137→89574 is a ~63.4 s cut). Audio values can be fractional (`3096.78004535147`).
- **Units are inferred as milliseconds** (the numbers only make sense that way for a ~minute-long clip). Nothing in the file states the unit.
- **There is no timeline-position attribute anywhere.** Sequential placement per track (clip N starts where clip N-1 ends) is an **assumption** that fits this simple one-track-per-type project. Always say so in the UI; don't present it as the real layout.
- Paths are **plaintext absolute Windows paths** → leak the user's Windows username and folder layout. Worth a privacy note whenever a file is shared.

## 4. Embedded thumbnails (the part that was missed once)

```xml
<Thumbnails><Group id="0"><Line id="0">
  <Thumbnail type="2" time="0" width="160" height="90">iVBORw0KGgoAAAANSUhEUg...(line-wrapped base64)...</Thumbnail>
```

- Sample has **2** `<Thumbnail>` elements: `160×90` (with `type="2" time="0"`) and `227×128`. Both decode to valid PNGs (2803 and 2310 bytes). The 160×90 one is a screenshot matching the screen-recording source.
- Verification that holds: base64 body starts `iVBORw0KGgo` → PNG magic `89 50 4E 47 0D 0A 1A 0A`; width/height read from the PNG **IHDR** (bytes 16–24) equal the XML `width`/`height` attributes exactly. Check IHDR, don't trust the attributes alone.
- **The data is element text, not an attribute, and it is line-wrapped.** A search for long unbroken base64 runs, or for `attr="long value"`, finds nothing and wrongly concludes "no thumbnails". That exact mistake happened once. Correct approach: match `<Thumbnail ...>` … `</Thumbnail>` on the raw text, strip all whitespace from the body, then base64-decode. Render with `data:image/png;base64,...`.
- Count and sizes will differ per project; don't hardcode 160×90.

## 5. Menu / disc-authoring block

`<VideoMenu>` carries a full disc-menu system: `SimpleMenu` (111 in sample), `ChapterMask` (18), `ChapterContentRect` (18), `DrawObject` (183), `PenProperties`/`BrushProperties` (92 each), and a `<TranslationTable>` with **13** `<Translation typeid="#602" language="..">` entries (confirmed: `language="cn"` with strings like `播放`, `章节`; a Danish `Kapitel $captionnumber$ af $captioncount$` string also present).

- `<Text key=".." value=".."/>` and `<AutoText>` tags (78 / 36) live in this menu/translation area. The `TextGroup Text, TextGroup AutoText` selector matched **0** timeline text objects. Treat those Text tags as **menu strings, not timeline text clips** (inferred from where they sit).
- `<Image path="$basepath$...">` (11 tags, 4 unique paths) are **external template asset references** to the editor's bundled menu backgrounds. Not embedded, nothing to render. Do not confuse with `<Thumbnail>`.

## 6. Sample stats (for sanity-checking a parser)

3,612 XML elements, 4,021 lines, 13 `<Effect>`, most common tags: `Property` 2,229, `Rect` 274, `DrawObject` 183, `SimpleMenu` 111.

## 7. Gotchas

- **`table.innerHTML = '<tr>…'` auto-wraps rows in `<tbody>`**, so `table.children.length` reports 1. Count with `querySelectorAll('tr')`. This produced a false "only 1 row" bug alarm during testing; the extractor was fine.
- A selector working in isolation but "failing" in the page is usually a test-harness problem; re-test with the page's own render functions before changing code.
- Audio and video **media are never embedded** — only paths. Previews of audio need a placeholder (e.g., 1000 Hz tone). Video clips can use the real thumbnail.

## 8. Not established

- App version; time units (ms is inferred); actual timeline positions; whether other projects use multiple tracks or overlapping clips; what the 13 effects contain beyond their count; how `DeletionGroup` / `PipGroup` / `MixGroup` populate in richer projects (only one small sample seen).

## 9. Extractor behavior (reference build: `vep_extractor.html`)

Single-file dark-UI HTML. Drop/pick a `.vep` → stats grid, honest-notice banner, sequential timeline (real thumbnail on the video clip, tone-placeholder audio clips), thumbnails panel, sources table, menu summary, resource-reference table, depth-capped raw XML tree. All client-side.
